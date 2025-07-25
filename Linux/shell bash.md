# 指南

查询命令是否为Bash shell 的内置命令：type

```java
type [-tpa] name
    ：不加参数时，type会直接显示出name是外部命令还是bash内置命令
-t  ：当加入-t参数时，type将name以下面这些字眼显示出它的意义：
      file   ： 表示为外部命令
     alias ： 表示该命令为命令别名所设置的名称
     builtin： 表示该命令为bash的内置命令
-p ：如果后面接的name为外部命令时，才会显示完整的文件名
-a ：会由 PATH 变量定义的路径中，将所有含 name 的命令都列出来，包含alias
```

## 输出

### echo

```bash
echo [-e] 
# -e, 转义，\n变换行
```



## 变量

#### 查看

使用`echo $变量名`或是`echo ${变量名}`就可以查看变量的定义

#### 设置与修改

直接用等号（=）连接变量与它的内容就好

- 双引号内的特殊字符如“$”等，可以保有原本的特性，例如：
  - var=”lang is $LANG”，则echo $var就会得到 lang is zh_CN.UTF-8
- 单引号内的特殊字符则仅为一般字符（纯文本），不过
  - 可以用转义符【\】将特殊符号（如【Enter】、$、\、空格、’ 等）变成一般字符
- 若该变量需要在其他子程序执行，则需要以export来使变量变成环境变量

#### 取消

若要取消变量的设置，可以使用unset

### 变量键盘读取、数组与声明（read、array、declare）

#### read

要读取来自键盘输入的变量，就是用read这个命令。这个命令最常被用在shell脚本的编写当中，想要跟用户交互？用这个命令就对了，关于脚本的写法，我们以后再说，先看read的相关语法吧！

```bash
read  [-pt] variable
-p：后面可以接提示字符
-t：后面可以接等待的【秒数】，这个比较有趣，不会一直等待使用者
```

#### declare，typeset

declare或typeset是一样的功能，就是声明变量的类型。如果使用declare后面没有接任何的参数，那么bash就会主动的将所有的变量名称与内容通通显示出来，就好像使用set一样，那么declare还有什么语法呢？

```bash
declare [-aixr] variable
-a：将后面名为variable的变量定义成为数组（array）类型
-i：将后面名为variable的变量定义成为整数（integer）类型
-x：用法与export一样，就是将后面的variable编程环境变量
-r：将变量设置成为readonly类型，该变量不可被改变内容，也不能被unset
```

由于在默认的情况下，bash对于变量有几个基本的定义：

- 变量类型默认为字符串，所以若不指定变量类型，则1+2为一个字符串而不是计算式
- bash环境里的数值运算，默认最多仅能达整数形态，所以1/3结果是0

### 数据流重定向

**数据流重定向**可以将standard output（简称 stdout）与standard error output（简称 stderr）分别传送到其它的文件或设备，而分别传送所需要的特殊字符如下：

- 标准输入（stdin）：代码为0，使用<或<<
- 标准输出（stdout）：代码为1，使用>或>>
- 标准错误输出（stderr）：代码为2，使用2>或2>>

**如果你想要累加而不是删除原本内容，用 >> 即可。**

/dev/null 垃圾桶黑洞设备与特殊写法，可以吃掉任何导向这个设备的信息

输出到同一个文件

```bash
命令 > test 2>&1
命令 &> test
```

standard input：<与<<

这个< 是什么呢？简单来说就是，**将原本需要由键盘输入的内容，改由文件内容来替换**

 << ，它的意思是【结束的输入字符】，即设置一个敏感字符，一旦输入它并回车，本次输入就结束

### 命令执行的判断根据：;、&&、||

#### cmd;cmd（不考虑命令相关性的连续命令执行）

示例，关机前同步数据：

```java
sync;sync;shutdown -h now
```

这些命令具有相关性，前一个命令是否成功执行与后一个命令是否要执行有关。

#### `$`?(命令返回值)与&&或||

| 命令执行情况   | 说明                                                         |
| -------------- | ------------------------------------------------------------ |
| cmd1 && cmd2   | 1. 若 cmd1 运行完毕且正确运行($?=0)，则开始运行 cmd2。 2. 若 cmd1 运行完毕且为错误 ($?≠0)，则 cmd2 不运行。 |
| cmd1 \|\| cmd2 | 1. 若 cmd1 运行完毕且正确运行($?=0)，则 cmd2 不运行。 2. 若 cmd1 运行完毕且为错误 ($?≠0)，则开始运行 cmd2。 |

### 管道命令（pipe）

**管道命令【|】仅能处理经由前一个命令传来的正确信息，也就是标准输出（STDOUT）信息，对于标准错误信息没有直接处理的能力。**

在每个管道后面接的第一个数据必定是【命令】，而且这个命令必须要能够接受标准输入的数据才行，这样的命令才可为管道命令。也就是说，管道命令要注意两个点：

- 管道命令仅会处理标准输出，对于标准错误会予以忽略。
- 管道命令必须能够接受来自前一个命令的数据成为标准输入继续处理才行。

#### 选取命令（cut、grep）：

cut：

cut主要的用途在于将同一行里面的数据进行分解，最常使用在分析一些数据或文字数据的时候。不过，cut在处理多空格相连的时候，可能会吃力一点，后面我们会介绍awk。

```bash
cut -d‘分隔字符’ -f fields    <==用于有特定分隔字符，相当于拆分成数组，取第fields个元素，可取复数。例：cut -d ';' -f 3,4
cut -c 字符区间              <==用于排列整齐的信息，截取字符。例：cut -c 5-20
-d：后面接分隔字符，与-f一起使用
-f：根据-d的分隔字符将一段信息划分成为数段，用-f取出第几段的意思
-c：以字符（characters）的单位取出固定字符区间
```

grep：

刚刚的cut是将一行的信息中，取出我们想要的，而grep则是一行信息，若有我们想要的，就拿出来。

```bash
grep [-acinv] [--color=auto] '查找字符' filename
-a：将二进制文件以文本文件的方式查找数据
-c：计算找到 ‘查找字符’ 的次数
-i：忽略大小写差异，即大小写不敏感
-n：顺便输出行号
-v：反向选择，亦即显示出没有“查找字符”内容的那一行
--color=auto：可以将找出的关键字部分加上颜色的内容
```

awk

默认的字段的分隔符为“空格键”或“Tab键”。

每一行的字段都有变量名称，如$1、$2等，而$0则代表一整行数据

其实，awk内部处理主要依赖了3个变量

- NF：每一行（`$`0）拥有的字段数
- NR：目前awk所处理的是第几行数据
- FS：目前的分隔字符，默认是空格键

例：

```bash
docker ps -a|awk '$2=="redis" {print "ID\t" $1}'
# 查询$2为redis，打印$1
```

```bash
docker ps -a|awk 'BEGIN {FS=':'} {print "ID\t" $1}'
# 修改分隔符
```

### 文件比对

diff

```bash
diff [-bBi] from-file to-file
# from-file：一个文件名，作为原始比对文件的文件名
# to-file：一个文件名，作为目标比对文件的文件名
# 注意：from-file或to-file可以用-替换（标准输入）
# -b：忽略一行中，仅有多个空白的差异
# -B：忽略空白行的差异
# -i：忽略大小写的不同
```

cmp

相对于diff的广泛用途，cmp似乎就用的没有那么多了，cmp主要也是在比对两个文件，它主要利用字节单位去比对（diff是以行去比对），因此，当然也可以比对二进制文件。

```bash
cmp [-l] file1 file2
-l：将所有不同点的字节处都列出来，因为cmp默认仅会输出第一个发现的不同点
```

### 判断式

#### test

**1. 关於某个档名的『文件类型』判断，如 test -e filename 表示存在否**

| 测试的参数 | 代表意义 |      |      |      |      |
| ---------- | -------- | ---- | ---- | ---- | ---- |
| -e   | 该『档名』是否存在？                    |||||
| -f   | 该『档名』是否存在且为文件(file)？      |||||
| -d   | 该『档名』是否存在且为目录(directory)？ |||||

关於两个整数之间的判定，例如 test n1 -eq n2

| 测试的参数 | 代表意义 |      |      |      |      |
| ---------- | -------- | ---- | ---- | ---- | ---- |
| -eq  | 两数值相等 (equal)                     |
| -ne  | 两数值不等 (not equal)                 |
| -gt  | n1 大於 n2 (greater than)              |
| -lt  | n1 小於 n2 (less than)                 |
| -ge  | n1 大於等於 n2 (greater than or equal) |
| -le  | n1 小於等於 n2 (less than or equal)    |

**判定字串的数据**
| 测试的参数 | 代表意义 |      |      |      |      |
| ---------- | -------- | ---- | ---- | ---- | ---- |
| test -z string    | 判定字串是否为 0 ？**若 string 为空字串，则为 true**         |      |
| test -n string    | 判定字串是否非为 0 ？**<span “>若 string 为空字串，则为 false。 注： -n 亦可省略** |      |
| test str1 = str2  | 判定 str1 是否等於 str2 ，若相等，则回传 true                |      |
| test str1 != str2 | 判定 str1 是否不等於 str2 ，若相等，则回传 false             |      |

例子

```bash
test "${UP}" = "UP" && echo "SUCC" || echo "FAIL"
```

#### 判断符号[ ]

其实就是替换掉test

```basg
[ -z "${HOME}" ]
```

注意：

- 在中括号[]内的每个组件都需要空格来分隔
- 在中括号内的变量，最好都以双引号括号起来
- 在中括号内的常数，最好都以单或双引号括号起来

## Shell脚本：

### if…then

```bash
if [ 条件判断式一 ] ; then
	当条件判断式一成立时，可执行命令
elif [ 条件判断式二 ] ; then
	当条件判断式二成立时，可执行命令 
else
    当条件判断式一与二均不成立时，可执行的命令
fi
```

### case…esac 判断

case可以来处理多个变量的情况

```bash
case $变量名称 in    <==关键字case，还有变量前有美元符
   "第一个变量内容")    <==变量内容建议用双引号，关键字是右括号
         程序段
         ;;            <==每个类别的结尾使用两个分号来处理
   "第二个变量内容")
         程序段
         ;;
   *)                  <==最后一个变量内容使用*代替其他所有值
        不含第一个变量内容与第二个变量内容的其他程序执行段
        exit 1
        ;;
esac                   <==最终的case结尾
```

### 循环（loop）

#### while do done、until do done（不定循环）

不定循环最常见的就是下面的两种状态

```bash
while [ condition ]  <==如果
do            <==do是循环的开始
        程序段落
done        <==done是循环的结束
```

```bash
until [ condition ]  <==直到
do            <==do是循环的开始
        程序段落
done        <==done是循环的结束
```

#### for…do…done（固定循环）

```bash
for (( 初始值; 限制值; 赋值运算 ))
do 
    程序段
done

for c in A B C
do
  echo ${c}
done
```

### shell脚本的跟踪与调试

sh命令可以帮助我们了解脚本文件是否存在语法问题

```java
sh [-nvx] scripts.sh
-n：不要执行脚本，仅检查语法的问题
-v：在执行脚本前，先将脚本文件的内容输出到屏幕上
-x：将使用到的脚本内容显示到屏幕上
```

### tee

```bash
echo "Hello"|tee [-a] h.txt
# -a, 追加
```

> [为初学者介绍的 Linux tee 命令（6 个例子） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/34510815)

# 参考文献

[Linux 基础 - vim与BASH | 弟弟快看-教程 (ddkk.com)](https://www.ddkk.com/zhuanlan/server/linux/4/6.html)

[Linux 基础 - BASH | 弟弟快看-教程 (ddkk.com)](https://www.ddkk.com/zhuanlan/server/linux/4/7.html)