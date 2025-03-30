---
title: Shell 实用技巧（二）
date: 2024-08-07 20:58:11
tags: linux
categories: Notes
---

## Shell 实用技巧
1. **创建简单的菜单**

```bash
#!/usr/bin/env bash
# 实例文件：dbinit.1
#
## 修改提示符
## PS1 命令行提示符
## PS2 续行提示符
## PS3 select提示符
DBLIST=$(sh ./listdb | tail -n +2)

PS3="0 inits >"

select DB in $DBLIST
do
    if [ $DB ]
    then
        echo Initializing database: $DB

        PS3="$((++i)) inits> "

        mysql -u user -p $DB <myinit.sql
    fi
done
```

2. **简单的RPN计算器**
> 6RPN（Reverse Polish notation，逆波兰表示法或逆波兰记法）是一种由波兰数学家扬 • 武卡谢维奇于1920 年引入的数学表达式方式。在逆波兰记法中，所有运算符都置于操作数的后面，因此也被称为后缀表示法。逆波兰记法不需要括号来标识运算符的优先级。

```bash
#!/usr/bin/env bash
# 实例文件：rpncalc
#
# 简单的RPN命令行（整数）计算器
#
# 获取用户提供的参数并计算
# 参数形式为：a b运算符
# 允许使用*x*代替*
#
# 检查参数数量：
if [ \( $# -lt 3 \) -o \( $(($# % 2)) -eq 0 \) ]
then
    echo "usage: calc number number op [ number op ] ..."
    echo "use x or '*' for multiplication"
    exit 1
fi

ANS=$(($1 ${3//x/*} $2))
shift 3
while [ $# -gt 0 ]
do
    # 在 $(( )) 中，除了位置参数（如 $1、$2）​，变量名前面并不需要加 $
    ANS=$((ANS ${2//x/*} $1))
    shift 2
done
echo $ANS
```

3. **浮点数计算**

```bash
# 实例文件：func_calc

# 简单的命令行计算器
function calc {
    # 仅适用于整数！--> echo The answer is: $(( $* ))
    # 浮点数
    awk "BEGIN {print \"The answer is: \" $* }";
} # 函数calc定义完毕
```
4. **正则模式**

    - `.`匹配任意单个字符
    - 星号`*`匹配上一个字符的 0 次或多次出现
        -  `.*` 匹配 0 个或多个任意字符,甚至是空行
        - `..*` 匹配任意单个字符以及紧随其后的 0 个或多个任意字符,也就是一个或多个字符，但不能是空行
    - 脱字符 `^` 匹配文本行的行首位置
    - 美元符号 `$` 匹配文本行的行尾位置，因此 `^$` 匹配空行
    - 方括号`[]`中的一组字符匹配其中任意某个字符
    - 如果方括号内的第一个字符是脱字符`[^abcd]`，则匹配的是**不在该字符组中的**任意字符
    - `{}` 区间表达式
        - `\{n,m\}`n 是重复的最小次数，m 是重复的最大次数
        - `\{n\}`，则表示“只重复 n 次”​
        - `\{n,\}`，则表示“至少重复 n 次”​

5. **搜索压缩文件**
    - zcat
    - zgrep

6. **awk**

    - 反转每行单词
        ```bash
        $ awk '{
            for (i=NF; i>=0; i--) {
                printf "%s ", $i;
            }
            printf "\n"
        }' <filename>
        ```

    - 汇总数字列表
        ```bash
        ## next 命令，它会停止处理该输入行，并从下一行输入重新开始
        ls -l | awk '/^total/{next} {sum += $5}; END {print sum}'

        ## tail +2 可实现同样的效果
        ls -l | tail +2 | awk '/^total/{next} {sum += $5}; END {print sum}'
        ```
    - 统计字符串出现次数
        ```awk
        #!/usr/bin/awk -f
        # 实例文件：asar.awk
        # Awk关联数组
        # 用法：ls -lR /usr/local | asar.awk

        NF > 7 {
            ## 变量初始为空
            ## ++ 变为 1
            user[$3]++
        }

        END {
            for (i in user) {
                printf "%s owns %d files\n", i, user[i]
            }
        }
        ```
        ```bash
        $ ls -lR /usr/local | awk -f asar.awk
        bin owns 68 files
        albing owns 1801 files
        root owns 13755 files
        man owns 11491 files
        $
        ```
    - 输出直方图
        ```awk
        #!/usr/bin/awk -f
        # 实例文件：hist.awk
        # 用Awk生成直方图
        # 用法：ls -lR /usr/local | hist.awk

        function max(arr, big)
        {
            big = 0;
            for (i in user)
            {
                if (user[i] > big) { big=user[i];}
            }
            return big
        }

        NF > 7 {
            user[$3]++
        }
        END {
            # 进行缩放
            maxm = max(user);
            for (i in user) {
                #printf "%s owns %d files\n", i, user[i]
                scaled = 60 * user[i] / maxm ;
                printf "%-10.10s [%8d]:", i, user[i]
                for (i=0; i<scaled; i++) {
                    printf "#";
                }
                printf "\n";
            }
        }
        ```
    - 输出匹配词之后的文本
        ```bash
        $ cat para.awk
        /keyphrase/ { flag=1 }
        flag == 1 { print }
        /^$/ { flag=0 }

        $ awk -f para.awk < searchthis.txt
        ```

7. **bash 关联数组**

> 其他语言中也称为散列或字典

```bash
# 实例文件：cnt_owner
# 使用bash统计文件所有者
# 将“ls -l”的输出通过管道传入该脚本

## 定义关联数组
declare -A AACOUNT
## -a 后的变量会被认为是数组，默认以空格为分割符
while read  -a LSL
do
    # 只考虑包含7个及以上字段的行
    if (( ${#LSL[*]} > 7 ))         # 数组大小
    then
        NDX=${LSL[3]}               # 字符串赋值
        (( AACOUNT[${NDX}] += 1 ))  # 算术递增
    fi
done

## ${AACOUNT[@]} 会生成数组所有元素值的列表，如果加上惊叹号变成${!AACOUNT[@]}，则得到的是该数组所有索引的列表。
for VALS in "${!AACOUNT[@]}"        # 各个元素的索引
do
    echo $VALS "owns" ${AACOUNT[$VALS]} "files"
done
```

8. **sort 排序**
```bash
sort -r # reverse
sort -f # 不区分大小写
sort -n # 数值排序
sort -u # 去掉重复输出
```

如果配合 `uniq -c` 使用，`sort -rn` 可以非常方便地给出一个按照降序排列的列表。
> uniq -d  只查看重复行

- IP地址排序

    ```bash
    ## -t 选项用于指定字段之间的分隔符
    $ sort -t . -k 1,1n -k 2,2n -k 3,3n -k 4,4n ipaddr.list
    10.0.0.2
    10.0.0.5
    10.0.0.20
    192.168.0.2
    192.168.0.4
    192.168.0.12
    ```

9. **tr**
```bash
tr ';:.!?' ',' <other.punct >commas.all
# 将大写字母转换为小写字母
tr 'A-Z' 'a-z' <be.fore >af.ter
# 删除所有回车符
tr -d '\r' <file.dos >file.txt
```

10. **wc**
```bash
$ wc data_file
5     15     60 data_file
# 只显示行数
$ wc -l data_file
5 data_file

# 只显示单词数
$ wc -w data_file
15 data_file

# 只显示字符数（通常等同于字节数）
$ wc -c data_file
60 data_file

## 输出数字及文件名
wc -l data_file
## 只输出数字
wc -l < data_file
# 虽然也能只输出数字，但是cat增加了脚本运行开销
cat data_file | wc -l
```

11. **find**
    - `-delete`
    - `-newermt`
    - `-mtime`
    - `-size`
    - `-print0` 往往配合 `xargs -0`
    - `-type`
    - `-exec` 
> 文件大小还包含了单位 k（千字节）​。如果单位是 c，则表示字节（或字符）​。如果使用 b 作为单位或者不写任何单位，则表示块。 \
> 对于碰到的每个文件，先测试其是否为目录，如果是，才测试名称是否符合模式。 \
> 所以 type放到name前面使用能略微提高搜索效率。

12. **locate**

13. **查找命令**
    - `which`
        - 在各版本linux表现不同，且不是内建命令
    - `type -P`
        - shell内建命令，命令不存在时不会输出错误消息，脚本中建议用
    - `command -v/-V`
        - shell内建命令，`v`找不到时不输出错误消息，`V`找不到时输出错误消息

14. **函数参数及返回值**
    - 向函数内部的变量赋值
    - 用`echo`或`printf`将输出发送到标准输出，并用`$()`或者` `` `调用函数
    ```bash
    # 实例文件：func_max.1

    # 定义函数：
    ## 函数内部没有用local声明的变量都是全局变量
    function max ()
    {
        local HIDN
        if [ $1 -gt $2 ]
        then
            BIGR=$1
        else
            BIGR=$2
        fi
        HIDN=5
    }
    
    # 实例文件：func_max.2

    # 定义函数：
    function max ()
    {
        if [ $1 -gt $2 ]
        then
            echo $1
        else
            echo $2
        fi
    }
    # 调用函数
    max 10 20
    # 调用函数并将结果赋值给变量
    BIGR=$(max 10 20)
    echo $BIGR
    # 调用函数并将结果赋值给变量
    BIGR=`max 10 20`
    echo $BIGR
    ```
    > 在函数中引用参数也是使用 $1、$2 等。但是，$0 保持不变，其中包含的还是所调用脚本的名称。
    >
    > $FUNCNAME 本身引用的是该数组的第 0 个元素，其中包含的是当前执行函数的名称。
    >
    > $FUNCNAME 仅存在于函数执行期间。

15. **信号捕获**
    其实`-SIGKILL（-9）`无法捕获。该信号可以立刻“杀死”进程，压根没机会捕获。

    ```bash
    ## 列出所有可捕获的信号
    trap -l

    ## 第一个参数填捕获到对应信号后的行为
    ## 比如捕获退出信号时清理临时文件
    trap "Clean tmp files" EXIT

    ## 如果命令是被信号 signal number 终止的，则脚本的退出状态码为 128+signal number。
    ```
16. **alias**
    - `''` 单引号包括的变量会在执行别名命令时生效变量值
    - `""` 双引号包括的变量，在设置别名时就会使值生效，写死
    ```bash
    ## 删除别名 h
    ## \ 可以使别名失效，防止有人将unalias命令设置成别名，如果有人将其设置为 : 就会使其失效
    \unalias h
    ## 删除所有别名
    unalias -a 
    # 定义别名时不允许使用参数，如 $1, $2
    ```

17. **避开别名和函数**
    - 使用 bash shell 的 builtin 命令来忽略 shell 函数和别名，执行实际的内建命令。
    - 使用 `command` 命令来忽略 shell 函数和别名，并执行实际的外部命令。
    - 如果只是想避开别名扩展，但仍允许执行函数，在命令前加上 `\` 前缀即可。
    > type -a cmd 会显示所有包含cmd命令的地方，如果是函数，还会显示出函数代码

18. **计算已过去的时间**
    - 使用内建命令 `time` 或 bash 变量 `$SECONDS`
    - `$SECONDS` 会记录 shell 启动以来的时长（秒数）​。如果为其赋值，那么随后在扩展该变量时，得到的值就是先前的赋值与赋值操作以后的时长（秒数）之和。

19. **自定义文件描述符及重定向**
    ```bash
    # 将标准输出定向到 out.log文件
    exec 1> out.log
    # 将错误输出重定向标准输出
    exec 2>&1
    # 定义文件描述符 3 输出到other.log文件
    exec 3> other.log
    # 关闭文件描述符 3
    exec 3>&- 

    ## 函数也可重定向输出
    # 根据传进参数会分别进行输出，无需写迭代循环
    ERROUT(){
        printf "Error: %b\n" "$@"
    } >&2
    ## printf 小技巧
    test="a b c"
    printf "Print %b\n" "${test[@]}"
    ## 会打印三条输出

    ```

    重定向的顺序通常很重要
    ```bash
    # 标准输出被重定向到文件 my.file，然后标准错误被重定向到和标准输出相同的地方。所有的输出都会出现在 my.file 中
    somecmd >my.file 2>&1
    # 标准错误被重定向到标准输出（此时标准输出指向的是屏幕）​，然后标准输出被重定向到 my.file。因此，只有标准输出消息会出现在文件中，而错误消息仍旧会出现在屏幕上。
    somecmd 2>&1 >my.file

    somecmd 2>&1 | othercmd
    # bash 4.x 版本中的一种便捷语法可以同时将标准输出和标准错误连入管道。
    somecmd |& othercmd
    ```

20. **处理日期和时间**

    - `date`

    ```bash
    # 将日期转为纪元秒格式
    date -d "20230113" "+%s" 
    # 纪元秒格式转换需要加 @符号
    date -d "@1673000000"

    # 更多形式查阅其他文档
    date '+%Y-%m-%d %H:%M:%S %z'
    date -d 'today' '+%Y-%m-%d %H:%M:%S %z'
    date -d 'tomorrow' '+%Y-%m-%d %H:%M:%S %z'
    date -d 'yesterday' '+%Y-%m-%d %H:%M:%S %z'
    date -d '1 day ago' '+%Y-%m-%d %H:%M:%S %z'
    date -d '-1 days' '+%Y-%m-%d %H:
    %M:%S %z'
    ```

    - `printf`
        - `printf` 有一个二进制版本
        - 这里的`printf`指的是bash内建版本
    ```bash
    # 输出日期
    printf '%(%F %T)T\n' '-1'
    # 可以为变量赋值
    printf -v today '%(%F)T' '-1'
    echo $today
    # 有两个特殊的参数值可以使用：-1代表当前时间，-2 代表 shell 被调用的时间。
    ```
21. **比较文件**

    - 比较纯文本文件
    - 比较二进制文件

    通用方法：
    以上两种文件都可以通过计算sha值来确认是否相同。

    比较纯文本文件比较简单，直接使用 `diff` 命令即可。
    diff 常用选项：
    - `-b` 忽略空白行
    - `-c` 显示差异的上下文
    - `-u` 显示差异的上下文，并且显示差异的行号
    - `-y` 显示差异的上下行
    - `-B` 忽略大小写
    - `-i` 忽略大小写
    - `-q` 不显示差异的上下文
    - `-r` 递归比较目录
    
    diff 比较二进制文件不会显示太多信息，只会告诉两个文件不一样，需要使用 `cmp` 命令。
    
    或者通过解压word文档，用解压出来的xml文件进行对比。

    ```bash
    # 此行 sed可以将每个<>替换为自身加上换行符，从而将包含内容的行独立出来
    sed -e 's/>/>\
    /g; s/</\
    </g' document.xml > word.xml
    ```
    > word文档实际上是由里面的图片和xml文件压缩所得，可以尝试用zip解压出来看看

