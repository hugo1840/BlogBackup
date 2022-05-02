@[TOC](Linux学习笔记（十）：Linux账号管理和访问控制)
# 账号与群组
每个Linux账号至少会使用两个ID，即**UID**（User ID）和**GID**（Group ID）。使用指令`id 账户名`可以查看。文件`/etc/passwd`中记录的账号信息的每行第三个和第四个字段分别代表UID和GID。文件`/etc/group`中记录的用户组信息的每行第三个字段代表对应的GID。
## 账号
用户在登录Linux终端时一般会用到两个系统文件。首先系统会在`/etc/passwd`文件中寻找输入的账号及对应的UID、GID、家目录和Shell设定等信息，然后系统会在`/etc/shadow`文件中核对输入的密码与该账号是否匹配。

**/etc/passwd**
`/etc/passwd`文件中的每一行对应一个账号。除了root账号、普通用户账号之外，还包括许多**系统账号**如bin, adm, daemon, nobody。切忌不要随便删除这些系统账号信息。每一行的账号信息包含七个字段，以冒号分隔，依次为

- 账号名：登录Linux用的账号名；
- 密码：以字母`x`代替，实际密码存储在`/etc/shadow`文件中；
- **UID**：用户ID。**0**代表**系统管理员**，具有root权限；**1\~999**代表系统账号，保留给系统使用，通常不可作为登录使用；1000及以上代表一般用户账号；
- **GID**：用户组ID；
- 用户信息说明：解释账号用途；
- **家目录**：该账户的家目录；
- **Shell**：该账户预设的Shell。

```bash
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
```

**/etc/shadow**
`/etc/shadow`文件中每一行的账号信息包含九个字段，以冒号分隔，依次为

- 账号名
- 密码：经过加密后的密码；
- **最近修改密码的日期**：以1970年1月1日作为第一天累加得到的天数；
- **密码不可被更改的天数**：从最近修改密码的日期起，在给定天数之内密码不能被更改，0代表随时可以修改；
- **密码需要重新变更的天数**：从最近修改密码的日期起，在给定天数结束之前必须重置密码，否则密码将**过期**；
- **密码需要变更期限前的警告天数**：在密码将要过期时，提前给定的天数予以警告；
- **密码过期后的账号宽限天数**：密码过期后用户仍然可以登录系统取得bash，但是如果在给定的宽限时间内没有重置密码的话，密码就会**失效**。密码失效后，就不能再使用该密码登录系统了；
- **账号失效日期**：以1970年1月1日作为第一天累加得到的天数，即直接指定密码的失效日期，而不管该密码是否已经过期；
- 保留：该字段保留给新功能使用。

```bash
root:$w$6$wtbCce/PxMe5wm$Jr.XP6oacai/T7KFh...:16559:0:99999:7:::
```

**忘记密码怎么办**
一般用户忘记密码，请管理员使用`passwd`指令直接设定新密码即可。如果是root用户忘记密码，则需要重启进入单人维护模式修改密码；或者，以Live CD开机后挂载根目录，然后去`/etc/shadow`中清空root账号的密码字段，重启后无需密码即可登录，最后再使用`passwd`修改密码即可。

## 群组
与群组相关的配置文件是`/etc/group`和`/etc/gshadow`。

**/etc/group**
`/etc/group`文件中每一行的信息包含四个字段，以冒号分隔，依次为

- 组名
- 群组密码：以字母`x`代替，真正的密码在`/etc/gshadow`文件中；
- **GID**
- **该群组支持的账号名**：加入该群组的账号名，以逗号分隔。

```bash
root:x:0:fang,alex
bin:x:1:
```

**有效群组与初始群组**
一个账号可以加入多个群组。`/etc/passwd`文件中账户GID对应的群组称为**初始群组**（initial group）。账号登录后即拥有初始群组的相关权限。使用指令`groups`可以查看当前登录账号所属的所有群组，其中第一个被称为该账号的**有效群组**（effective group）。在当前登录账号下新建的文件对应的群组为该账号的有效群组。

```bash
[fang@study ~] groups
fang users root
[fang@study ~] touch testfile
[fang@study ~] ll testfile
-rw-rw-r--.	1	fang	fang	0	Jul 20 19:54	testfile
```

使用`newgrp 群组名`指令可以切换当前账号的有效群组（必须是账号所属的群组）。该指令功能是通过新建一个shell环境实现的，因此使用`exit`指令可以撤销`newgrp`操作，返回到原有的shell中。

一个账号要加入一个新群组有两种途径，一是root账号使用`usermod`指令设置，二是root账号利用`gpasswd`指令。

**/etc/gshadow**
`/etc/gshadow`文件中每一行的信息包含四个字段，以冒号分隔，依次为

- 组名
- 密码栏：开头为`!`的表示无合法密码；
- **群组管理员账号**
- **该群组支持的账号名**：与`/etc/group`相同。

# 账号管理
## 用户新增与删除
**useradd**
创建新账号可以使用`useradd`指令，其格式为`useradd [-u UID] [-g 初始用户组] [-G 次要用户组] [-mM] [-c 说明栏] [-d 家目录绝对路径] [-s shell] 账号名`。其选项与参数包括：

- -u：直接指定一个特定的UID给创建的账号；
- **-g**：指定初始用户组，并将其GID置于`/etc/passwd`对应行的第四栏；
- -G：指定账号加入的其他用户组，会修改`/etc/group`中的内容；
- **-m**：强制，要建立账号家目录（一般账号默认）；
- **-M**：强制，不要建立账号家目录（系统账号默认）；
- -c：对应`/etc/passwd`中第五栏的说明内容；
- **-d**：指定某个**绝对路径**成为家目录，而不使用默认值`/home/账号名`；
- **-r**：建立一个系统账号（UID有限制）；
- **-s**：指定shell，默认是`/bin/bash`；
- **-e**：指定账号失效日期（`/etc/passwd`第八栏），格式为`YYYY-MM-DD`；
- **-f**：指定密码过期后的宽限天数（`/etc/passwd`第七栏），0为立即失效，**-1**为永远不失效。

使用指令`useradd -D`可以查看创建用户时的选项默认值。其中，GROUP=100仅对SUSE等少数Linux发行版有效；对于RHEL、Fedora、CentOS而言，新建用户默认的初始群组GID与账号UID相同。

**passwd**
`passwd`指令用于设置账号密码。
一般用户指令格式（账号名可省略，需要输入旧密码）：
`passwd [--stdin] [账号名]`
root用户指令格式（不需要输入旧密码）：
`passwd [-l] [-u] [--stdin] [-S] [-n 日数] [-x 日数] [-w 日数] [-i 日期] 账号名`
其选项与参数包括：

- \-\-stdin：可接收管道数据作为密码输入，常用于shell脚本；
- **-l**：Lock，会在`/etc/shadow`第二栏前面加上!，使密码失效；
- **-u**：Unlock，与 **-l** 作用相反；
- -S：列出密码相关参数，即shadow文件内的大部分信息；
- **-n**：shadow的第四栏，多久不可修改密码天数；
- **-x**：shadow的第五栏，多久内必须修改密码；
- **-w**：shadow的第六栏，密码过期前的警告天数；
- **-i**：shadow的第七栏，密码失效日期。

```bash
passwd -l fang  # 使fang账号的密码失效
passwd -u fang  # 使fang账号的密码恢复正常
```

也可以使用`chage`指令列出详细的密码参数，格式为`chage -l 账号名`。

**usermod**
`usermod`指令可以用于修改账号属性，格式为`usermod [-cdegGlsuLU]`。其选项与参数与`useradd`指令基本相同，不同之处在

- **-l**：接新的账号名，即修改账号名；
- **-a**：与 **-G** 合用，可**增加**次要用户组而**非设置**；
- -L：暂时将账号密码冻结，与`passwd -l`作用相同；
- -U：解锁账号密码，与`passwd -u`作用相同。

**userdel**
`userdel [-r] 账号名`用于删除账号。选项r表示连同其家目录一起删除。

**用户功能**
**id**
`id [账号名]`用于查看账号名的UID和GID等信息。

**chsh**
即change shell。指令格式为`chsh [-ls]`。选项s表示要修改自己的shell，选项l表示列出系统上可用的shell。

## 群组新增与删除
**groupadd**
指令格式为`groupadd [-g gid] [-r] 组名`，其选项与参数包括：

- -g：设置GID；
- -r：建立系统用户组。

**groupmod**
指令格式为`groupmod [-g gid] [-n 新组名] 组名`，其选项与参数包括：

- -g：修改GID；
- -n：修改组名。

**groupdel**
指令格式为`groupdel 组名`。可以删除群组的前提是没有用户将该组作为自己的初始群组。

**用户组管理员：gpasswd**

```bash
# 设置群组密码
gpasswd groupname 
# 设置-A列表中的账号为组管理员，组管理员可以向该群组添加或移出账号
# 将-M列表中的账号加入群组
gpasswd [-A user1, ...] [-M user3, ...] groupname
# -r表示删除群组密码
# -R使群组密码失效
gpasswd [-rR] groupname
```

# 访问控制ACL
访问控制列表（Access Control List, ACL）可以针对单一用户、单一文件或目录进行rwx权限设置。如果有一个目录，需要给多个用户使用，每个用户或每个用户组的权限都不相同，就可以用到ACL。

**getfacl**
`getfacl`指令用于获取给定文件或目录的ACL设置。

```bash
[fang@study ~]$ getfacl acl_test1
# file: acl_test1
# owner: root
# group: root
user::rwx # 文件拥有者的权限
user:fang:r-x # fang账号的权限
group::r-- # 文件所属用户组的权限
mask::r-x # 文件默认的有效权限
other::r-- # 其他人拥有的权限
```

**setfacl**
`setfacl`指令用于设置给定文件的ACL规范，指令格式为`setfacl [-bkRd] [{-m | -x} acl参数] 目标文件名`。其选项与参数包括：

- **-m**：设置acl参数，不与-x同时使用；
  * 针对用户：**u:[账号列表]:[rwx]**，账号列表为空表示对文件拥有者设置；
  * 针对群组：**g:[用户组列表]:[rwx]**，用户组列表为空表示对文件所属群组设置；
  * 针对**有效权限**： **m:[rwx]**
- **-x**：删除acl参数，不与-m同时使用；
- **-b**：删除**所有的**acl参数；
- **-k**：删除**默认的**acl参数；
- **-R**：递归设置acl参数，子目录也会被设置；
- **-d**：设置acl默认参数，只对目录有效，目录下新建的文件会使用该默认设置。

其中，有效权限（effective permission）是指对用户或用户组设置的权限必须在**mask**的权限范围内才会生效。

```bash
[fang@study ~]$ touch acl_test2
[fang@study ~]$ setfacl -m u:username1:rx acl_test2
[fang@study ~]$ ll acl_test2
-rw-r-xr--+ # 权限部分多了个加号，使用getfacl查看权限
[fang@study ~]$ setfacl -m u::rwx acl_test2
-rwxr-xr--+
[fang@study ~]$ setfacl -m g:groupname1:rx acl_test2
[fang@study ~]$ setfacl -m m:r acl_test2
[fang@study ~]$ getfacl acl_test2
# file: acl_test2
# owner: fang
# group: fang
user::rwx
user:username1:r-x #effective:r--   # x权限不会生效
group::r-x
group:groupname1:r-x #effective:r--   # x权限不会生效
mask::r-- # 有效权限范围
other::r--
```

# 用户身份切换
## su
`su`指令可用于进行任何身份的切换，指令格式为`su [-lm] [-c 命令] [username]`。其选项与参数包括：

- **\-**：使用“**su -**”表示使用**login-shell**的变量文件读取方式来登录系统；相反地，单纯使用“**su**”会以**nonlogin shell**方式登录。如果username省略，则表示切换为**root**身份；
- **-l**：与“**su -**”类似，以login-shell登录，但是需要加切换的账号，即username不能省略；
- **-m**：等同于 **-p**，使用目前的环境设置，而不读取切换账号的配置文件；
- **-c**：只进行一次命令。

```bash
# Example 1
[fang@study ~]$ su	# 当前用户是fang
Password:	# 输入root的密码
[root@study fang]$ env | grep fang   # 用户身份已变成root
USER=fang
PATH=...:/home/fang/.local/bin:/home/fang/bin
MAIL=/var/spool/mail/fang
PWD=/home/fang
LOGNAME=fang
[root@study fang]$ env | grep root
HOME=/root
[root@study fang]$ exit	# 退出su环境

# Example 2
[fang@study ~]$ su -
Password:	# 输入root的密码
[root@study ~]$ env | grep root   # 环境信息全部切换为root
USER=root
PATH=...:/usr/bin:/root/bin
MAIL=/var/spool/mail/root
PWD=/root
LOGNAME=root
HOME=/root
[root@study ~]$ exit	

# Example 3
# 以root身份和环境执行一次命令
[fang@study ~]$ su - -c "head -n 3 /etc/shadow"
...
[fang@study ~]$ 	# 不用exit退出

# Example 4
[fang@study ~]$ su -l sasuke
Password:   # 输入sasuke的账户密码
[sasuke@study ~]$ su -	
Password:   # 输入root账户密码
[root@study ~]$ exit	# 退出root身份
[sasuke@study ~]$ exit	# 退出sasuke身份
[fang@study ~]$ exit  
```

## sudo
相比su需要知道切换用户的密码，==sudo的执行仅需要自己的密码，通过设置甚至不需要密码即可执行sudo==。但是仅有 **/etc/sudoers** 内的用户能够执行sudo。sudo的执行流程为：

1. 当用户执行sudo时，系统在/etc/sudoers文件中查找该用户是否有sudo执行权限；
2. 如果有执行权限，让用户输入自己的密码来确认（root用户不用）；
3. 如果密码输入成功，开始执行sudo后面接的命令。

修改/etc/sudoers文件可直接输入`visudo`命令（不建议使用`vi /etc/sudoers`）。

```bash
[root@study ~]$ visudo
......
root	ALL=(ALL)	ALL   
fang	ALL=(ALL)	ALL   # 需要新增的行
......
```
文件中四栏的意义分别是：

- **用户账号**：可以使用sudo命令的账号；
- **登录者的来源主机名**：对应账号由哪一台主机连接到本Linux主机时可以执行sudo命令，**ALL**表示可以是来自任意一台网络主机；
- **(可切换的身份)**：对应账号可以切换的身份，**(ALL)** 表示可以切换成任何身份；
- **可执行的命令**：可以用该身份执行的命令，**ALL**表示可以执行任何命令。命令需填写为**绝对路径**。

**wheel用户组**
```bash
[root@study ~]$ visudo
...... 
%wheel	ALL=(ALL)	ALL   # 大约在第98行，与系统有关
# 去掉%wheel前的注释符号，%表示用户组
......
# 将账号sasuke加入wheel用户组
[root@study ~]$ usermod -a -G wheel sasuke
# 这样任何加入wheel用户组的账号都能使用sudo
```

**免密码功能**
```bash
[root@study ~]$ visudo
...... 
%wheel	ALL=(ALL)	NOPASSWD: ALL
# 使用NOPASSED: ALL可以在执行sudo时免输入密码
......
```

**有限制的操作**

```bash
[root@study ~]$ visudo
......
user108	ALL=(root)	/usr/bin/passwd # 绝对路径命令
# 表示user108可以切换成root使用passwd命令
...... 

# 切换为user108
[user108@study ~]$ sudo passwd sasuke   # 可以修改sasuke的密码
......
[user108@study ~]$ sudo passwd   # 修改root的密码，危险！
Changing password for user root.

# 切换回root
[root@study ~]$ visudo
......
user108	ALL=(root)	!/usr/bin/passwd, /usr/bin/passwd [A-Za-z]*, !/usr/bin/passwd root
# !表示不可执行，这样设置后user108不能执行sudo passwd和sudo passwd root 
...... 
```

**使用别名**
如果有多个用户，需要能够使用sudo执行同样的命令，为了避免多次重复，可以使用别名来编辑`/etc/sudoers`文件。可以使用的别名标识符为**User_Alias**（用户别名）、**Host_Alias**（来源主机别名）、**Cmnd_Alias**（命令别名）。别名必须使用大写字母，如ADMPW、ADMPWCMD。
```bash
[root@study ~]$ visudo
......
User_Alias ADMPW = user1, user2, user3, sasuke, fang
Cmnd_Alias ADMPWCOM = !/usr/bin/passwd, /usr/bin/passwd [A-Za-z]*, !/usr/bin/passwd root
ADMPW	ALL=(root)	ADMPWCMD
......
```

# 用户信息传递
**查询用户**
查询目前已经登录的用户，使用指令`w`或`who`。
查看每个账号最近的登录时间，使用`lastlog`指令。

**用户对话**
使用`write 账号名 终端界面`给已经登录的用户发送消息。

```bash
[root@study ~]$ who
sasuke	tty3	2020-12-01 07:55
fang	tty2	2020-12-01 08:12
[root@study ~]$ write sasuke tty3
Hello there:
How you doing?
# 按下Ctrl D 退出消息编辑

[root@study ~]$ mesg n # 不想被打扰，可以设置拒收消息，但是root发来的消息不可拒收
[root@study ~]$ mesg
is n

# 使用wall指令可以对所有用户发送广播
[root@study ~]$ wall "I will shutdown my Linux Server in 5 minutes."
```

**邮箱**
使用`mail`指令可以发送邮件。用户邮箱位于`/var/spool/mail/`目录下。
