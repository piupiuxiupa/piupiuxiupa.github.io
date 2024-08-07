---
title: Shell 实用技巧
date: 2024-08-06 20:06:47
tags: linux
---
# Shell 实用技巧
1. **保存多个命令的输出**
    - 使用`{}`将命令组合在一起，然后重定向
        - 花括号实际上是保留字，两侧必须有空白字符，闭合括号前面的命令的分号不能少
    - 使用`()`将命令放入子shell重定向输出

    用子shell的方式不会改变当前shell的环境，比如下面的命令组包含`cd ..`，在用花括号的方式中会切换当前路径，而子shell方式中不会切换当前shell的路径，所有的命令都在子shell当中执行，不影响当前shell环境。

    ```bash
    ## 可以用{} or ()
    { ls; cd ..; pwd; ls; } > /tmp/all.out
    (ls; cd ..; pwd; ls) > /tmp/all.out

    test $result -eq 1 && { echo "Result" is 1; exit 0; } \
        || { echo "Run Away!"; exit 120; }
    ```

2. **跳过文件标题**
```bash
## 显示后十行
tail -n 10
tail -10 
## 跳过第一行, 可用于一些跳过标题的情况
tail -n +2
```

3. **使用或替换内建命令与外部命令**
```bash
## type 可查看命令属于哪种,builtin属于内建命令
root@localhost:~# type cd
cd is a shell builtin

## 可查看有哪些内建命令
enable -a
## 关闭shell内建命令
enable -n $cmd
## 忽略shell函数的ls,执行内建ls命令
command ls
```
4. **交换STDERR和STDOUT**
```bash
./myscript 3>&1 1>&2 2>&3

## 操作完成后关闭文件描述符3
./myscript 3>&1 1>stdout.logfile 2>&3- | tee -a stderr.logfile
```

5. **避免意外覆盖文件**
```bash
## 告诉bash重定向输出时不要覆盖任何现有文件
set -o noclobber
## 关闭该选项
set +o noclobber
## 使用 >| 重定向输出。即便是设置了 noclobber，bash 也会忽略该选项，并覆盖文件
echo rewrite >| tmp.txt
```
> noclobber 选项仅针对 shell 重定向输出时的文件覆盖行为。它并不能阻止其他程序覆盖文件

6. **将数据与脚本存放在一起**
```bash
$ cat ext
#
# 下面是here-document
#
grep $1 <<EOF
mike x.123
joe  x.234
sue  x.555
pete x.818
sara x.822
bill x.919
EOF

$ cat donors
#
# 里面的$100中的$1会被误解释为参数$1, 因此需要加''或\EOF
# 
grep $1 <<'EOF'
# 捐赠人及其捐赠额
pete $100
joe  $200
sam  $ 25
bill $ 9
EOF

$ cat myscript.sh
## 使用 <<-，然后就可以在每行的开头用制表符（仅限制表符！）缩进shell 脚本中的 here-document 部分
...
    grep $1 <<-'EOF'
        lots of data
        can go here
        it's indented with tabs
        to match the script's indenting
        but the leading tabs are
        discarded when read
        EOF
    ls
...
```
7. **until, select**

    until 循环用于在条件为假时重复执行一组命令。与 while 循环相反，它在每次迭代开始时检查条件。

    select 循环用于创建一个简单的菜单选择。它会显示一个菜单并等待用户输入选择，然后根据选择执行相应的命令。
    select 语句能够轻松地在 STDERR 上为用户生成编号列表。
    用户所输入的选项编号保存在 $REPLY 中，选项值保存在 select 语句指定的变量中。

8. **同时执行多个命令**
```bash
## 3 个命令的输出会在屏幕上交错出现
## 最后一条命令不在后台运行
ls & date & cd 
## 三条命令都在后台运行
ls & date & cd &
```