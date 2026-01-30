---
title: Git 高级使用技巧
date: 2026-01-30 14:48:00
tags: tools
categories: Notes
excerpt: Git高级操作技巧和最佳实践
---

# Git 高级使用技巧

在掌握了Git的基本操作后，让我们深入了解一些高级功能和最佳实践，这些技巧将帮助您更高效地管理代码版本。

## 1. Git Hooks - 自动化工作流

Git Hooks 是一系列脚本，可在特定事件发生时自动执行，如提交、推送等。

### 常用钩子：

```bash
# pre-commit: 提交前执行，常用于代码质量检查
# .git/hooks/pre-commit
#!/bin/sh
# 检查是否有未格式化的Python代码
if git diff --cached --name-only | grep "\.py$" > /dev/null; then
    echo "Checking Python code formatting..."
    black --check $(git diff --cached --name-only | grep "\.py$")
    if [ $? -ne 0 ]; then
        echo "Please format your Python code with 'black'"
        exit 1
    fi
fi

# pre-push: 推送前执行
# .git/hooks/pre-push
#!/bin/sh
# 检查是否推送到了main分支
remote="$1"
url="$2"

while read local_ref local_sha remote_ref remote_sha; do
    if [ "$remote_ref" = "refs/heads/main" ]; then
        echo "Pushing to main branch is not allowed!"
        exit 1
    fi
done
```

## 2. Git Submodules - 管理外部依赖

当项目需要引用其他Git仓库时，使用子模块管理：

```bash
# 添加子模块
git submodule add https://github.com/user/repo.git path/to/submodule

# 克隆包含子模块的项目
git clone --recursive https://github.com/user/project.git

# 更新子模块
git submodule update --remote

# 初始化子模块
git submodule init
git submodule update
```

## 3. Git Worktrees - 并行开发多个分支

Worktrees 允许您在同一仓库的不同分支上同时工作：

```bash
# 创建一个新的工作树
git worktree add ../project-feature feature-branch

# 查看所有工作树
git worktree list

# 删除工作树
git worktree remove ../project-feature
```

## 4. Git Reflog - 恢复意外丢失的提交

当不小心丢失提交时，使用reflog找回：

```bash
# 查看引用日志
git reflog

# 恢复到特定的HEAD状态
git reset --hard HEAD@{1}

# 查看特定分支的reflog
git reflog show feature-branch

# 恢复被删除的分支
git reflog | grep branch-name
git branch recovered-branch <commit-hash>
```

## 5. Git Aliases - 自定义命令别名

简化常用Git命令：

```bash
# 配置别名
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.unstage 'reset HEAD --'
git config --global alias.last 'log -1 HEAD'
git config --global alias.visual '!gitk'

# 复杂别名
git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"
```

## 6. Git Rebase 进阶用法

### 交互式变基 (Interactive Rebase)

```bash
# 修改最近3个提交
git rebase -i HEAD~3

# 重新排序提交
# squash: 合并到前一个提交
# fixup: 合并到前一个提交（丢弃提交信息）
# drop: 删除提交
# edit: 编辑提交
# reword: 修改提交信息
```

### 基于上游分支的变基

```bash
# 保持特性分支与主分支同步
git checkout feature-branch
git rebase main

# 处理变基冲突
git rebase --continue    # 解决冲突后继续
git rebase --abort      # 放弃变基
git rebase --skip       # 跳过当前提交
```

## 7. Git Filter-Branch - 重写历史

当需要大规模修改提交历史时：

```bash
# 删除文件及其历史
git filter-branch --force --index-filter \
"git rm --cached --ignore-unmatch path/to/large-file.zip" \
--prune-empty --tag-name-filter cat -- --all

# 修改作者信息
git filter-branch --force --env-filter '
if [ "$GIT_COMMITTER_NAME" = "Old Name" ]
then
    export GIT_COMMITTER_NAME="New Name"
    export GIT_COMMITTER_EMAIL="new.email@example.com"
fi
if [ "$GIT_AUTHOR_NAME" = "Old Name" ]
then
    export GIT_AUTHOR_NAME="New Name"
    export GIT_AUTHOR_EMAIL="new.email@example.com"
fi
' --tag-name-filter cat -- --branches --tags
```

## 8. Git LFS (Large File Storage)

管理大型文件：

```bash
# 安装Git LFS
git lfs install

# 跟踪特定类型的文件
git lfs track "*.psd"
git lfs track "*.zip"

# 查看LFS文件
git lfs ls-files
```

## 9. Git Sparse Checkout - 部分检出

只检出仓库中的部分文件：

```bash
# 启用稀疏检出
git sparse-checkout init --cone

# 只检出特定目录
git sparse-checkout set path/to/directory

# 检出多个路径
git sparse-checkout set path1 path2 path3

# 恢复完整检出
git sparse-checkout disable
```

## 10. Git Subtree - 合并外部项目

将外部项目合并到当前项目：

```bash
# 添加子树
git subtree add --prefix=path/to/subtree https://github.com/user/repo.git main

# 推送更改到子树
git subtree push --prefix=path/to/subtree origin main

# 从子树拉取更新
git subtree pull --prefix=path/to/subtree https://github.com/user/repo.git main
```

## 11. 高级合并策略

### Ours/Mine 策略

```bash
# 合并时保留当前分支的版本
git merge -X ours <branch>

# 合并时保留另一个分支的版本
git merge -X theirs <branch>
```

### 自定义合并驱动

```bash
# .gitattributes
*.conf merge=ours
*.lock merge=union

# 配置合并驱动
git config merge.ours.driver true
git config merge.union.driver "cat %B %O >> %A"
```

## 12. Git Patches - 交换代码变更

创建和应用补丁：

```bash
# 创建补丁
git diff > patch.diff
git format-patch HEAD~1  # 为最近一次提交创建补丁

# 应用补丁
git apply patch.diff
git am < patch.diff  # 应用格式化的补丁
```

## 13. Git Bisect 进阶用法

自动化二分查找：

```bash
# 自动化查找
git bisect start HEAD v1.0.0
git bisect run python test_script.py

# 跳过无法测试的提交
git bisect skip
```

## 14. Git 配置最佳实践

```bash
# 配置自动换行处理
git config --global core.autocrlf input  # Unix/Linux/macOS
git config --global core.autocrlf true   # Windows

# 配置默认编辑器
git config --global core.editor "vim"

# 配置差异工具
git config --global diff.tool vimdiff
git config --global merge.tool vimdiff

# 配置提交模板
git config --global commit.template ~/.gitmessage
```

## 15. Git 安全最佳实践

```bash
# 验证提交签名
git log --show-signature

# 验证标签签名
git verify-tag <tag-name>

# 使用GPG签名提交
git config --global commit.gpgsign true
git config --global user.signingkey <key-id>
```

## 16. Git 性能优化

```bash
# 压缩仓库
git gc --aggressive --prune=now

# 清理未引用的对象
git fsck

# 优化推送
git config --global push.default simple
```

## 17. Git 工作流最佳实践

### Git Flow 模型

```bash
# 使用git-flow工具
git flow init
git flow feature start feature-name
git flow feature finish feature-name
git flow release start 1.0.0
git flow release finish 1.0.0
```

### GitHub Flow 模型

```bash
# 简化的工作流
git checkout -b feature-branch
# 开发...
git add .
git commit -m "Add feature"
git push origin feature-branch
# 创建Pull Request
# 合并后删除分支
git branch -d feature-branch
git push origin --delete feature-branch
```

掌握这些高级Git技巧将显著提升您的开发效率和代码管理能力。记住，在使用破坏性命令（如rebase、filter-branch）前务必备份重要数据。