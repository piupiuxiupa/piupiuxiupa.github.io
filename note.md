bash 中一切都是字符串

**在shell脚本中嵌入文档**

```bash
:<<'DOC'
    Write some notes.
DOC
```

**导出变量**

将希望传给其他脚本的变量导出
要想查看所有已导出的变量，敲入命令 env（或者内建命令export -p）

```bash
export MYVAR
export NAME=value
```
env 生成的列表是 set 生成的列表的子集，因为并非所有变量都被导出了。

**处理包含空格的参数**

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
**获取默认值**

如果指定参数（这里是 $1）不存在或为空，则将运算符之后的内容（本例为 /tmp）作为值。否则，使用已经设置好的参数值

```bash
${1:-/tmp}
```

**设置默认值**

首次引用 shell 变量时，如果该变量没有值，则使用赋值运算符为其赋值

赋值运算符有一个重要的例外：不能对位置参数（如 $1或 $*）赋值

```bash
cd ${HOME:=/tmp}
```

> := 执行赋值操作，同时返回运算符右侧的值。:- 只做了前者一半的工作：返回值，但不赋值

**使用空值作为有效的默认值**

如果写成不带冒号的 ${HOME=/tmp}，那么赋值操作仅会在该变量不存在的情况下（从未设置或已明确删除）发生

**不只使用字符串常量作为默认值**

能出现在 shell 变量引用右侧的内容并不局限于字符串常量。

```bash
cd ${BASE:="$(pwd)"}
```

**对不存在的参数输出错误消息**

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

**变量字符串替换**

```bash
# 从字符串name的索引位置start开始，返回长度为number的子串
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

**变量字符串大小写转换**

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

**简单算数**
用 `$(( ))` 或 `let` 进行整数运算
```bash
##  $(( ))可以用 ** 进行幂运算
COUNT=$((COUNT + 5 + MAX * 2))
let COUNT+='5+MAX*2'

let COUNT+=5
```

**测试文件属性**

```bash
# 是否更新（检查文件的修改时间）​。现有文件要比不存在的文件“新”​。
test FILE1 -nt FILE2　　
#是否更旧。同样，不存在的文件要比现有文件“旧”​。
test FILE1 -ot FILE2　　
#具有相同设备和 inode 编号（即便由不同链接所指向，也视为相同的文件）​。
test FILE1 -ef FILE2　　
# 测试文件大小是否不为空
test -s FILE
```
**用正则表达式匹配**

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