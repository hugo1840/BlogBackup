@[TOC](Linux学习笔记（三）：目录管理、文件操作与查阅)
# 目录与路径
## 目录的相关操作
下面的字符代表一些特殊的目录：

```bash
.          # 当前目录
..         # 上一层目录
-          # 前一个工作目录
~          # 当前用户的家目录
~account   # account用户的家目录
```
常见的处理目录的指令有cd、pwd、mkdir、rmdir。

**cd：change directory 变换目录**

```bash
[fang@study ~]$ su -                 # 由fang切换到root用户
[root@study ~]# cd ~fang             # 进入fang的家目录（方括号后面的#表示root用户，不是注释）
[root@study fang]# cd ~              # 进入当前用户root的家目录
[root@study ~]# cd                   # 进入当前用户root的家目录
[root@study ~]# cd ..                # 进入上层目录，即根目录下
[root@study /]# cd -                 # 回到刚才的目录，即root的家目录
[root@study ~]# cd /var/spool/mail   # 进入绝对路径表示的目录
[root@study mail]# cd ../postfix     # 进入相对路径表示的目录
```

**pwd：print working directory 显示当前所在目录**

-P参数用于显示正确完整路径而不是链接文档。
```bash
[root@study ~]# pwd
/root
[root@study ~]# cd /var/mail  # /var/mail/是一个链接文档
[root@study mail]# pwd
/var/mail/
[root@study mail]# pwd -P     # 显示正确完整路径而不是链接文档
/var/spool/mail/
[root@study mail]# ls -ld /var/mail
lrwxrwxrwx.	1	root	root	10	May 4 17:51	/var/mail -> spool/mail
```

**mkdir：make directory 创建新目录**
多层目录需要一层一层的建立，但是 ==-p 参数可以用于直接创建多层级目录。-m参数用来强制给新目录预设权限==。如果没有设置文件权限，系统会采用默认属性（与下文的umask有关）。

```bash
[fang@study ~]$ cd /tmp
[fang@study tmp]$ mkdir test
[fang@study tmp]$ mkdir -p test1/test2/test3/test4
[fang@study tmp]$ mkdir -m 711 test2   # 预设权限rwx--x--x
[fang@study tmp]$ ls -ld test*
drwxr-xr-x.	2	root	root	6	Jun 4 14:03	test
drwxr-xr-x.	3	root	root	18	Jun 4 14:04	test1
drwx--x--x.	2	root	root	6	Jun 4 14:05	test2
```

**rmdir：remove directory 移除空目录**
对于非空的多层级目录，需要一层一层由下至上删除。也可以使用 -p 参数。
```bash
[fang@study tmp]$ ls -ld test*
drwxr-xr-x.	2	root	root	6	Jun 4 14:03	test
drwxr-xr-x.	3	root	root	18	Jun 4 14:04	test1
drwx--x--x.	2	root	root	6	Jun 4 14:05	test2
[fang@study tmp]$ rmdir test
[fang@study tmp]$ rmdir test1
rmdir: failed to remove 'test1': Directory not empty
[fang@study tmp]$ rmdir -p test1/test2/test3/test4
[fang@study tmp]$ ls -ld test*
drwx--x--x.	2	root	root	6	Jun 4 14:05	test2
```

## 执行文件路径变量：$PATH
ls指令的完整文件名为 /bin/ls，但是不管在任何目录下执行该指令都只需输入ls，其原因在于环境变量**PATH**。当我们在执行ls指令时，==系统会去PATH中定义的目录下搜寻文件名为ls的可执行文件，并执行搜寻到的第一个同名文件==。使用echo指令可以打印出PATH中定义的目录。

```bash
[root@study ~]# echo $PATH
/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin:/root/bin
[root@study ~]# exit
[fang@study ~]$ echo $PATH
/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin:/home/fang/.local/bin:/home/fang/bin
```
可见，root和用户fang的PATH中都定义了 /bin 或 /usr/bin 目录，所以在这两个用户的任何目录下都能直接执行 ls 指令。

不同用户预设的PATH不同，因此默认能够直接执行的指令也不同。PATH可以被修改。比如，可以向PATH中添加新的目录 /fang。

```bash
[fang@study ~]$ PATH="${PATH}:/fang"   # 冒号一定不能忘了！！！
```

# 文件与目录管理
## 文件与目录的检视
**ls**指令可以列出当前目录或给定目录下的文件信息。指令格式为 `ls [-aAdfFhilnrRSt] [文件名或目录名]`。主要的选项与参数包括：
* **-a**：列出所有文件，包括以 **.** 开头的隐藏文件（**常用**）；
* -A：列出所有文件，包括隐藏文件，但不包括 **.** 和 **. .** 这两个目录；
* **-d**：仅列出目录本身，而不包括目录内的文件数据（**常用**）；
* -f：对列出的结果**不**进行排序；
* **-F**：在文件名末尾附加数据结构信息。*: 代表可执行文件，/: 代表目录，=: 代表socket文件，|: 代表FIFO文件；
* -h：将文件容量以易读的方式列出（GB、KB）；
* -i：列出inode号码；
* **-l**：长数据串行出，包括文件属性与权限等信息（**常用**）；
* -n：列出UID和GID，而非用户和群组的名称；
* -r：将排序结果反向输出；
* **-R**：连同子目录的内容一同列出来；
* -S：以文件大小进行排序，而不是以文件名排序；
* -t：按时间排序，而不是以文件名排序；
* --full-time：列出完整的时间格式（年、月、日、时、分）。

## 复制、移动与删除
**复制文件或目录：cp**
指令格式为 `cp [-adfilprsu] 来源文件 目标文件`。主要的选项与参数包括：
* **-a**：相当于 -dr -\-preserve=all（**常用**）；
* -d：若**来源文件**为链接文件（link file），复制**链接文件**属性而非文件本身；
* -f：force，若**目标文件**已存在且无法开启，则移除后再尝试一次；
* **-i**：若目标文件已存在，在覆盖前先询问（**常用**）；
* -l：复制成为**硬链接**文件（hard link，与源文件inode相同），而非复制文件本身；
* **-p**：连同文件的属性（权限、用户、时间）一同复制（**常用**）；
* **-r**：用于目录的递归复制（**常用**）；
* -s：复制成为**符号链接**文件（symbolic link，与源文件inode不同），即“快捷方式”；
* -u：update，目标文件不存在或者比来源文件更旧时才复制**更新**文件；
* \-\-preserve=all：复制文件的所有属性。除 -p 的权限相关参数外，还复制SELinux属性（与强制访问控制安全有关）。

cp指令可以在目的文件名中**重命名**复制的文件。参数 -u 常用于**备份**工作。没有任何参数时，cp复制的是源文件，而非链接文件的属性。可以同时将多个文件复制到同一个目录下。 

**移除文件或目录：rm**
指令格式为 `rm [-fir] 文件或目录`。主要的选项与参数包括：
* -f：force，忽略不存在的文件，不会出现警告信息；
* -i：在删除前询问；
* -r：**递归删除**目录和目录下的文件（谨慎使用！！！）。

**移动或重命名文件：mv**
指令格式为 `mv [-fiu] 来源文件 目标文件`。主要的选项与参数包括：
* -f：force，如果目标文件已经存在，不会询问而是直接覆盖；
* -i：覆盖已存在的目标文件前先询问；
* -u：update，若目标文件已存在，只有在来源文件较新时才会更新目标文件。

## 路径的文件名与目录名
当文件路径较长时，可以使用 basename 和 dirname 指令分别获取路径中的文件名与目录名。

```bash
[fang@study ~]$ basename /etc/sysconfig/network
network
[fang@study ~]$ dirname /etc/sysconfig/network
/etc/sysconfig
```

# 文件内容查看
查看文件内容通常用到以下指令：==cat、tac、nl、less、more、head、tail、od==。
## 直接查阅文件内容
直接查阅文件内容可以使用cat、tac、nl指令。
**cat**支持由第一行开始显示文件内容。指令格式为 `cat [-AbEnTv] 文件名`。其选项与参数包括：
* -A：相当于-vET，可以列出一些特殊字符；
* -b：列出行号，空白行不标行号；
* -E：将结尾的断行字符 $ 显示出来；
* -n：打印出行号，且空白行也有行号；
* -T：将tab按键以 ^I 显示出来；
* -v：列出一些看不出来的特殊字符。

与cat相反，**tac**支持从最后一行到第一行反向在屏幕上显示出来。 

**nl**支持显示时输出行号。指令格式为 `nl [-bnw] 文件名`。其选项与参数包括：
* -b a：空行也列出行号，相当于 cat -n；
* -b t：空行不列出行号（默认）；
* -n ln：行号数字在行号字段的最左边显示；
* -n rn：行号数字在行号字段的最右边显示，且不加0；
* -n rz：行号数字在行号字段的最右边显示，且加0；
* -w：行号字段占用的字符数。

## 可翻页查阅
**more**支持一页一页地显示文件内容。指令格式为 `more 文件名`。在more程序运行中可以使用如下按键：
* 空格键：向下翻一页；
* Enter：向下前进一行；
* /字符串：在显示的内容中搜索该字符串；
* **:f**	：显示文件名以及目前显示的行数；
* **q**	：退出more，不再显示文件内容；
* **b** 或 **Ctrl-b**：往回翻页，只对一般文件有效。

**less**与more类似，而且支持向前翻页。指令格式为 `less 文件名`。在less程序运行中可以使用如下按键：
* 空格键：向下翻一页；
* PageDown：向下翻一页；
* PageUp：向上翻一页；
* /字符串：向下搜索字符串；
* ?字符串：向上搜索字符串；
* **n**	：重复前一个搜索（与 / 或 ？有关）；
* **N**	：反向重复前一个搜索（与 / 或 ？有关）；
* **g**	：前进到这个资料的第一行；
* **G**	：前进到这个资料的最后一行；
* **q**	：退出less程序。

## 其他查阅指令
 **head**可以只查看文件前面几行。参数 **-n** 后可以设定要显示的行数。**tail**可以只查看文件结尾几行。同样可以通过参数 **-n** 设定要显示的行数。
 
 **Question**：如何显示文件 /etc/man_db.conf 的第10行到第20行？
 **Answer**：使用管道 |。 
```bash
head -n 20 /etc/man_db.conf | tail -n 10
```

**od指令**
如何读取非文本文档，比如执行文档 /usr/bin/passwd 的内容？执行文档通常是二进制文件（**binary file**），因此可以使用 **od** 指令读取。指令格式为 `od [-t TYPE] 文件名`。其参数与选项包括：
* -t a：使用默认字符输出；
* -t c：使用ASCII字符输出；
* -t d[num]：使用十进制（decimal）输出，每个整数占用 num bytes；
* -t f[num]：使用浮点数（float）输出，每个整数占用 num bytes；
* -t o[num]：使用八进制（octal）输出，每个整数占用 num bytes；
* -t x[num]：使用十六进制（hexadecimal）输出，每个整数占用 num bytes。

**touch指令**
Linux的文件存在以下三个时间参数：
* modification time (**mtime**)：文件的内容发生改变时会更新 mtime；
* status time (**ctime**)：文件的状态（比如属性、权限）发生改变时会更新 ctime；
* access time (**atime**)：当文件的内容被读取时会更新 atime。

前面提到的 ==ls 指令默认会显示文件的 mtime属性==。使用 \-\-time选项可以显示atime和ctime。
```bash
ls -l /etc/man_db.conf
ls -l --time=atime /etc/man_db.conf
ls -l --time=ctime /etc/man_db.conf
```

==touch指令可以修改文件的时间属性，若文件不存在则会创建一个新文件==。==默认更新为当前的日期或时间，且同时更新atime和mtime（ctime无法被修改）==。指令格式为 `touch [-acmdt] 文件名`。其选项与参数包括：
* -a：仅更新 atime；
* -c：若文件不存在，则不创建新文件；
* -d：后接要更新的日期，而不是当前日期，也可使用 \-\-date="日期或时间"；
* -m：仅更新 mtime；
* -t：后接要更新的日期，而不是当前日期，格式为[YYYYMMDDhhmm]。



References
[1] [硬链接与符号链接的区别](https://blog.csdn.net/qq_28897525/article/details/80657465)

