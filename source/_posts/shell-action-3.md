---
title: Shell 实用技巧（三）
date: 2024-08-15 10:53:19
tags: linux
categories: Notes
---
1. **getopts**
    - `getopts` 是一个内建命令，用于处理命令行选项。
    - 它可以让脚本在运行时接受命令行参数。
    - 它的语法是：`getopts optstring variable`。
    - `optstring` 填入选项字符，如 `abc` 表示选项 a、b 和 c 是合法的选项。
    - 如果是必填项要在选项后面加冒号，如`a:bc`
    - 对于接受参数的选项，参数会保存在 shell 变量 `$OPTARG`中
    - 当选项是不需要输入参数的时候，`$OPTARG`为空
    - `$OPTIND` 初始值为1，每解析一个选项，`$OPTIND` 会加1
    
    ```bash
    #!/usr/bin/env bash
    # 实例文件：getopts_example
    #
    # getopts用法
    #
    aflag=
    bflag=
    while getopts 'ab:' OPTION
    do
        case $OPTION in
            a) aflag=1
            ;;
            b) bflag=1
            bval="$OPTARG"
            ;;
            ?) printf "Usage: %s: [-a] [-b value] args\n" ${0##*/} >&2
            exit 2
            ;;
        esac
    done

    shift $(($OPTIND - 1))

    if [ "$aflag" ]
    then
        printf "Option -a specified\n"
    fi
    if [ "$bflag" ]
    then
        printf 'Option -b "%s" specified\n' "$bval"
    fi
    printf "Remaining arguments are: %s\n" "$*"

    myscript -a -b alt plow harvest reap
    ## 这里解析完选项后 $OPTIND 为 4, 因此$OPTIND - 1，丢弃解析过的选项，剩余的参数为 plow harvest reap
    ```
    - 如果想让脚本输出自己自定义的错误信息，需要在 `getopts` 选项列表前加上`:`, 如`:ab:c` 使`getopts`不输出错误消息
    ```bash
    #!/usr/bin/env bash
    # 实例文件：getopts_custom
    #
    # 解析参数时使用自定义错误消息
    #
    aflag=
    bflag=
    while getopts :ab: FOUND
    do
        case $FOUND in
            a) aflag=1
            ;;
            b) bflag=1
            bval="$OPTARG"
                ;;
            # 缺少参数返回 :
            \:) printf "argument missing from -%s option\n" $OPTARG
                printf "Usage: %s: [-a] [-b value] args\n" ${0##*/}
                exit 2
                ;;
            # 不支持的选项返回 ?
            \?) printf "unknown option: -%s\n" $OPTARG
                printf "Usage: %s: [-a] [-b value] args\n" ${0##*/}
                exit 2
                ;;
            esac >&2
        done
    shift $(($OPTIND - 1))
    ```
2. **用read解析文本**
    ```bash
    #!/usr/bin/env bash
    # 实例文件：parseViaRead
    #
    # 用read语句解析ls -l的输出
    # ls -l的输出类似于以下这样：
    # -rw-r--r--  1 albing users 126 2006-10-10 22:50 fnsize

    ls -l "$1" | { read PERMS LCOUNT OWNER GROUP SIZE CRDATE CRTIME FILE ;
    echo $FILE has $LCOUNT 'link(s)' and is $SIZE bytes long. ;
    }

    # read 将各单词读入数组变量
    read -a MYRAY
    ```

3. **读取整个文件**
    > mapfile 和 readarray
    >
    > 能够将整个文件读入数组，每个数组元素对应文件中的一行

    ```bash
    mapfile -t -s 1 -n 1500 -C showprg -c 100 BIGDATA < /tmp/myfile.data
    # -s 1 跳过第一行输入
    # -n 1500 读取 1500行
    # -t 删除每行行尾的换行符
    # -c 100 每读取 100行调用一次用户自定义函数 showprg, 默认每 5000行调用一次
    # 结果放进数组 BIGDATA
    ```

4. **每次提取一个字符**
    ```bash
    #!/usr/bin/env bash
    # 实例文件：onebyone
    #
    # 从输入中一次解析一个字符

    while read ALINE
    do
        for ((i=0; i < ${#ALINE}; i++))
        do
            ACHAR=${ALINE:i:1}
            # 在这里执行某些操作，如echo $ACHAR
            echo $ACHAR
        done
    done
    ```
5. **提取数据中的特定字段**

    - `cut`
    - `awk`
    - `grep -o` 正则提取

    ```bash
    cut -d: -f1,6,7 /etc/passwd
    cut -d: -f1-3

    # 默认输出分隔符是空格，通过 OFS变量修改
    awk 'BEGIN{FS=":"; OFS="\t"} {print $1 $7 $6}' /etc/passwd

    ## ip地址提取
    ip a | grep -oP '([01]?\d\d?|2[0-4]\d|25[0-5])\.([01]?\d\d?|2[0-4]\d|25[0-5])\.([01]?\d\d?|2[0-4]\d|25[0-5])\.([01]?\d\d?|2[0-4]\d|25[0-5])'
    ```

6. **更新文件中的特定字段**

    ```bash
    $ grep csh /etc/passwd
    root:*:0:0:Charlie &:/root:/bin/csh

    $ sed 's;/csh$;/sh;' /etc/passwd | grep '^root'
    root:*:0:0:Charlie &:/root:/bin/sh

    # 对字段进行运算或修改特定字段中的内容
    $ cat data_file
    Line 1 ends
    Line 2 ends
    Line 3 ends
    Line 4 ends
    Line 5 ends

    $ awk '{print $1, $2+5, $3}' data_file
    Line 6 ends
    Line 7 ends
    Line 8 ends
    Line 9 ends
    Line 10 ends
    # 如果第2个字段中包含'3'，将其修改为'8'并做标记
    $ awk '{ if ($2 == "3") print $1, $2+5, $3, "Tweaked" ; else print $0; }' \
    data_file
    Line 1 ends
    Line 2 ends
    Line 8 ends Tweaked
    Line 4 ends
    Line 5 ends
    ```

7. **修剪空白字符**
    - `shopt -s extglob` 开启扩展通配符匹配
    - 利用 `read`的设计方法
    - `${##} 或 ${%%}` 变量字符串切割

    ```bash
    shopt -s extglob
    ## 开启后可以用 +(), 匹配多个空格符
    ${line##+([[:space:]])}
    ${line%%+([[:space:]])}

    shopt -s extglob

    while IFS= read -r line; do
    echo "None: $line" # 保留所有的空白字符
    echo "Ld: ${line##+([[:space:]])}" # 修剪行首空白字符
    echo "Tr: ${line%%+([[:space:]])}" # 修剪行尾空白字符
    line="${line##+([[:space:]])}" # 修剪行首和……
    line="${line%%+([[:space:]])}" # ……行尾的空白字符
    echo "All: $line" # 显示修剪后的内容
    done < whitespace
    ```

8. **编写安全的shell脚本**

    ```bash
    #!/usr/bin/env bash
    # 实例文件：security_template

    # 设置合理的/安全的路径
    PATH='/usr/local/bin:/bin:/usr/bin'
    # 确保导出$PATH
    \export PATH

    # 清除所有别名。切记：开头的\用于禁止别名扩展
    \unalias -a

    # 清除命令路径散列
    hash -r

    # 将硬性限制设置为0，关闭核心转储（core dumps）
    ulimit -H -c 0 --

    # 设置合理的/安全的IFS（注意，这里使用的语法仅适用于bash和ksh93，不具备可移植性！）
    IFS=$' \t\n'

    # 设置并使用合理的/安全的umask变量
    # 注意，这不会影响已经在命令行上重定向的文件
    # 022产生0755权限，077产生0700权限……
    UMASK=022
    umask $UMASK

    until [ -n "$temp_dir" -a ! -d "$temp_dir" ]; do
        temp_dir="/tmp/meaningful_prefix.${RANDOM}${RANDOM}${RANDOM}"
    done
    mkdir -p -m 0700 $temp_dir \
    || (echo "FATAL: Failed to create temp dir '$temp_dir': $?"; exit 100)

    # 尽全力清除临时文件
    # 注意，一定要先设置好$temp_dir，千万不要修改！
    cleanup="rm -rf $temp_dir"
    trap "$cleanup" ABRT EXIT HUP INT QUIT
    ```
9. **脚本重定向输出**

    exec 通常会使用其参数所指定的命令来替换正在运行的 shell

    ```bash
    # 保存旧的STDERR
    exec 3>&2

    # 将所有发往STDERR的输出重定向到错误日志文件
    exec 2> /path/to/error_log

    # 通过还原STDERR并关闭文件句柄3来停止重定向
    exec 2>&3-
    ```

10. **Argument list too long**

    使用 xargs 命令（可能还要配合 find）来拆分参数列表。
    
    find -print0 配合 xargs -0。可以使 find 使用空字符（文件名中不会出现这种字符）代替空白字符作为输出记录分隔符，xargs 使用空字符作为输入记录分隔符。这样能正确地解析包含怪异字符的文件名。

    ARG_MAX 限制了 exec* 系列系统调用的总空间需求，内核以此知道它必须分配的最大缓冲区。execve 的 3 个参数是：程序名称、参数向量和环境。

    ```bash
    ## 查看系统限制值
    getconf ARG_MAX
    getconf LINE_MAX
    ```

11. **自定义提示符**

    提示字符串分别为 $PS0、$PS1、$PS2、$PS3、$PS4

    ```bash
    # 实例文件：prompts

    # 用户名@短格式主机名、日期和时间，以及当前工作目录（CWD）：
    export PS1='[\u@\h \d \A] \w \$ '

    # 用户名@长格式主机名、ISO 8601格式的日期和时间、当前工作目录的基本名称（\W）:
    export PS1='[\u@\H \D{%Y-%m-%d %H:%M:%S%z}] \W \$ '

    # 用户名@短格式主机名、bash版本号，以及当前工作目录（\w）：
    export PS1='[\u@\h \V \w] \$ '

    # 换行符、用户名@短格式主机名、基本PTY、shell层级、历史编号、换行符，以及完整的工作
    目录名称（$PWD）：
    export PS1='\n[\u@\h \l:$SHLVL:\!]\n$PWD\$ '

    # 用户名@短格式主机名、上一个命令的退出状态，以及当前工作目录：
    export PS1='[\u@\h $? \w \$ '

    # 换行符、用户名@短格式主机名，以及后台作业数：
    export PS1='\n[\u@\h jobs:\j]\n$PWD\$ '

    # 换行符、用户名@短格式主机名、终端、shell层级、历史编号、作业数量、bash版本号，以
    及完整的工作目录：
    export PS1='\n[\u@\h t:\l l:$SHLVL h:\! j:\j v:\V]\n$PWD\$ '

    # 换行符、用户名@短格式主机名、代表终端的T、代表shell层级的L、代表命令编号的C，以及
    ISO 8601格式的日期和时间：
    export PS1='\n[\u@\h:T\l:L$SHLVL:C\!:\D{%Y-%m-%d_%H:%M:%S_%Z}]\n$PWD\$ '

    # 用户名@短格式主机名、浅蓝色的当前工作目录：
    export PS1='\[\033[1;34m\][\u@\h:\w]\$\[\033[0m\] '

    # 在xterm的标题栏和提示符中同时显示用户名@短格式主机名、当前工作目录：
    export PS1='\[\033]0;\u@\h:\w\007\][\u@\h:\w]\$ '

    # 同时更新颜色和xterm：
    export PS1='\[\033]0;\u@\h:\w\007\]\[\033[1;34m\][\u@\h:\w]\$\[\033[0m\] '
    ```

12. **设置$CDPATH**

    CDPATH 是一个以冒号为分隔符的目录列表，用作内建命令 cd 的搜索路径​。可以将它看作 cd 的 $PATH。
    ```bash
    CDPATH='.:/etc:/usr'
    ```

13. **在会话间同步shell历史记录**

```bash
# 将当前历史记录追加到历史文件
history -a 

# 将历史记录读入当前shell
history -n

HISTTIMEFORMAT='%Y-%m-%d_%H:%M:%S; '

HISTTIMEFORMAT=': %Y-%m-%d_%H:%M:%S; '
```

14. **批量解压ZIP文件**

```bash
unzip '*.zip'

for x in /path/to/date*/name/*.zip; do unzip "$x"; done

for x in $(ls /path/to/date*/name/*.zip 2>/dev/null); do unzip $x; done
```

15. **获取文件元数据**

- (-path /proc -o -path...) -prune 用于去除各种无关的目录
- printf 格式以 d 作为前缀，然后是八进制模式的文件权限、用户名、用户组等。
- -type l 表示查找符号链接并显示每个链接的指向。

```bash
#!/usr/bin/env bash
# 实例文件：archive_meta-data

printf "%b" "Mode\tUser\tGroup\tBytes\tModified\tFileSpec\n" > archive_file
find / \( -path /proc -o -path /mnt -o -path /tmp -o -path /var/tmp \
  -o -path /var/cache -o -path /var/spool \) -prune \
  -o -type d -printf 'd%m\t%u\t%g\t%s\t%t\t%p/\n' \
  -o -type l -printf 'l%m\t%u\t%g\t%s\t%t\t%p -> %l\n' \
  -o -printf '%m\t%u\t%g\t%s\t%t\t%p\n' >> archive_file
```

16. **查找只出现在一个文件中的内容**

命令 grep -vf right left 中，-f 选项将参数 right 视为模式文件，从中提取模式（一行一个模式）

uniq -u 能够显示出文件中独有的行

uniq -d 能够显示出同时在两个文件中出现的行

```bash
$ cat left
record_01
record_02.left only
record_03
record_05.differ
record_06
record_07
record_08
record_09
record_10

$ cat right
record_01
record_02
record_04
record_05
record_06.differ
record_07
record_08
record_09.right only
record_10

# 仅显示出现在文件left中的行
$ comm -23 left right
record_02.left only
record_03
record_05.differ
record_06
record_09

# 仅显示出现在文件right中的行
$ comm -13 left right
record_02
record_04
record_05
record_06.differ
record_09.right only

# 仅显示同时出现在两个文件中的行
$ comm -12 left right
record_01
record_07
record_08
record_10
```

17. **在目录间快速移动**

- pushd
- popd
- dirs

如果对一个新目录使用 pushd，那么它会将前一个目录压入栈中

如果使用 pushd 时没有指定目录，那么它会交换栈顶的两个目录的位置

当使用 popd 时，它会弹出栈顶保存的当前位置，切换到新的栈顶目录

如果不记得目录栈中都有哪些目录，可以使用内建命令 dirs 按照从左到右的顺序显示。加上 -v 选项后，显示形式更形象。

pushd +2 会将编号为 2 的目录置为栈顶（并切换到该目录）并将其他目录下压。

要想看到类似于栈的目录列表，但又不希望出现编号，可以使用 -p 选项

18. **历史命令重用**

```bash
## 重复上一次命令
!!
## 重复上一次命令的最后一个参数
!$
## 重复上一次命令的第1个参数
!:1
## 重复上一次命令的第2个参数，以此类推
!:2

## 使用脱字符（^）替换机制
$ /usr/bin/somewhere/someprog -g -A -yknot -w /tmp/soforthandsoon
...
$ ^-g -A^-gB^
/usr/bin/somewhere/someprog -gB -yknot -w /tmp/soforthandsoon
...

## 进行命令行内容替换时要以 ^ 起始，当你想在命令行最后添加更多文本时，结尾的（第 3 个）^才有必要。
$ /usr/bin/somewhere/someprog -g -A -yknot
...
$ ^-g -A^-gB^ /tmp^
/usr/bin/somewhere/someprog -gB -yknot /tmp
...

## 要想删除部分内容并将其替换为空值，可以像下面这样做。
$ /usr/bin/somewhere/someprog -g -A -yknot /tmp
...
$ ^-g -A^^
/usr/bin/somewhere/someprog -yknot /tmp
...
$ ^knot^
/usr/bin/somewhere/someprog -gA -y /tmp
...
$

## 用:替换
## 如果想修改命令行中出现的所有实例，则需要在 s 之前加上 g（以表示全局替换）
$ /usr/bin/somewhere/someprog -g -H -yknot -w /tmp/soforthandsoon
Error: -H not recognized. Did you mean -A?
$ !!:s/H/A/
/usr/bin/somewhere/someprog -g -A -yknot -w /tmp/soforthandsoon
...
$ !!:gs/s/S/
/usr/bin/Somewhere/Someprog -g -S -yknotS -w /tmp/SoforthandSoon
...
$

## 使用历史命令时，可以加入 :p 修饰符，这会使得 bash 只输出命令
$ rm !$:p
rm ?b1.txt
$
```

fc 命令可以将最近执行的命令（或者一批命令）保存在临时文件中，调用编辑器，并允许你按照适合的方式修改其中的命令，然后在退出编辑器时自动重新执行编辑过的命令。

如果调用 fc 时不带参数，则只使用最后一行。你也可以用参数指定某一行：fc 1004 会使用命令历史记录中编号为1004 的命令行，而 fc -5 会使用第 5 个最近的命令。也可以指定范围参数，例如，fc 1001 1005 允许编辑编号为 1001 到 1005 的命令行，fc -5 -1 允许编辑最近执行过的 5 个命令。

当你退出 fc 命令所调用的编辑器时，会重新执行临时文件中剩余的所有命令，即使未做任何改动，这时可以删除文件中的所有命令行，保存并退出。