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
