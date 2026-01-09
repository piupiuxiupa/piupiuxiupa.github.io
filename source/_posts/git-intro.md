---
title: Git 基本用法
date: 2024-07-08 13:33:02
tags: tools
categories: Notes
excerpt: Git基础操作笔记
---

# Git 基本用法

> git的结构就像一棵树一样
>
> main主干，dev，test，issue ... 都是分支树杈

## pull & push

```bash
## 初始化一个git仓
git init 
git remote add $alias $url
git branch -M master(main)
## 查看远程仓库
git remote -v
git push -u origin master 

## 远程分支名:本地分支名
## 旧版为master ，新版默认main
git pull origin master:master 

## 修改git地址
git remote set-url origin $url
## 已跟踪文件取消跟踪，配合.gitignore，之后推送都忽略
git rm -r --cached $file
```

## reset

```bash
git ls-files ## 查看暂存区文件

git reset --soft ## 工作区和暂存区都不清空
git reset --hard ## 工作区暂存区全部清空
git reset --mixed ## 保留工作区，清理暂存区，默认选项

git reset HEAD^ ## 回退到上一次提交

git log ## 公共提交记录
git reflog ## 本地操作记录

## 一般不建议用reset回退
## 如果用了reset回退，需要强制push
git log                 # 找到目标 commit hash
git reset --hard <hash>
git push -f             # 强制推送覆盖远程（慎重操作）
```

## switch/checkout

```bash
## 创建一个分支
git branch $branch 
## 可以切换分支也可以切换至某次提交或恢复某次提交的文件，如分支与文件名相同会产生冲突，默认是切换分支
git checkout $branch($commit) 
## 切换某分支的某次提交
git checkout -b $branch $commit 
## 仅切换至某分支
git switch $branch
## 误删除文件，还未add时
## 放弃所有修改，恢复全部
git checkout .
git checkout -- *
## 放弃某个文件，恢复某个文件
git checkout -- $filename
## 如果一个分支提交过后可以用-d选项删掉分支，没有提交想删除需用-D强制删除分支
git branch -d $branch
git branch -D $branch
## 图表形式查看分支结构
git log --graph --oneline --decorate --all
```

## merge

```bash
# 先切换至main分支
# 然后再执行如下命令合并develop分支到main分支
git merge $branch
## 两个分支修改了同一个文件的同一处位置会产生冲突
## 查看冲突具体内容，修改文件中的冲突后再提交
git diff
## 中止合并
git merge --abort
```

## rebase

```bash
## 如果当前有两个分支，main和dev
## 当前处在dev分支时，rebase会把dev分支接在main分支最新一次提交后
git switch dev
git rebase main
## 当前处在main分支时，rebase会把main分支接在dev分支最新一次提交后
git switch main
git rebase dev
```

### rebase 手动修改合并提交

```bash
# commit_id 指要修改的某个commit的前一个历史commit
git rebase -i $commit_id
# 会进入一个编辑页，包含输入的commit_id的下一个commit一直到HEAD

# 类似如下
pick d1ac219 # 'Update post'
pick 594574b # 'Update post'
pick e76b482 # update
pick 32be783 # Fix: Modify some content
pick 416d78e # update content
pick 415c264 # update
pick d1ddbce # update

# 通过将内容前的pick改成drop来删除某次commit
# 将某些commit改成squash，来合并最近的几次提交
# 此类rebase徐压迫远程仓库开启强制push才能推送
# 否则只能本地生效
# 推荐用于本地合并提交后推送，保证commit内容整洁，已推送内容不建议修改
```

## stash

```bash
# 暂存当前工作区的所有修改
git stash

# 查看stash列表
git stash list

# 恢复最近一次stash的内容
git stash pop

# 恢复指定的stash内容
git stash apply stash@{0}
```

## revert

```bash
## 撤销某次提交
git log                 # 找到目标 commit hash
git revert <hash>       # 自动生成撤销该提交内容的 commit
git push
```

## cherry-pick

```bash
# 只合并某个分支里的 某一次提交
# 切换到目标分支，比如 main
git checkout main  

# 把某次提交合并过来（commit id 用 git log 查）
git cherry-pick <commit-id>  

# 推送到远程
## 前提是有权限推送到main分支
git push origin main
```

## bisect

二分查找某次有bug的提交

```bash
# 假设当前HEAD有错误，但在某次commit时还是正常的
# 开启二分查找
git bisect start
# 标记当前HEAD为坏commit
git bisect bad HEAD
# 标记已知是好的commit
git bisect good $commit_id
# 此时git会输出好与坏中间的那个commit
# 然后测试commit，标记好坏，重复步骤即可找到引入bug的第一个提交
# 结束查找
git bisect reset

# 如果中间的某次提交是不完整的工作提交可以使用skip跳过
git bisect skip

# 查看日志
git bisect log
# 重放日志
git bisect log > bisect.log
git bisect replay bisect.log
# 查看进度
git bisect status
# 可视化查看二分查找过程
git log --oneline --graph --boundary $(git bisect rev-list --bisect-vars | tr ' ' '\n' | grep -E '^(BISECT_START|BISECT_BAD|BISECT_GOOD)')
```
