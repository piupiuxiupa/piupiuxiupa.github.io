---
title: Shell 实用技巧（二）
date: 2024-08-07 20:58:11
tags: linux
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

