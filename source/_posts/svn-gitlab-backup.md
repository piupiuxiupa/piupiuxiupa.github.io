---
title: SVN GitLab 备份与还原
date: 2024-10-15 11:09:29
tags: tools
categories: Posts
---

# SVN & GitLab 备份与还原


## svn

- 热备
  - 通过svn的hook达到热备效果
  - svnsync
- 冷备
  - 通过 svnadmin dump 导出备份数据
  - rsync或scp 传输备份数据到其他机器

### svnsync热备
#### 创建svn 仓库
```bash
# 接创建路径
svnadmin create /home/svn
```
#### 修改svn配置文件
```
# svnserver.conf
# 取消如下配置注释
anon-access = none
auth-access = write
password-db = passwd
authz-db = authz
```
#### 添加admin账号权限
```ini
# authz
[groups]
admin = <username>

[/]
@admin = rw
```
#### 添加admin账号密码
```ini
# passwd
[users]
admin = <password>
```

#### 启动svn服务
```bash
svnserve -d -r /home/svn
```

#### svnsync备份
```bash
# 在文件第二行加入 exit 0
cp pre-revprop-change.tmpl pre-revprop-change

# 初始化备份仓库
svnsync init svn://<backup_ip> svn://<svn_ip>

# 会提示输入账号密码
svnsync sync svn://<backup_ip>
# 或
svnsync sync --non-interactive svn://<backup_ip> --sync-username <username> --sync-password <password> --source-username <username> --source-password <password>
```

#### 将svnsync写入post-commit hook

钩子脚本的具体写法和shell脚本写法是一样的。钩子脚本就是被某些版本库事件触发的程序。

post-commit 在提交完成成功创建版本之后执行该钩子，提交已经完成，不可更改。

pre-commit 提交完成前触发执行该脚本

```bash
# hooks目录
cp post-commit.tmpl post-commit
# 将如下命令写入 post-commit
svnsync sync --non-interactive svn://<backup_ip> --sync-username <username> --sync-password <password> --source-username <username> --source-password <password>
```
### 冷备
#### svnadmin dump

```bash
## 查看版本
svnlook youngest /home/svn

## youngest_num填入svnlook查到值
svnadmin dump /home/svn -r 0:$youngest_num --incremental > /home/svn_backup/svn.dump
# 可以尝试写成增量导出脚本，之后用增量文件会比全量快
## 恢复备份文件
svnadmin load /home/svn < /home/svn_backup/svn.dump

## 然后做成 rsync或者 scp传输 dump文件到备份服务器
```

## gitlab备份

```bash
gitlab-rake gitlab:backup:create
# 备份文件在/var/opt/gitlab/backups/目录下

## 修改备份文件权限
chmod 777 /var/opt/gitlab/backups/*.tar
## 停止服务
gitlab-ctl stop unicorn
gitlab-ctl stop sidekiq

## 恢复
gitlab-rake gitlab:backup:restore BACKUP=1481434654

## 启动服务
gitlab-ctl restart
```