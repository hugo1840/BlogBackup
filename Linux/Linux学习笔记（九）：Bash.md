@[TOC](Linux学习笔记（九）：Bash)
# Bash是什么
Shell是用户与Linux内核交互的接口，它既是一种命令语言，也是一种程序设计语言。相比图形界面，使用shell的传输速度更快，不容易出现断线或者信息外流的问题。Shell有许多不同的版本，比如Bourne Shell (sh)、C Shell、TCSH、K Shell。Linux目前使用的是**Bourne Again Shell**，简称为**bash**，是Bourne Shell的增强版本。文件 **/etc/shells** 中描述了系统中支持的shell版本，**/etc/passwd** 中则记录了每个用户登录后使用的预设shell。

下面介绍一些在终端中使用shell的技巧：
* 使用bash时，按下**tab**键可以进行命令或文本补全；
* 输入 `type [-t] command` 查看命令command是否为外部指令 (file)、命令别名 (alias) 或者bash内建指令 (builtin)；
* 当输入的指令很长时，输入**反斜杠** \ 可以进行指令换行；
* 按下**Ctrl u**可以删除光标前的所有指令串，**Ctrl k**则可以向后删除所有指令串；
* 按下**Ctrl a**可以让光标移动到指令串最前面，**Ctrl e**则可以移动到最后面。

# Shell变量
shell变量可以分为环境变量和用户自定义变量。环境变量一般都大写表示，比如PATH、HOME、MAIL、SHELL等。

## 定义与使用
变量使用等号定义，且必须满足以下规则：
* 等号两边不能直接接空格；
* 变量名不能以数字开头，且只能包括英文字母与数字；
* 变量内容中如果有空格可以使用单引号或双引号将其包含起来。==使用**单引号**时，变量内容中以 **\$** 开头的特殊字符会被转化为一般文本==，失去其特殊意义；
* 变量内容中使用**斜杠 \\** 可以将特殊字符（空格、换行符、\$、\、引号等）变成一般文本字符；
* 在长指令串中可以包含子指令串，但必须用 **反单引号\`\`** 或者 **\$()** 包围起来；
* 使用变量内容的格式为 **\$变量名** 或者 **\${变量名}**，还可以使用 **"\$变量名"** 或者 **\${变量名}** 来扩增变量内容；
* 使用**export**命令可以将一般变量变为环境变量，以供子程序使用；
* 使用**unset**命令可以取消变量定义。

```bash
firstname=Lebron
playername=LebronJames
playername=Lebron\ James
playername="Lebron James"
playername='Lebron James'
playername="${firstname} James"
echo ${playername}   # 显示Lebron James
playername='${firstname} James'
echo $playername # 显示${firstname} James

version=`uname -r`
echo ${version}   # 显示Linux内核版本
version=$(uname -r)
echo $version
unset version   # 取消变量
echo $version   # 输出为空

echo ${PATH}
PATH="$PATH":/home/bin
PATH=${PATH}:/home/bin
echo $PATH
export PATH   
```

## 环境变量
**env**
使用env指令可以查看shell环境下的所有环境变量。其中一些重要的环境变量包括：
* HOME：用户的家目录；
* SHELL：当前环境使用的shell版本；
* HISTSIZE：系统记录的历史指令的条数；
* PATH：执行文件的搜寻路径；
* LANG：语系信息，与文件编码和字符排序有关；
* RANDOM：随机数生成器。

**set**
set指令可以显示bash环境内的所有变量，包括环境变量。其中，有如下几个特殊变量：
* PS1：命令提示字符设定，即在终端中输入指令前的那一串字符，形如 **[用户名@主机名 工作目录]\$**，比如`[root@xxx home]# `；
* \$：当前shell的进程号PID（`echo $$` 查看）；
* ?：上一个指令执行后的返回值，一般等于0，表示成功执行（`echo $?` 查看）。

**export**
在已经打开的shell终端中执行bash指令可以打开一个bash子程序，输入exit可以退出子程序。==bash子程序只会继承bash父程序中的环境变量==。如果想在子程序中使用父程序中自定义的变量，需要先使用export指令将自定义变量变成环境变量。

## 其他指令
**read**
使用read指令可以从键盘读入数据并存入变量（不需要事先定义变量）。该指令常用于shell脚本。指令格式为 `read -p "提示语" -t 秒数 变量名`，其中秒数为等待执行者输入的时间。

**declare**
使用declare指令可以设置变量类型。

```bash
sum=50+100+150
echo $sum   # 输出50+100+150
declare -i sum=50+100+150   # -i表示整数
echo $sum   # 输出300

declare -x sum   # -x表示设为环境变量
declare -r sum   # -r表示设为只读变量
declare +x sum   # +x表示取消环境变量（加号表示取消）
declare -a arr   # -a表示数组
declare -p arr   # -p表示列出变量的类型
```

**array**
定义数组时直接给元素赋值即可。

```bash
var[1]=Bronny
var[2]=Maximus
var[3]=Julie
echo "${var[1]}, ${var[2]}, ${var[3]}"
```

**ulimit**
ulimit指令用于限制当前shell环境中用户能够使用的系统资源，包括可以打开的文件数目、可以使用的CPU时间以及内存大小。

## 变量内容修改（高级）
**变量内容的删除与替换**
| 变量设定方式 | 说明 |
| -- | -- |
| ${变量#关键词} | 若变量内容从头开始的数据与关键词匹配，删除匹配的最短数据 |
| ${变量##关键词} | 若变量内容从头开始的数据与关键词匹配，删除匹配的最长数据 |
| ${变量%关键词} | 若变量内容从尾向前的数据与关键词匹配，删除匹配的最短数据 |
| ${变量%%关键词} | 若变量内容从尾向前的数据与关键词匹配，删除匹配的最长数据 |
| ${变量/旧字符串/新字符串} | 若变量内容与旧字符串匹配，将第一个旧字符串替换为新字符串 |
| ${变量//旧字符串/新字符串} | 若变量内容与旧字符串匹配，将所有的旧字符串替换为新字符串 |

**变量的测试与内容修改**
| 变量设定方式 | str没有设定 | str为空字符 | str设定为非空字符 |
| -- | -- | -- | -- |
| var=${str-expr} | var=expr | var= | var=$str |
| var=${str:-expr} | var=expr | var=expr | var=$str |
| var=${str+expr} | var= | var=expr | var=expr |
| var=${str:+expr} | var= | var= | var=expr |
| var=${str=expr} | str=expr, var=expr | str不变, var= | str不变, var=$str |
| var=${str:=expr} | str=expr, var=expr | str=expr, var=expr | str不变, var=$str |
| var=${str?expr} | expr输出至stderr | var= | var=$str |
| var=${str:?expr} | expr输出至stderr | expr输出至stderr | var=$str |

# 指令别名
**alias, unalias**
alias指令可以设定指令的别名，可以用来简化经常使用但是比较长的指令。

```bash
[fang@myhost ~]$ alias   # 查看目前已经存在的别名
alias ls='ls --color=auto'
alias rm='rm -i'
alias vi='vim'
...
[fang@myhost ~]$ alias ll='ls -l --color=auto'   # 定义别名
[fang@myhost ~]$ ll /home/fang   # 使用别名
[fang@myhost ~]$ unalias ll   # 取消别名
```

**history**
使用history指令可以查看执行过的历史指令，指令后接数字n可以查看最近执行过的n条指令。使用 `! 数字` 和 `!!` 可以执行历史指令。

```bash
[fang@myhost ~]$ history 3
67 ls   # 指令前的数字表示该指令在当前shell中是第几条历史指令
68 history
69 history 3
[fang@myhost ~]$ !68   # 执行第68条指令：history
[fang@myhost ~]$ !!   # 执行上一条指令，相当于按向上箭头再按Enter
```

# Bash操作环境
**指令搜寻顺序**
在终端中执行一个指令时，系统按如下顺序搜寻并执行指令：
1. 以绝对或相对路径执行指令，比如`/bin/ls`或`./ls`；
2. 由alias别名找到该指令；
3. 由bash内建的指令执行；
4. 通过环境变量$PATH中的路径顺序搜索到第一个指令来执行。

**环境配置文件**
Linux系统中的shell可以分为login shell和non-login shell，它们使用的环境配置文件不相同。<u>**login shell**需要输入账户名和密码登录才能启动，比如由终端tty1~tty6登录后启动的shell；**non-login shell**则不需要账户名和密码登录就能使用，比如使用X Windows图形化界面打开的终端、以及在已经启动的终端中执行bash命令打开的子程序</u>。

login shell一般会读取以下两个配置文件：
* /etc/profile
* \~/.bash_profile 或 \~/.bash_login 或 \~/.profile

non-login shell仅会读取 ~/.bashrc 文件。

**终端环境设定**
在Linux开机登录界面，按下**Ctrl Alt F1\~F6**可以分别打开tty1\~tty6对应的纯命令行交互界面。使用stty指令可以展示终端中某些特殊按键代表的意义。set指令可以用于设定指令输入/输出环境。

**通配符**
bash环境中的通配符（wildcard）与正则表达式有一些区别。
| 符号 | 说明 |
| -- | -- |
| * | 代表0到无穷多个任意字符 |
| ? | 代表有且仅有一个任意字符 |
| [ ] | 代表一定有一个在括号内的字符，比如[ab5] |
| [-] | 代表一个连续的范围，比如[0-9], [a-z] |
| [^] | 代表一定有一个字符，且不在括号中，比如[\^abc] |

# 数据流重导向
**标准输出、标准错误输出、标准输入**
标准输出（**STDOUT**）是指指令执行成功后返回的信息。标准错误输出（**STDERR**）是指指令执行失败后返回的错误信息。标准输入（**STDIN**）一般是指由键盘（或者文件）输入的数据。标准输出和标准错误输出默认都会显示到屏幕上。数据流重导向可以将它们传送到其他文件或者设备，其规则为：
* STDIN：代码为0，重导向符号使用 < 或 <<；
* STDOUT：代码为1，重导向符号使用 > 或 >>；
* STDERR：代码为2，重导向符号使用 2> 或 2>>；
* 可以使用 2>&1 或 &> 将STDOUT和STDERR同时重定向到同一个文件。

如果重导向符号后的文件名不存在，则会被自动创建。==符号 > 与 >> 的区别在于前者会**覆盖**文件中已有的内容，而后者会将输出**累加**到文件中已有的数据之后==。

**/dev/null 黑洞**
如果想忽略输出的内容，可以将输出重定向到/dev/null。==重定向到 /dev/null 的所有数据都会被丢弃==。
```bash
[fang@myhost ~]$ find /home -name .bashrc > filename1 2> filename2
[fang@myhost ~]$ find /home -name .bashrc 2> /dev/null

[fang@myhost ~]$ find /home -name .bashrc > filename 2> filename   # 错误
[fang@myhost ~]$ find /home -name .bashrc > filename 2>&1   # 正确
[fang@myhost ~]$ find /home -name .bashrc &> filename   # 正确
```

**标准输入：< 与 <<**
使用 < 符号可以将标准输入由键盘改为文件，而 << 符号可用于指定键盘输入的结束字符。

```bash
[fang@myhost ~]$ cat > myfile
testing myfile ...   # 由键盘输入
# 按Ctrl d 结束键盘输入
[fang@myhost ~]$ cat myfile
testing myfile ...

[fang@myhost ~]$ cat > myfile < ~/.bashrc   # 由文件输入
[fang@myhost ~]$ cat > myfile << "eof"
> new testfile ...
> eof   # 不用输入Ctrl d
[fang@myhost ~]$ cat myfile
new testfile ...
```

**命令执行的判断依据 ; && ||**
如果需要连续执行多条指令时，可以使用分号隔开指令。如果不同指令之间存在相关性，则可以使用&&或||进行判断。如果有多个判断符号，从左至右依次判断。
| 指令| 说明 |
|--|--|
| cmd1 && cmd2 | 只有当cmd1成功执行时才会执行cmd2 |
| cmd1 \|\| cmd2 | 只有当cmd1执行出错时才会执行cmd2 |
| cmd1 && cmd2 \|\| cmd3 | 如果cmd1执行成功，则仅执行cmd2；<br>如果cmd1执行失败，则仅执行cmd3 |
| cmd1 \|\| cmd2 && cmd3 | 如果cmd1执行成功，则仅执行cmd3；<br>如果cmd1执行失败，则先执行cmd2，cmd2执行成功才会执行cmd3 |

# 管线命令
管线（pipeline）命令 | 能够将左边指令的标准输出**STDOUT**转变为标准输入**STDIN**并传递给右边的指令。下面介绍一些支持管线命令的常用指令。

**cut, grep**
cut指令能够以行为单位切分数据并取出其中特定位置的内容，指令格式为`cut -d '分隔符' -f 位置` 或 `cut -c 字符区间`。grep指令则能够以行为单位搜寻给定的字符串，指令格式为`grep [-acinv] [--color=auto] '字符串' 文件名`。

**sort, wc, uniq**
sort用于排序，排序结果与语系编码有关，指令格式为`sort [-fbMnrtuk] 文件名或STDIN`。uniq用于取消显示排序结果中的重复项，指令格式为`uniq [-ic]`。wc指令用于显示文件包含的行数或字符数，指令格式为`wc [-lwm]`。

**tee**
tee指令用于双向重导向，可以同时将数据流导向给文件和STDOUT（即屏幕）。指令格式为`tee [-a] 文件名`。

**tr, col, join, paste, expand**
tr指令用于删除或替换数据流中的文字，指令格式为`tr [-ds] '字符串'`。col指令一般用于将tab键转变为对等的空格键，指令格式为`col -x`。join指令用于合并两个文件中有相同内容的行，其语法为`join [-ti12] 文件1 文件2`。==使用join指令前必须对文件进行排序==。paste指令用于直接将两个文件的行合并（默认以tab键分隔），而不会比较文件的内容，其语法为`paste [-d] 文件1 文件2`。expand指令也可以将tab键转变成空格，指令格式为`expand [-t] 文件名`。

**split**
split指令可以将一个大文件分区为多个小文件，以便于复制。其指令格式为`split [-bl] 大文件名 分区文件名前导符`。

**xargs**
xargs指令可以读入STDIN数据流，将其转化为指令参数，从而使得某些原本不支持管线命令的指令可以使用STDIN（比如ls指令）。其语法为`xargs [-0epn] command`。

**减号-**
在管线命令中，经常会将前一个指令的STDOUT作为后一个指令的STDIN，某些指令需要用到文件名来进行处理时（比如tar），该STDIN与STDOUT可以用减号-替代。

```bash
[fang@myhost ~]$ last   # 显示登录账户信息
fang	pts/1	192.168.101.25	Sat	Feb 7 12:35 still logged in
root	pts/1	192.168.101.23	Sat	Feb 7 12:13 - 18:46 (06:33)
...
[fang@myhost ~]$ last | cut -d ' ' -f 1
fang
root
... 

[fang@myhost ~]$ last | grep 'root'   # 取出包含root的行
[fang@myhost ~]$ last | grep -v 'root'   # 取出不包含root的行
[fang@myhost ~]$ last | grep 'root' | cut -d ' ' -f 1
[fang@myhost ~]$ grep --color=auto 'MANPATH' /etc/man_db.conf

[fang@myhost ~]$ cat /etc/passwd | sort   # 默认以tab为分隔符，第一个数据为排序依据
[fang@myhost ~]$ cat /etc/passwd | sort -t ':' -k 3   # 以:作为分隔符，第3个数据为排序依据
[fang@myhost ~]$ last | cut -d ' ' -f 1 | sort

[fang@myhost ~]$ [fang@myhost ~]$ last | cut -d ' ' -f 1 | sort | uniq   # 排序去重
[fang@myhost ~]$ last | cut -d ' ' -f 1 | sort | uniq -c   # 对重复结果计数

[fang@myhost ~]$ cat /etc/man_db.conf | wc   # 文件计数
131	723	5171   # 依次为行数、字数（英文单字）、字符数
[fang@myhost ~]$ last | grep [a-zA-Z] | grep -v 'wtmp' | grep -v 'reboot' | \
> grep -v 'unknown' | wc -l   # 统计去除wtmp、reboot、unknown和空白行之后的用户登录人次

[fang@myhost ~]$ last | tee accfile | cut -d ' ' -f1   # 将数据流保存到文件的同时也显示到屏幕
[fang@myhost ~]$ ls -l /home | tee ~/homefile | more

[fang@myhost ~]$ last | tr '[a-z]' '[A-Z]'   # 将last输出中的小写字母替换为大写字母之后再显示
[fang@myhost ~]$ cat /etc/passwd | tr -d ':'   # 在文件输出的数据中去掉冒号再显示
[fang@myhost ~]$ cat ~/passwd | tr -d '\r' > ~/passwd.linux   # 删除文件中的DOS断行符CRLF后转存为新文件

[fang@myhost ~]$ cat -A /etc/man_db.conf   # 显示文件中所有特殊字符
[fang@myhost ~]$ cat /etc/man_db.conf | col -x | cat -A | more   # 将tab键^I变成空格键再输出

# 使用join指令前先用sort对文件进行排序
[fang@myhost ~]$ join -t ':' /etc/passwd /etc/shadow | head -n 3   
# 以冒号为分隔符（默认是空格），合并两个文件中第一个字段相同的行，并显示前三行
[fang@myhost ~]$ join -t ':' -1 4 /etc/passwd -2 3 /etc/group | head -n 3
# 以冒号为分隔符，合并第一个文件中第4个字段与第二个文件中第3个字段相同的行，把相同的字段放在每行输出内容的最前面，并显示前三行

[fang@myhost ~]$ paste /etc/passwd /etc/shadow   # 以tab键为分隔符合并两个文件的行再显示

# 将/etc/services保存为多个300k大小的文件，文件名前缀为services
[fang@myhost ~]$ cd /tmp; split -b 300k /etc/services services   
[fang@myhost tmp]$ ll -k services*
-rw-rw-r--	1	fang	fang	307200	Jul 9 22:52	servicesaa
-rw-rw-r--	1	fang	fang	307200	Jul 9 22:52	servicesab
-rw-rw-r--	1	fang	fang	 55893	Jul 9 22:52	servicesac
[fang@myhost tmp]$ cat services* >> servicesback   # 合并多个文件

[fang@myhost ~]$ id root 
uid=0(root)	gid=0(root)	groups=0(root)
# 取出/etc/passwd文件中前三行的第一栏（账户名），使用id依次显示每个账号uid等信息
[fang@myhost ~]$ cut -d ':' -f 1 /etc/passwd | head -n 3 | xargs -n 1 id
# 以下是几种错误做法
[fang@myhost ~]$ id $(cut -d ':' -f 1 /etc/passwd | head -n 3)   # id只能接受一个参数
[fang@myhost ~]$ cut -d ':' -f 1 /etc/passwd | head -n 3 | id   # id不是管线命令
[fang@myhost ~]$ cut -d ':' -f 1 /etc/passwd | head -n 3 | xargs id   # id只能接受一个参数

# 依次显示/etc/passwd中每个账户的uid信息，直到遇到sync字符串就停止
[fang@myhost ~]$ cut -d ':' -f 1 /etc/passwd | xargs -e'sync' -n 1 id
# 找到/usr/sbin下具有特殊权限的文件，用ls指令列出详细信息
[fang@myhost ~]$ find /usr/sbin -perm /7000 | xargs ls -l

[fang@myhost ~]$ mkdir /tmp/homeback
# 将/home打包后传给STDOUT，经过管线后变成后面指令的STDIN
[fang@myhost ~]$ tar -cvf - /home | tar -xvf - -C /tmp/homeback
```

