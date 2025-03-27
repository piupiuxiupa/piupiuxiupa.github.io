---
title: GitLab 常用操作与问题记录
date: 2024-11-29 09:49:00
tags: tools
categories: Posts
---

# GitLab 常用操作与问题记录

## 修改密码
```bash  

gitlab-rails console

user = User.find_by_username('root')

user.password = 'NewSecurePassword'

user.password_confirmation = 'NewSecurePassword'

user.save!
``` 

## External url

影响到点击克隆按钮显示的地址

打开 GitLab 的主配置文件：/etc/gitlab/gitlab.rb

修改
```ini
external_url 'http://your.gitlab.domain'
```
  

重新配置 gitlab 并重启
```bash
sudo gitlab-ctl reconfigure

sudo gitlab-ctl restart
```
  

3. 检查 SSH 和 HTTP/HTTPS 配置

    ```ini
    gitlab_rails['gitlab_shell_ssh_host'] = 'your.gitlab.domain'
    ```
- 如果依然显示错误可尝试清理缓存
    ```bash
    sudo gitlab-rake cache:clear
    ```
- 检查服务状态
    ```bash
    sudo gitlab-ctl status
    ```

    Nginx 配置冲突

    • 确保没有其他服务占用 GitLab 的端口。

    • 检查 Nginx 配置是否生成正确：
    ```bash
    sudo gitlab-ctl show-config | grep nginx
    ```

## GitLab 升级迁移

跨多个版本升级时，根据官方文档说明需要逐步升级到最新版本

- GitLab 8: `8.11.Z` > `8.12.0` > `8.17.7`
- GitLab 9: `9.0.13` > `9.5.10`
- GitLab 10: `10.0.7` > `10.8.7`
- GitLab 11: `11.0.6` > [`11.11.8`](https://archives.docs.gitlab.com/15.11/ee/update/index.html#1200)
- GitLab 12: `12.0.12` > [`12.1.17`](https://archives.docs.gitlab.com/15.11/ee/update/index.html#1210) > [`12.10.14`](https://archives.docs.gitlab.com/15.11/ee/update/index.html#12100)
- GitLab 13: `13.0.14` > [`13.1.11`](https://archives.docs.gitlab.com/15.11/ee/update/index.html#1310) > [`13.8.8`](https://archives.docs.gitlab.com/15.11/ee/update/index.html#1388) > [`13.12.15`](https://archives.docs.gitlab.com/15.11/ee/update/index.html#13120)
- GitLab 14: [`14.0.12`](https://archives.docs.gitlab.com/15.11/ee/update/index.html#1400) > [`14.3.6`](https://archives.docs.gitlab.com/15.11/ee/update/index.html#1430) > [`14.9.5`](https://archives.docs.gitlab.com/15.11/ee/update/index.html#1490) > [`14.10.5`](https://archives.docs.gitlab.com/15.11/ee/update/index.html#14100)
- GitLab 15: [`15.0.5`](https://archives.docs.gitlab.com/15.11/ee/update/index.html#1500) > [`15.1.6`](https://archives.docs.gitlab.com/15.11/ee/update/index.html#1510) (for GitLab instances with multiple web nodes) > [`15.4.6`](https://archives.docs.gitlab.com/15.11/ee/update/index.html#1540) > [latest `15.Y.Z`](https://gitlab.com/gitlab-org/gitlab/-/releases)

GitLab 15 includes the following required upgrade stops:
- 15.0.5.

- 15.1.6. GitLab instances with multiple web nodes.

- 15.4.6.

- 15.11.13. The latest GitLab 15.11 release.

GitLab 16 includes the following required upgrade stops:

- 16.0.9. Instances with lots of users or large pipeline variables history.

- 16.1.7. Instances with NPM packages in their package registry.

- 16.2.10. Instances with large pipeline variables history.

- 16.3.8.

- 16.7.z. The latest GitLab 16.7 release.

- 16.11.z. The latest GitLab 16.11 release.

GitLab 17 includes the following required upgrade stops:

- 17.3.z. The latest GitLab 17.3 release.

- 17.5.z. The latest GitLab 17.5 release.

- 17.8.z. Not yet released.

- 17.11.z. Not yet released.


```bash
## 升级前请务必先备份
## 关闭selinux
setenforce 0
## 永久关闭selinux，并重启
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
rpm -Uvh gitlab-ce-11.11.8-ce.0.el7.x86_64.rpm
```
> https://gitlab.cn/support/toolbox/upgrade-path/

## GitLab 基础操作

```bash
## 获取 gitlab 版本信息
gitlab-rake gitlab:env:info
## 查看 gitlab 运行状态
gitlab-ctl status
## 查看 gitlab 运行日志
gitlab-ctl tail
## 重启 gitlab
gitlab-ctl restart
## 备份 gitlab
gitlab-backup create
## 旧版本备份
gitlab-rake gitlab:backup:create
## 恢复备份
## timestamp 填时间戳而不是文件名
gitlab-backup restore BACKUP=timestamp
gitlab-rake gitlab:backup:restore BACKUP=TIMESTAMP
## 重新配置gitlab
gitlab-ctl reconfigure
## 检查 gitlab 是否正常运行
gitlab-rake gitlab:check
## 检查 gitlab 仓库是否正常运行
gitlab-rake gitlab:repos:check
gitlab-rake gitlab:check SANITIZE=true
## 启用详细模式
gitlab-rake gitlab:check VERBOSE=true
gitlab-ctl status
## 重新加载配置
gitlab-ctl reconfigure
```
## gitlab 升级 14 版本迁移问题

```bash
## 迁移项目到新的存储路径
gitlab-rake gitlab:storage:migrate_to_hashed
## 使用之后会提示 Found 6 projects using Legacy Storage.
## 查看使用 Legacy Storage 的项目数量 
gitlab-rake gitlab:storage:legacy_projects
## 查看这六个具体是什么项目
## 查看使用旧版本的项目
gitlab-rake gitlab:storage:list_legacy_projects
```

### read-only状态导致的迁移问题

```ruby
# 进入 Rails 控制台
gitlab-rails console

# 查询 项目 read-only 打开的 
projects = Project.where(repository_read_only: true)

# 关闭 项目的 read-only
projects.each do |p|
  p.update!(repository_read_only:nil)
end
```

```bash
# 存储库迁移
gitlab-rake gitlab:storage:migrate_to_hashed

# 查看所列出项目总数，与页面显示数量进行对比
gitlab-rake gitlab:storage:hashed_projects

# 查看，全部迁移成功以下两条命令应该为 0 
gitlab-rake gitlab:storage:legacy_projects
gitlab-rake gitlab:storage:legacy_attachments

# 全部迁移成功，以下两条命令应该没有输出
gitlab-rake gitlab:storage:list_legacy_projects
gitlab-rake gitlab:storage:list_legacy_attachments
```

###  @hashed 目录迁移失败

```bash
# 查看具体哪些项目迁移失败
gitlab-rake gitlab:storage:list_legacy_projects
# 在日志中搜索该项目，查看具体路径
grep 'xxxx' /var/log/gitlab/sidekiq/current
# 发现报错Repository cannot be moved from '@hashed/b2/81/b281bcxxxxxxxx.wiki' to
# 查看目录下是否有 xxx.wiki的目录
ls -l /var/opt/gitlab/git-data/repositories/@hashed/98/96/
# 发现有该目录，将其移动到 /tmp 目录下
sudo mv /var/opt/gitlab/git-data/repositories/@hashed/b2/81/b281bcxxxxxxx /tmp/
# 然后再执行迁移
sudo gitlab-rake gitlab:storage:migrate_to_hashed
```

### 后台迁移任务没完成

> https://archives.docs.gitlab.com/15.11/ee/update/background_migrations.html
>
> https://archives.docs.gitlab.com/15.11/ee/update/#upgrade-paths

14版本后可在页面中看见后台迁移任务。
在管理中心监控中后台迁移可看见未完成迁移任务。

升级中可能因为Background Migration任务没有执行完，导致升级失败。
需要等待任务执行完成后，再进行升级。

## 升级 16 大版本配置项已废弃的问题
如 gitlab_rails['gitlab_default_can_create_group'] has been deprecated since 15.5 and was removed in 16.0.

解决方法：在 `gitlab.rb` 中将提示配置项删除或注释

```bash
# 修改完配置项后，重新配置，重启
gitlab-ctl reconfigure
gitlab-ctl restart
```

## 后台任务进度未知问题

运行`gitlab-psql` 进入控制台

```postgresql
# 获取正在迁移的任务 ID
SELECT
 id,
 job_class_name,
 table_name,
 column_name,
 job_arguments
FROM batched_background_migrations
WHERE status NOT IN(3, 6);

# 查看任务进度
# 将 xx 替换为查询到的 ID
# 多次运行查询如果没有新行添加说明完成或暂停
# 有变化则仍在运行
SELECT
 started_at,
 finished_at,
 finished_at - started_at AS duration,
 min_value,
 max_value,
 batch_size,
 sub_batch_size
FROM batched_background_migration_jobs
WHERE batched_background_migration_id = XX
ORDER BY id DESC
limit 10;
```

## 配置 ldap

```rb
gitlab_rails['ldap_enabled'] = true
gitlab_rails['ldap_servers'] = YAML.load <<-EOS
  main: # 可以配置多个 LDAP 服务器，这里是主服务器
    label: 'LDAP' # 显示在登录页面的名称
    host: 'ldap.example.com'
    port: 389
    uid: 'sAMAccountName' # 用于用户登录的属性，例如 AD 中的 sAMAccountName
    bind_dn: 'cn=admin,dc=example,dc=com' # 用于绑定的服务账户
    password: 'password' # 服务账户密码
    encryption: 'plain' # 加密方式：plain、start_tls 或 simple_tls
    verify_certificates: true
    smartcard_auth: false
    active_directory: true # 如果使用的是 Active Directory，设置为 true
    allow_username_or_email_login: false
    lowercase_usernames: false
    block_auto_created_users: false
    base: 'dc=example,dc=com' # 用户搜索的 Base DN
    user_filter: '' # 额外的筛选条件，例如只允许特定组的用户访问
    group_base: ''
    admin_group: '' # 如果有管理员组，可以设置此项
    sync_ssh_keys: false # 是否同步用户的 SSH 密钥
EOS
```

```bash
# 重新配置
gitlab-ctl reconfigure
# 重启
gitlab-ctl restart
# 测试，如果配置成功，输出会显示用户和组信息。
gitlab-rake gitlab:ldap:check
```

## redis 开启RDB 导致 oom gitlab 崩溃

公司使用了云存储 nas，磁盘性能不好，导致 gitlab 出现问题。

gitlab 自带的 redis 配置默认开启 RDB 持久化，如果磁盘性能较差，可能导致 oom 崩溃问题。

- Redis 在进行 RDB 持久化时，会创建一个子进程（通过 fork），父进程继续提供服务，子进程负责写入 RDB 文件。
- 如果内存使用过高，fork 操作会耗费大量内存资源，因为操作系统需要复制父进程的页表。
- 在生成 RDB 文件期间，如果 Redis 正在进行大量写操作，内存页面会频繁被修改，导致操作系统必须复制这些页面，增加了内存和 CPU 的开销。
- 如果磁盘性能较差，RDB 文件的写入速度较慢，会让子进程占用更多时间，从而使父进程的 COW 操作被放大。

```bash
## 默认配置
save 900 1
save 300 10
save 60 10000
appendonly no

## 修改配置
save ""
appendonly yes
```
> 最好的方法当然是换性能好的磁盘
>
> 而且磁盘性能不行会出现各种奇怪的问题

## 迁移后 webhooks 访问 500

旧版本 webhook 与新版本不兼容导致

1. 找到项目 id

2. 在数据库中查询webhook的 id： `select id from web_hooks where project_id=2016;`

3. 通过接口请求删除 webhooks
    ```bash
    curl --header "Private-Token: 4Yn1dffiTDpy5c2xvN8X" -X DELETE http://gitlab.com/api/v4/projects/<project_id>/hooks/<webhook_id>
    ```