@[TOC](Linux学习笔记（四）：默认权限、特殊权限与文件搜索)
# 默认权限与隐藏权限
## 文件预设权限：umask
umask指令可以指定当前用户在创建文件或目录时的默认权限值。在终端直接输入umask会以数字形式显示默认权限值。输入 **umask -S** 则会以字符形式显示。

==umask输出的数字表示的是要减去的权限值==。比如数字形式 **0022**，其中第一位代表**特殊权限**，后三位数字依次对应所属用户、群组和其他用户（数字0代表所属用户没有被移除任何权限，数字2代表群组和其他用户的可读权限w被移除）。

创建文件与目录的预设情况为：
* 若创建的是没有执行权限的一般文件，预设权限为 **-rw-rw-rw-**；
* 若创建的是目录，默认所有权限开放，即 **drwxrwxrwx**。

当umask的值为0022时，移除相应权限后，用户创建文件和目录的默认权限值分别变为：
* 创建一般文件时：**\-rw-r\-\-r-\-**
* 创建目录时：**drwxr-xr-x**

umask后加数字可以修改默认移除的权限值。

```bash
[root@study ~]$ umask
0022
[root@study ~]$ umask -S
u=rwx,g=rx,o=rx
[root@study ~]$ umask 002   # 未修改特殊权限
[root@study ~]$ umask -S
u=rwx,g=rwx,o=rx
```

## 文件隐藏属性
除了r、w、x的基本权限外，在Linux传统的Ext2/Ext3/Ext4文件系统下，还可以使用 **chattr** 指令设定其他的系统隐藏属性，并使用 **lsattr** 指令查看。

**chattr**指令格式为 `chattr [+-=] [ASacdisu] 文件或目录`。选项与参数包括：
* +：（在已存在参数的基础上）增加一个特殊参数；
* -：（在已存在参数的基础上）移除一个特殊参数；
* =：（在移除已存在参数的基础上）设定新的参数；
* A：存取此文件或目录的访问时间**atime**不会被改变；
* S：对文件进行修改时会**同步**写入磁盘中（默认是异步写入）；
* **a**：该文件只能**增加**数据，而不能删除也不能修改（**仅root用户**可设定该参数）；
* c：自动**压缩**该文件，在读取时会自动解压缩；
* d：该文件或目录**不会**被dump程序**备份**；
* **i**：该文件**不能**被删除、改名、设定链接、写入或新增数据（**仅root用户**可设定该参数）；
* s：该文件被删除时会被完全移出硬盘空间（误删后**无法找回**）；
* u：该文件被删除时不会被移出硬盘空间（误删后**可以找回**）。

```bash
[root@study ~]$ cd /tmp
[root@study ~]$ touch test0
[root@study ~]$ chattr +i test0
[root@study ~]$ rm test0
rm: remove regular empty file 'test0'? y
rm: cannot remove 'test0': Operation not permitted
[root@study ~]$ chattr -i test0
[root@study ~]$ rm test0
```

**lsattr**指令可以显示文件隐藏属性。格式为 `lsattr [-adR] 文件或目录`。其选项与参数包括：
* -a：显示隐藏文件的属性；
* -d：仅列出目录本身的属性，而非目录内的文件名；
* -R：连同子目录的数据也同时列出。

## 文件特殊权限：SUID/SGID/SBIT
**Set UID, SUID**
当**s**字符出现在文件**拥有者**的**x**权限位置时，就被称为SUID权限。其限制与功能包括：
* 仅对二进制程序（**binary program**）有效；
* 执行者对于该程序需具有执行权限**x**；
* 本权限仅在执行该程序的过程中有效（**runtime**）；
* ==**执行者**将具有该程序拥有者（**owner**）的权限==。

SUID用途举例：一般用户使用passwd指令修改/etc/shadow中自己账户的密码。

**Set GID, SGID**
当**s**字符出现在文件**群组**的**x**权限位置时，就被称为SGID权限。SGID可以针对文件或目录来设定。

针对**文件**，其限制与功能包括：
* 对二进制程序（**binary program**）有效；
* 执行者对于该程序需具有执行权限**x**；
* 本权限仅在执行该程序的过程中有效（**runtime**）；
* ==**执行者**将具有该程序**群组**的权限==。

针对**目录**，其限制与功能包括：
* 用户对此目录具备**r**与**x**权限时，该用户能够进入此目录；
* 用户在此目录下的有效群组（**effective group**）将会变成该目录的群组；
* 若用户在此目录下具有**w**权限（可以新建文件），则 ==**使用者**创建的新文件的群组与此目录的群组相同==。

SGID用途举例：项目开发中的应用。

**Sticky Bit, SBIT**
SBIT权限（以字母**t**表示）仅针对**目录**有效。一般来说，当用户a对于目录B是群组或者其他用户的身份，并对目录B具有**w**权限时，该用户a可以对目录B下**任何用户**新建的文件和目录进行删除、移动和更名操作。但是，当目录B被设置了SBIT权限时，比如 drwxrwxrwt，==仅有文件的创建者和root用户能够删除、移动、重命名该文件==。

**SUID/SGID/SBIT权限设定**
可以使用数字形式和之前提到的chmod指令修改特殊权限。==SUID、SGID、SBIT对应的数字分别为4、2、1，且必须放在代表用户、群组和其他用户的三位数字之前==。

```bash
[root@study ~]$ cd /tmp
[root@study tmp]$ touch test1
[root@study tmp]$ chmod 4755 test1; ls -l test1
-rwsr-xr-x	1	root	root	0	Jun 16 02:23	test1
[root@study tmp]$ chmod 6755 test1; ls -l test1
-rwsr-sr-x	1	root	root	0	Jun 16 02:23	test1
[root@study tmp]$ mkdir test2
[root@study tmp]$ chmod 1755 test2; ls -l test2
-rwxr-xr-t	1	root	root	0	Jun 16 02:23	test2
[root@study tmp]$ chmod 7666 test2; ls -l test2     # 数字6代表没有x权限
-rwSrwSrwT	1	root	root	0	Jun 16 02:23	test2
```
由于**s**和**t**权限都是用来取代**x**权限的，当文件拥有者、群组和其它用户都没有**x**权限时，也无法设置**s**和**t**权限。此时==特殊权限都为空，分别同大写字母**S**和**T**表示==。

同样也可以通过符号设置特殊权限。

```bash
[root@study tmp]$ chmod u=rwxs,go=x test1; ls -l test1
-rws--x--x	1	root	root	0	Jun 16 02:23	test1
[root@study tmp]$ chmod g+s,o+t test1; ls -l test1
-rws--s--t	1	root	root	0	Jun 16 02:23	test1
```

**观察文件类型：file**
利用file指令可以查看文件的格式，比如是否为ASCII文件、data文件或者binary文件，等等。

# 文件名搜索
常用于文件搜索的指令包括which、whereis、locate、find。

**which**
which指令会列出在**PATH目录**中找到的给定指令的第一个执行文档的完整文件名。若加上参数 **-a** 则会列出找到的所有指令文件名。

**whereis**
whereis指令只会查找某些特定目录下的文件，因此速度较快。使用参数 **-l** 会列出whereis会去查询的几个主要目录。

**locate**
locate指令会利用**数据库**（**/var/lib/mlocate**）来搜索文件名，速度也很快。使用参数 **-S** 会列出locate所使用的数据库文件的相关信息。使用 **updatedb** 指令可以更新locate使用的数据库，避免因为数据库未及时更新导致找不到文件。

**find**
find指令会直接搜索**硬盘**，因此速度较慢，但是功能最强大。一般在使用前面三个指令都找不到文件后，才会使用find指令。指令格式为 `find [PATH] [option] [action]`。find指令的常用参数与选项包括：
* -mtime n：在n天之前的一天内被修改过内容的文件；
* -mtime +n：在n天之前（不含n天本身）被修改过内容的文件；
* -mtime -n：在n天之内（含n天本身）被修改过内容的文件；
* -newer file：列出比文件file还要新的文件的名称；
* -uid n：n为用户账号ID，记录在/etc/passwd里；
* -gid n：n为组名ID，记录在/etc/group里；
* -user name：name为使用者账户名；
* -group name：name为组名；
* -nouser：寻找的文件的拥有者不存在于/etc/passwd；
* -nogroup：寻找的文件的群组不存在于/etc/group；
* -name filename：搜寻文件名为filename的文件；
* -size [\+\-]SIZE：寻找比SIZE还要大（+）或小（-）的文件。SIZE的规格：c代表bytes，k代表1024bytes。比如 -size +50k 即为寻找大于50KB的文件；
* -type TYPE：寻找的文件类型为TYPE，比如一般文件（f）、设备文件（d、c）、目录（d）、链接文件（l）、socket（s）、FIFO（p）等；
* -perm mode：寻找的文件的权限**刚好等于**mode；
* -perm -mode：寻找的文件的权限**囊括**mode中的所有权限，比如 -perm -0744（-rwxr-\-r-\-） 会把权限为 4755（-rwsr-xr-x） 的文件也列出来，因为后者的权限囊括了前者；
* -perm /mode：寻找的文件的权限包含mode中的**任一**权限即可；
* -exec command：command为其他指令，用于处理搜寻到的结果。

```bash
[root@study ~]$ which [-a] command
[root@study ~]$ whereis [-bmsu] filename
[root@study ~]$ locate [-ir] keyword
[root@study ~]$ find [PATH] [option] [action]
# 将系统中24小时内文件内容被修改过的文件列出
[root@study ~]$ find / -mtime 0     # 0代表目前的时间
# 在/etc下寻找日期比/etc/passwd更新的文件
[root@study ~]$ find /etc -newer /etc/passwd
# 寻找文件名包含关键词passwd的文件
[root@study ~]$ find / -name "*passwd*"
# 找出/usr/bin和/usr/sbin两个目录下具有SUID或者SGID权限的文件
[root@study ~]$ find /usr/bin /usr/sbin -perm /6000
# 将上面找到的文件用 ls -l 指令列出来, {}代表find找到的内容，\;表示指令结束
[root@study ~]$ find /usr/bin /usr/sbin -perm /6000 -exec ls -l {} \;
```

# 权限与指令的关系
| 目标 | 所需指令 | 目录所需权限 | 文件所需权限 |
| -- | -- | -- | -- |
| 使用户进入某个目录<br>使其成为可工作目录 | 变换工作目录的指令，如cd | 至少具有x权限。<br>若想使用ls指令，还需具备r权限 | 无 |  
| 用户在某个目录内读取文件 | cat, more, less | 至少具有x权限 | 至少具有r权限 |  
| 用户修改一个文件 | vi编辑器 | 至少具有x权限 | 至少具有r和w权限 |  
| 用户建立一个新文件 | touch | 至少具备w和x权限 | 无 |  
| 用户进入某个目录<br>并执行该目录下的某个指令 | 无 | 至少具有x权限 | 至少具有x权限 |  
