---
title: Shell 实用技巧（一）
date: 2024-08-06 20:06:47
tags: linux
categories: Notes
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

9. **在shell脚本中嵌入文档**

```bash
:<<'DOC'
    Write some notes.
DOC
```

10. **导出变量**

> bash 中一切都是字符串

将希望传给其他脚本的变量导出
要想查看所有已导出的变量，敲入命令 env（或者内建命令export -p）

```bash
export MYVAR
export NAME=value
```
env 生成的列表是 set 生成的列表的子集，因为并非所有变量都被导出了。

11. **处理包含空格的参数**

将所有可能包含文件名的命令参数全部加上引号。引用变量时，将其放入双引号中。
```bash
$ cat quoted.sh
# note the quotes
ls -l "${1}"
$
$ ./quoted.sh "Oh the Waste"
-rw-r--r--  1 smith users 28470 2007-01-11 19:22 Oh the Waste
$
```

12. **获取默认值**

如果指定参数（这里是 $1）不存在或为空，则将运算符之后的内容（本例为 /tmp）作为值。否则，使用已经设置好的参数值

```bash
## 如果值为路径直接写，或者加 ""双引号，用单引号会导致路径失效
${1:-/tmp}
```
13. **设置默认值**

首次引用 shell 变量时，如果该变量没有值，则使用赋值运算符为其赋值

赋值运算符有一个重要的例外：不能对位置参数（如 $1或 $*）赋值

```bash
cd ${HOME:=/tmp}
```
> := 执行赋值操作，同时返回运算符右侧的值。:- 只做了前者一半的工作：返回值，但不赋值

14. **使用空值作为有效的默认值**

如果写成不带冒号的 ${HOME=/tmp}，那么赋值操作仅会在该变量不存在的情况下（从未设置或已明确删除）发生

15. **不只使用字符串常量作为默认值**

能出现在 shell 变量引用右侧的内容并不局限于字符串常量。

```bash
cd ${BASE:="$(pwd)"}
```

16. **对不存在的参数输出错误消息**

使用 set -u 命令，​“在变量替换时，将不存在的变量视为一种错误”​。

```bash
 #!/usr/bin/env bash
# 实例文件：check_unset_parms
#
USAGE="usage: myscript scratchdir sourcefile conversion"
FILEDIR=${1:?"Error. You must supply a scratch directory."}
FILESRC=${2:?"Error. You must supply a source file."}
CVTTYPE=${3:?"Error. ${USAGE}"}
 
 ## 还可以在其中执行其他命令
 CVTTYPE=${3:?"Error. $USAGE. $(rm $SCRATCHFILE)"}
```

17. **变量字符串替换**

```bash
# 从字符串name的索引位置start开始，返回长度为number的子串
# 如果子串为空格分隔的字符串，转为数组，比如 name="jack rose lucy", name=(${name})
# 然后用 ${name[@]:0:1}形式进行引用，不然会按字符串数量计数取子集
# 如果不转换为数组，直接 ${name:0:1} 会输出 j
${name:start:number}
# 返回字符串长度
${#name}
## 从字符串起始位置开始，删除匹配pattern的最短子串
${name#pattern}
## 从字符串起始位置开始，删除匹配pattern的最长子串
${name##pattern}
## 从字符串结束位置开始，删除匹配pattern的最短子串
${name%pattern}
## 从字符串结束位置开始，删除匹配pattern的最长子串
${name%%pattern}
## 替换字符串中第一次出现的pattern
${name/pattern/string}
## 替换字符串中出现的所有pattern
${name//pattern/string}

## 变量不为空，替换为逗号
${LIST:+,}
```

> FILE=${FULLPATHTOFILE##*/}
> 等价于（结尾处不含/时）
> basename $FULLPATHTOFILE

18. **变量字符串大小写转换**

```bash
# 转换成小写
${name,,}
# 转换成大写
${name^^}
# 首字母转换成大写
${name^}
# 大写转成小写，小写转大写
${name~~}
# 变量内容会转换成小写字母
declare -l name
# 全部大写
declare -u name
# 仅首字母大写
declare -c name
```

19. **简单算数**

用 `$(( ))` 或 `let` 进行整数运算
```bash
##  $(( ))可以用 ** 进行幂运算
COUNT=$((COUNT + 5 + MAX * 2))
let COUNT+='5+MAX*2'

let COUNT+=5
```

20. **测试文件属性**

```bash
# 是否更新（检查文件的修改时间）​。现有文件要比不存在的文件“新”​。
test FILE1 -nt FILE2　　
# 是否更旧。同样，不存在的文件要比现有文件“旧”​。
test FILE1 -ot FILE2　　
# 具有相同设备和 inode 编号（即便由不同链接所指向，也视为相同的文件）​。
test FILE1 -ef FILE2　　
# 测试文件大小是否不为空
test -s FILE
```

21. **用正则表达式匹配**

> bash 3.0 或更高版本

使用 =~ 运算符进行正则表达式匹配。只要能够匹配到某个字符串，就可以在 shell 数组变量 $BASH_REMATCH 中找到模式中的各个部分所匹配到的内容

```bash
#!/usr/bin/env bash
# 实例文件：trackmatch
#
for CDTRACK in *
do
    if [[ "$CDTRACK" =~ "([[:alpha:][:blank:]]*)- ([[:digit:]]*) - (.*)$" ]]
    then
        echo Track ${BASH_REMATCH[2]} is ${BASH_REMATCH[3]}
        mv "$CDTRACK" "Track${BASH_REMATCH[2]}"
    fi
done
```

22. **循环**
```bash
## while 循环
$ svn status bcb
M      bcb/amin.c
?      bcb/dmin.c
?      bcb/mdiv.tmp
A      bcb/optrn.c
M      bcb/optson.c
?      bcb/prtbout.4161
?      bcb/rideaslist.odt
?      bcb/x.maxc

## cut提取从第 8 列开始（一直到行尾）的字符串
svn status mysrc | grep '^?' | cut -c8- |
  while read FN; do echo "$FN"; rm -rf "$FN"; done

## 将输入传给两个变量，最后一个变量获得输入行中剩余的所有内容
svn status mysrc |
while read TAG FN
do
    if [[ $TAG == \? ]]
    then
        echo $FN
        rm -rf "$FN"
    fi
done

## for 循环
## 双括号表明这是算术表达式，在其中引用变量时，不用加 $（但 $1 等位置参数除外）​，只要是 bash 中出现双括号的地方，均是如此
for (( expr1 ; expr2 ; expr3 )) ; do list ; done

for (( i=0, j=0 ; i+j < 10 ; i++, j++ ))
do
    echo $((i*j))
done

## seq 可以生成浮点值，参数依次是起始值、增量、结束值
for fp in $(seq 1.0 .01 1.1)
do
    echo $fp; other stuff too
done

seq 1.0 .01 1.1 |
while read fp
do
    echo $fp; other stuff too
done
## $() 在子 shell 中执行命令

## 在第一个例子的for 循环中，seq 必须先运行完毕，然后用全部的输出结果替换掉 $()。对于冗长的数值序列来说，这种方法的时间开销和内存开销都不小。
## 在第二个例子中，seq 作为单独的命令运行，通过管道将其输出传入 while 循环，后者读取其中的每一行并做相应处理，seq 命令是与 while 并行运行的
```