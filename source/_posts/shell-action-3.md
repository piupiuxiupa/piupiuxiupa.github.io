---
title: Shell 实用技巧（三）
date: 2024-08-15 10:53:19
tags: linux
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