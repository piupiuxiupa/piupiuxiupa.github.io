---
title: shell 实用技巧（二）
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