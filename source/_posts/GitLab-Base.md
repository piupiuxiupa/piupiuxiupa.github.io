---
title: GitLab Base
date: 2024-11-29 09:49:00
tags: tools
---

# GitLab Base

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