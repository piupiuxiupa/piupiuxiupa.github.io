---
title: multipass
date: 2024-07-08 11:02:11
tags: tools
---

# Multipass

> official site: https://multipass.run/docs

## 启动前配置

```bash
## 开启mount挂载，方便拿或传文件到服务器，默认false
multipass set local.privileged-mounts=true
## 选择虚拟机驱动，hyperv/virtualbox
## win专业版可用hyperv，家庭版只能virtualbox
multipass set local.driver=hyperv
## 设置默认操作实例
multipass get client.primary-name=primary
```

## 启动实例

```bash
multipass launch -n $name -d 10G -m 2G -c 2 --bridged --mount <local-path>:<instance-path>
```

> 如果启动不了可以尝试重启multipass

## 调整实例配置

```bash
multipass stop handsome-ling
multipass set local.handsome-ling.cpus=4
multipass set local.handsome-ling.disk=60G
multipass set local.handsome-ling.memory=7G
```

## 删除

```bash
## 删除并清除
## delete 相当于把实例转为删除状态 
## -p or --purge 删除并清除
multipass delete $instance -p
## purge
multipass purge $instance
```

## IPv4 N/A

- 重启实例

  ```bash
  multipass restart $instance
  ```

- 停止后再启动实例

  ```bash
  multipass stop $instance
  multipass start $instance
  ```

- 重启multipass服务

  - 可直接杀进程

  ```powershell
  taskkill /f /pid $pid
  ```

- 状态suspending ，尝试suspend实例

  ```bash
  multipass suspend $instance
  ```

- 状态suspended， shell连接即可

  ```bash
  multipass shell $instance
  ```

  