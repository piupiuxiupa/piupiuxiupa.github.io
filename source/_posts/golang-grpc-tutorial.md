---
title: Golang gRPC 实战教程：从 proto 定义到四种通信模式的完整落地
date: 2026-11-12 22:35:39
tags: [golang, grpc, rpc, protobuf, 教程]
categories: Posts
excerpt: 以一套经过真实编译、运行、测试验证的完整示例为主线，讲清 Go 里 gRPC 的全流程：proto 规范与代码生成的坑、服务端四种 RPC 实现、拦截器与健康检查、客户端 deadline 与错误码处理、bufconn 测试，以及 TLS/keepalive/K8s 生产要点。
---

微服务之间互相调用，裸 HTTP + JSON 写多了就会发现两个痛点：接口没有契约，全靠口口相传；序列化性能和流式能力都不够用。gRPC 用 proto 文件当接口的唯一真值来源，HTTP/2 单连接多路复用，四种通信模式把「一问一答」到「双向实时」全覆盖，这也是它成为微服务间通信事实标准的原因。

这篇是实操手册，不是概念科普：从 proto 定义、代码生成，到服务端/客户端完整实现、拦截器、测试、生产部署要点，一条线走完。**文中所有代码都来自一个真实编译、运行、测试通过的项目**（Go 1.27 / grpc v1.85 / protoc 29.3 / protoc-gen-go v1.36.12），不是纸上谈兵的伪代码。

---

## 1. gRPC 核心概念

三个核心角色：

| 概念 | 说明 |
|---|---|
| `.proto` 文件 | 接口的唯一真值来源（single source of truth），服务端和客户端都从这里生成代码 |
| Channel/连接 | gRPC 在**单条 HTTP/2 连接**上多路复用所有 RPC，客户端应复用连接而非每请求新建 |
| Status Code | 标准错误码（`NotFound`、`Unavailable`……），客户端靠它决定「重试 / 快速失败 / 降级」 |

四种通信模式：

| 模式 | 签名特征 | 典型场景 |
|---|---|---|
| 一元（Unary） | `rpc X(A) returns (B)` | 普通 CRUD |
| 服务端流 | `rpc X(A) returns (stream B)` | 日志tail、结果集分批下发 |
| 客户端流 | `rpc X(stream A) returns (B)` | 批量上传、聚合上报 |
| 双向流 | `rpc X(stream A) returns (stream B)` | 聊天、实时同步 |

---

## 2. 环境准备

```bash
# 1) Go 插件（装到 $GOPATH/bin，确保该目录在 PATH 里）
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# 2) protoc 本体（Ubuntu 无包或想要新版本，直接下官方预编译二进制）
curl -sSL -o /tmp/protoc.zip \
  https://github.com/protocolbuffers/protobuf/releases/download/v29.3/protoc-29.3-linux-x86_64.zip
mkdir -p ~/tools/protoc && unzip -oq /tmp/protoc.zip -d ~/tools/protoc
export PATH=$PATH:~/tools/protoc/bin:~/go/bin

# 3) 验证
protoc --version          # libprotoc 29.3
protoc-gen-go --version   # protoc-gen-go v1.36.12
```

> 提示：`protoc-gen-*` 插件不需要单独「运行」，`protoc` 会按名字约定自动调用 `PATH` 里的它们。
> 现代替代方案是 [buf](https://buf.build)（自带依赖管理与 lint），本文用原生 protoc 讲清原理。

---

## 3. 定义 Proto 文件

推荐按「领域 + 版本」组织目录，生成代码与手写代码分离：

```
grpc-demo/
├── go.mod
├── proto/
│   └── user/v1/
│       └── user.proto      # proto 定义
├── gen/                    # 生成代码（不手改）
│   └── user/v1/
│       ├── user.pb.go
│       └── user_grpc.pb.go
├── server/main.go
└── client/main.go
```

`proto/user/v1/user.proto`：

```protobuf
syntax = "proto3";

package tutorial.user.v1;

// 格式: "<Go 导入路径>;<Go 包名>"，分号后的别名会成为生成代码的 package 名
// 模块路径换成你自己的（下文 protoc 命令里的 module= 同理）
option go_package = "github.com/kris/grpc-demo/gen/user/v1;userv1";

service UserService {
  // 一元 RPC：请求一次，响应一次
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  // 服务端流：一次请求，服务端返回一串消息
  rpc ListUsers(ListUsersRequest) returns (stream User);
  // 客户端流：客户端发一串消息，服务端汇总后响应一次
  rpc BatchCreateUsers(stream CreateUserRequest) returns (BatchCreateUsersResponse);
  // 双向流：两边各自独立收发，适合聊天/实时同步
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}

message User {
  string id = 1;
  string name = 2;
  string email = 3;
}

message GetUserRequest {
  string user_id = 1;
}

message GetUserResponse {
  User user = 1;
}

message ListUsersRequest {
  int32 page_size = 1;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
}

message BatchCreateUsersResponse {
  int32 created = 1;
  repeated string ids = 2;
}

message ChatMessage {
  string from = 1;
  string text = 2;
}
```

要点：

- **始终用 Request/Response 包装消息**，RPC 参数不要直接用 `string` 裸类型——裸类型以后没法加字段，包装消息可以向后兼容演进。
- 字段编号（`= 1`）一旦发布就**不能改**，只能新增。
- `go_package` 用小写版本后缀做别名（`userv1`），多版本共存不冲突。

---

## 4. 生成 Go 代码

```bash
go mod init github.com/kris/grpc-demo

protoc --go_out=. --go_opt=module=github.com/kris/grpc-demo \
       --go-grpc_out=. --go-grpc_opt=module=github.com/kris/grpc-demo \
       proto/user/v1/user.proto
```

> **踩坑提示**：网上常见写法是 `--go_opt=paths=source_relative`，它会把生成文件放到
> **proto 源文件的相对路径**下（即 `proto/user/v1/`），和 `go_package` 声明的 `gen/user/v1` 对不上。
> 想按 `go_package` 精确输出到 `gen/`，用 `module=<你的模块名>` 模式——它以模块前缀裁剪导入路径。

生成两份文件：

- `user.pb.go`：消息类型的序列化/反序列化代码
- `user_grpc.pb.go`：客户端 stub（`NewUserServiceClient`）与服务端接口（`UserServiceServer`）

拉取运行时依赖并验证编译：

```bash
go mod tidy
go build ./gen/...
```

---

## 5. 实现服务端

### 5.1 业务实现：实现生成的 `UserServiceServer` 接口

```go
type userServer struct {
	// 必须内嵌 UnimplementedUserServiceServer（不要用 Unsafe 版本）：
	// 以后 proto 新增 RPC 时，旧服务端仍能编译、并对未实现方法返回
	// codes.Unimplemented，保持前向兼容。
	userv1.UnimplementedUserServiceServer

	mu    sync.Mutex
	users map[string]*userv1.User
}
```

一元 RPC：

```go
func (s *userServer) GetUser(ctx context.Context, req *userv1.GetUserRequest) (*userv1.GetUserResponse, error) {
	if req.UserId == "" {
		// 错误必须用 status.Error 返回具体状态码；
		// 裸 error 会变成 codes.Unknown，客户端无法判断能否重试。
		return nil, status.Errorf(codes.InvalidArgument, "user_id is required")
	}

	s.mu.Lock()
	u, ok := s.users[req.UserId]
	s.mu.Unlock()

	if !ok {
		return nil, status.Errorf(codes.NotFound, "user %q not found", req.UserId)
	}
	return &userv1.GetUserResponse{User: u}, nil
}
```

服务端流——逐条 `Send`，结束返回 `nil`：

```go
func (s *userServer) ListUsers(req *userv1.ListUsersRequest, stream userv1.UserService_ListUsersServer) error {
	n := int(req.PageSize)
	if n <= 0 || n > len(s.users) {
		n = len(s.users)
	}

	i := 0
	s.mu.Lock()
	defer s.mu.Unlock()
	for _, u := range s.users {
		if i >= n {
			break
		}
		if err := stream.Send(u); err != nil {
			return err // 客户端断开时 Send 会报错，返回即释放资源
		}
		i++
	}
	return nil
}
```

客户端流——循环 `Recv` 直到 `io.EOF`，再 `SendAndClose` 回汇总响应：

```go
func (s *userServer) BatchCreateUsers(stream userv1.UserService_BatchCreateUsersServer) error {
	var (
		created int
		ids     []string
	)
	for {
		req, err := stream.Recv()
		if err != nil {
			if errors.Is(err, io.EOF) { // io.EOF 表示客户端发完了
				break
			}
			return err
		}
		if req.Name == "" {
			return status.Error(codes.InvalidArgument, "name is required")
		}

		id := fmt.Sprintf("u-%d", len(ids)+100)
		ids = append(ids, id)
		created++
	}
	return stream.SendAndClose(&userv1.BatchCreateUsersResponse{Created: int32(created), Ids: ids})
}
```

双向流——`Recv` 与 `Send` 独立进行，客户端关闭发送方向时 `Recv` 返回 `io.EOF`：

```go
func (s *userServer) Chat(stream userv1.UserService_ChatServer) error {
	for {
		msg, err := stream.Recv()
		if err != nil {
			return nil // 客户端 CloseSend 了，正常结束
		}
		if err := stream.Send(&userv1.ChatMessage{
			From: "server",
			Text: "echo: " + msg.Text,
		}); err != nil {
			return err
		}
	}
}
```

### 5.2 main：装配服务、健康检查、优雅退出

```go
func main() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("listen: %v", err)
	}

	// 拦截器链：横切关注点（日志、panic 恢复）在这里做，业务代码保持干净
	srv := grpc.NewServer(
		grpc.ChainUnaryInterceptor(loggingInterceptor, recoveryInterceptor),
	)

	userv1.RegisterUserServiceServer(srv, newUserServer())

	// 健康检查：Kubernetes 就绪探针依赖它，缺了会导致部署时 pod 被误杀
	hs := health.NewServer()
	hs.SetServingStatus("", healthpb.HealthCheckResponse_SERVING)
	healthpb.RegisterHealthServer(srv, hs)

	// 开发期开启反射（grpcurl 调试必备）；生产环境必须关闭——它暴露完整 API 面
	reflection.Register(srv)

	go func() {
		log.Printf("gRPC server listening on %s", lis.Addr())
		if err := srv.Serve(lis); err != nil {
			log.Fatalf("serve: %v", err)
		}
	}()

	// 优雅退出：先 GracefulStop 排空在途请求，15 秒超时则强停
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit
	log.Println("shutting down...")

	stopped := make(chan struct{})
	go func() {
		srv.GracefulStop()
		close(stopped)
	}()
	select {
	case <-stopped:
	case <-time.After(15 * time.Second):
		srv.Stop()
	}
	log.Println("bye")
}
```

---

## 6. 实现客户端

### 6.1 建连接（复用！）与一元调用

```go
// 连接复用：gRPC 在单条 HTTP/2 连接上多路复用所有 RPC，
// 不要每个请求建一条新连接（浪费 TCP/TLS 握手）。建一次，到处用。
conn, err := grpc.NewClient("localhost:50051",
	grpc.WithTransportCredentials(insecure.NewCredentials()), // 仅开发环境
)
if err != nil {
	log.Fatalf("dial: %v", err)
}
defer conn.Close()

client := userv1.NewUserServiceClient(conn)

// 每次调用都设置 deadline：不设的话，慢上游会无限挂起 goroutine
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()

// 通过 metadata 携带认证令牌 / trace id
ctx = metadata.AppendToOutgoingContext(ctx, "authorization", "Bearer demo-token")

resp, err := client.GetUser(ctx, &userv1.GetUserRequest{UserId: "u-1"})
if err != nil {
	log.Fatalf("GetUser: %v", err)
}
log.Printf("got user: id=%s name=%s", resp.User.Id, resp.User.Name)
```

> 注意：新版 gRPC 用 `grpc.NewClient`（惰性建连），老教程里的 `grpc.Dial` 已不推荐。

### 6.2 错误处理：按状态码决策

```go
_, err = client.GetUser(ctx, &userv1.GetUserRequest{UserId: "missing"})
if err != nil {
	st := status.Convert(err)
	log.Printf("code=%s msg=%s", st.Code(), st.Message())
	switch st.Code() {
	case codes.NotFound:
		// 资源不存在 → 不重试
	case codes.Unavailable:
		// 瞬时故障 → 适合重试
	case codes.DeadlineExceeded:
		// 超时 → 视业务降级
	}
}
```

### 6.3 服务端流：循环 `Recv` 直到 `io.EOF`

```go
stream, err := client.ListUsers(ctx, &userv1.ListUsersRequest{PageSize: 2})
if err != nil {
	log.Fatalf("ListUsers: %v", err)
}
for {
	u, err := stream.Recv()
	if errors.Is(err, io.EOF) {
		break
	}
	if err != nil {
		log.Fatalf("Recv: %v", err)
	}
	log.Printf("user: %s %s", u.Id, u.Name)
}
```

### 6.4 客户端流：循环 `Send`，最后 `CloseAndRecv`

```go
cstream, err := client.BatchCreateUsers(ctx)
if err != nil {
	log.Fatalf("BatchCreateUsers: %v", err)
}
for _, name := range []string{"Dave", "Eve"} {
	if err := cstream.Send(&userv1.CreateUserRequest{Name: name}); err != nil {
		log.Fatalf("Send: %v", err)
	}
}
cresp, err := cstream.CloseAndRecv()
if err != nil {
	log.Fatalf("CloseAndRecv: %v", err)
}
log.Printf("created=%d ids=%v", cresp.Created, cresp.Ids)
```

### 6.5 双向流：Send 与 Recv 各自独立，通常一收一发两个 goroutine

```go
bi, err := client.Chat(ctx)
if err != nil {
	log.Fatalf("Chat: %v", err)
}

// 单独的 goroutine 收，主 goroutine 发
done := make(chan struct{})
go func() {
	defer close(done)
	for {
		msg, err := bi.Recv()
		if errors.Is(err, io.EOF) || errors.Is(err, context.Canceled) {
			return
		}
		if err != nil {
			log.Printf("recv end: %v", err)
			return
		}
		log.Printf("%s says: %s", msg.From, msg.Text)
	}
}()
for _, text := range []string{"hello", "grpc", "bye"} {
	if err := bi.Send(&userv1.ChatMessage{From: "client", Text: text}); err != nil {
		log.Fatalf("send: %v", err)
	}
}
// 关闭发送方向：服务端 Recv 返回 io.EOF，但接收方向仍然打开
if err := bi.CloseSend(); err != nil {
	log.Fatalf("CloseSend: %v", err)
}
<-done
```

### 实测输出（服务端 + 客户端真实运行）

```
$ ./client
[unary] got user: id=u-1 name=Alice
[unary] expected error: code=NotFound msg=user "missing" not found
[unary] -> resource missing, no retry
[server-stream] user: u-2 Bob
[server-stream] user: u-3 Carol
[client-stream] created=2 ids=[u-100 u-101]
[bidi] server says: echo: hello
[bidi] server says: echo: grpc
[bidi] server says: echo: bye
all RPC patterns OK
```

服务端拦截器日志与 SIGTERM 优雅退出：

```
gRPC server listening on [::]:50051
method=/tutorial.user.v1.UserService/GetUser duration=1.369µs code=OK
method=/tutorial.user.v1.UserService/GetUser duration=5.363µs code=NotFound
shutting down...
bye
```

---

## 7. 错误处理与状态码

核心规则：**永远返回 `status.Error` 带具体状态码**。裸 `error` 到了对端就是 `codes.Unknown`，客户端无法判断该重试还是放弃。

```go
// ✗ 坏：调用方拿到 codes.Unknown，无法决策
return nil, fmt.Errorf("user not found")

// ✓ 好：具体状态码让客户端可以行动
return nil, status.Errorf(codes.NotFound, "user %q not found", req.UserId)
```

常用状态码速查：

| 状态码 | 何时使用 |
|---|---|
| `InvalidArgument` | 入参畸形（缺字段、格式错） |
| `NotFound` | 实体不存在 |
| `AlreadyExists` | 创建冲突，实体已存在 |
| `PermissionDenied` | 无权限 |
| `Unauthenticated` | 缺失或无效令牌 |
| `FailedPrecondition` | 系统状态不满足前置条件 |
| `ResourceExhausted` | 限流、配额用尽 |
| `Unavailable` | 瞬时故障，**可安全重试** |
| `Internal` | 未预期的 bug |
| `DeadlineExceeded` | 超时 |

重试语义：`Unavailable` 才会触发自动重试；`InvalidArgument` 重试没有意义。状态码给错，客户端重试逻辑就废了。

字段级校验错误可以附加 `errdetails.BadRequest`（`google.golang.org/genproto/googleapis/rpc/errdetails`），通过 `status.WithDetails` 携带结构化的逐字段错误信息。

---

## 8. 四种通信模式

| 模式 | 服务端写法 | 客户端写法 | 结束语义 |
|---|---|---|---|
| 一元 | 返回 `(resp, error)` | 直接拿到 `resp, err` | 同步往返 |
| 服务端流 | 循环 `stream.Send`，完事 `return nil` | 循环 `stream.Recv` 直到 `io.EOF` | 服务端流完即结束 |
| 客户端流 | 循环 `stream.Recv` 直到 `io.EOF`，`SendAndClose` | 循环 `stream.Send`，`CloseAndRecv` 拿响应 | `CloseAndRecv` 收尾 |
| 双向流 | `Recv`/`Send` 独立循环 | 收发两个 goroutine，`CloseSend` 关发送方向 | 各方向独立关闭 |

大消息（接近默认 4MB 上限）应改为流式分块传输，避免单消息体积限制并降低内存压力。

---

## 9. 拦截器

拦截器是 gRPC 的「中间件」，用于日志、认证、恢复 panic、指标等横切关注点。

```go
func loggingInterceptor(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
	start := time.Now()
	resp, err := handler(ctx, req)
	log.Printf("method=%s duration=%s code=%s", info.FullMethod, time.Since(start), status.Code(err))
	return resp, err
}

func recoveryInterceptor(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp any, err error) {
	defer func() {
		if r := recover(); r != nil {
			log.Printf("panic in %s: %v", info.FullMethod, r)
			err = status.Errorf(codes.Internal, "internal error")
		}
	}()
	return handler(ctx, req)
}
```

注册为链（按顺序执行）：

```go
srv := grpc.NewServer(
	grpc.ChainUnaryInterceptor(loggingInterceptor, recoveryInterceptor),
)
```

- 一元拦截器：`grpc.UnaryServerInterceptor` / `grpc.ChainUnaryInterceptor`
- 流式拦截器：`grpc.StreamServerInterceptor` / `grpc.ChainStreamInterceptor`（签名不同，需要单独写）
- 客户端侧也有对应拦截器（`grpc.WithUnaryInterceptor` 等），常用来注入 token / trace id
- 现成生态：`github.com/grpc-ecosystem/go-grpc-middleware`（日志、prometheus、重试等）

---

## 10. 用 bufconn 做测试

`bufconn` 提供内存内连接，走**完整 gRPC 栈**（序列化、拦截器、metadata 全都真实生效），没有网络开销——这是 gRPC 单测的标准做法。

```go
func setupTest(t *testing.T) userv1.UserServiceClient {
	t.Helper()
	lis := bufconn.Listen(1024 * 1024)
	t.Cleanup(func() { lis.Close() })

	srv := grpc.NewServer(
		grpc.ChainUnaryInterceptor(loggingInterceptor, recoveryInterceptor),
	)
	userv1.RegisterUserServiceServer(srv, newUserServer())
	t.Cleanup(func() { srv.Stop() })
	go srv.Serve(lis)

	conn, err := grpc.NewClient("passthrough:///bufconn",
		grpc.WithContextDialer(func(ctx context.Context, _ string) (net.Conn, error) {
			return lis.DialContext(ctx)
		}),
		grpc.WithTransportCredentials(insecure.NewCredentials()),
	)
	if err != nil {
		t.Fatalf("dial: %v", err)
	}
	t.Cleanup(func() { conn.Close() })
	return userv1.NewUserServiceClient(conn)
}
```

错误场景必须断言**具体状态码**（客户端依赖状态码决定重试策略）：

```go
func TestGetUser_Errors(t *testing.T) {
	client := setupTest(t)

	tests := []struct {
		name     string
		req      *userv1.GetUserRequest
		wantCode codes.Code
	}{
		{"empty user id", &userv1.GetUserRequest{UserId: ""}, codes.InvalidArgument},
		{"user not found", &userv1.GetUserRequest{UserId: "nonexistent"}, codes.NotFound},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			_, err := client.GetUser(context.Background(), tt.req)
			st, ok := status.FromError(err)
			if !ok {
				t.Fatalf("expected gRPC status error, got: %v", err)
			}
			if st.Code() != tt.wantCode {
				t.Errorf("code = %s, want %s; message: %s", st.Code(), tt.wantCode, st.Message())
			}
		})
	}
}
```

本教程示例的实测结果：

```
$ go test -v ./server/
--- PASS: TestGetUser (0.00s)
--- PASS: TestGetUser_Errors (0.00s)
    --- PASS: TestGetUser_Errors/empty_user_id
    --- PASS: TestGetUser_Errors/user_not_found
--- PASS: TestListUsers_Stream (0.00s)
--- PASS: TestBatchCreate_ClientStream (0.00s)
--- PASS: TestChat_Bidi (0.00s)
PASS  github.com/kris/grpc-demo/server  0.009s
```

需要打真实依赖（数据库、外部服务）的集成测试，用 build tag 隔离：`//go:build integration`，然后 `go test -tags=integration ./...`。

---

## 11. 生产环境要点

### 安全

- **生产必须开 TLS**：metadata 是明文传输的，凭据裸奔等于泄密。服务端用 `grpc.Credentials(credentials.NewServerTLSFromCert(&cert))`，客户端用 `credentials.NewClientTLSFromFile(...)`。
- 服务间认证：mTLS，或交给服务网格（Istio/Linkerd）。
- 用户认证：实现 `credentials.PerRPCCredentials` 注入 token，服务端在认证拦截器里校验。
- **生产关闭 reflection**：开着等于把全部 API 面送给攻击者枚举。

### 性能

| 配置 | 用途 | 典型值 |
|---|---|---|
| `keepalive.ServerParameters.Time` | 空闲连接 ping 间隔 | 30s |
| `keepalive.ServerParameters.Timeout` | ping 无应答判定超时 | 10s |
| `grpc.MaxRecvMsgSize` | 放宽默认 4MB 单消息上限 | 16MB |
| 连接池 | 高负载流式场景 | 4 连接 |

大多数服务**不需要**连接池——先压测再上复杂度。

### Kubernetes

- 注册健康检查服务（`grpc_health_v1`），否则就绪探针没法工作，部署滚动更新时会误杀 pod。
- 无头 Service（headless）+ `dns:///user-service:50051` + `round_robin` 负载均衡策略：

```go
conn, err := grpc.NewClient("dns:///user-service:50051",
	grpc.WithTransportCredentials(creds),
	grpc.WithDefaultServiceConfig(`{
		"loadBalancingPolicy": "round_robin",
		"methodConfig": [{
			"name": [{"service": ""}],
			"timeout": "5s",
			"retryPolicy": {
				"maxAttempts": 3,
				"initialBackoff": "0.1s",
				"maxBackoff": "1s",
				"backoffMultiplier": 2,
				"retryableStatusCodes": ["UNAVAILABLE"]
			}
		}]
	}`),
)
```

---

## 12. 常见错误对照表

| 错误做法 | 后果与修复 |
|---|---|
| 返回裸 `error` | 变成 `codes.Unknown`，客户端无法决策。改用 `status.Errorf` 带具体状态码 |
| 客户端调用不设 deadline | 慢上游无限挂起 goroutine。每次调用都 `context.WithTimeout` |
| 每个请求新建连接 | 浪费 TCP/TLS 握手。建一次复用，HTTP/2 自动多路复用 |
| 生产开 reflection | 攻击者可枚举所有方法。只在 dev/staging 开 |
| 所有错误都返回 `codes.Internal` | 状态码错了重试逻辑就废了。`Unavailable` 才触发重试 |
| RPC 参数用裸类型 `string` | 以后加不了字段。用 Request/Response 包装消息 |
| 不注册健康检查 | K8s 无法判断就绪状态，部署时杀 pod |
| 忽略 context 取消 | 调用方放弃了，服务端还在干。长操作里检查 `ctx.Err()` |
| 服务端不内嵌 `UnimplementedXxxServer` | proto 新增 RPC 后旧代码编译不过。内嵌它获得前向兼容 |
| `protoc --go_opt=paths=source_relative` 输出位置和 `go_package` 对不上 | 想按 go_package 落盘用 `--go_opt=module=<模块名>` |

---

## 附：示例项目完整结构

按本文走完后的项目长这样：

```
grpc-demo/
├── go.mod                    # module github.com/kris/grpc-demo
├── proto/user/v1/user.proto  # 服务与消息定义
├── gen/user/v1/              # protoc 生成代码
│   ├── user.pb.go
│   └── user_grpc.pb.go
├── server/
│   ├── main.go               # 服务端：四种 RPC + 拦截器 + 健康检查 + 优雅退出
│   └── main_test.go          # bufconn 单测：状态码断言 + 四种流模式
└── client/
    └── main.go               # 客户端：deadline + metadata + 四种调用模式
```

运行验证：

```bash
go run ./server &    # 或 build 后后台运行
go run ./client
go test ./server/    # bufconn 单测
```
