@[TOC](Ansible基本命令、角色、内置变量与tests判断)

# 常用基本命令
查看ansible版本

```bash
$ ansible --version
```

查看ansible的模块

```bash
# 查看支持的所有模块
$ ansible-doc -l
# 查看某个模块的介绍（以ping为例）
$ ansible-doc -s ping
```

执行ad-hoc命令

```bash
# 默认inventory路径可省略
$ anisble [-i INVENTORY路径] 主机别名或分组 -m 模块名 -a "模块参数"
$ ansible [-i /etc/ansible/hosts] webserver01 -m ping  # 使用主机别名
$ ansible webservers -m shell -a "hostname"  # 使用分组
# 使用非默认inventory路径
$ ansible -i /tmp/dbservers all -m shell -a "ip addr"  # 使用all关键字
```

执行playbook

```bash
# 检查yaml文件语法
$ ansible-playbook --syntax-check test.yml
# 试运行yaml剧本（模拟运行）
$ ansible-playbook --check test.yml
# 真正执行playbook
$ ansible-playbook [-i /path/hosts] test.yml
```

# Role目录结构

```bash
runRole.yml  # 用于调用同目录下的角色
testrole  # 角色
├── defaults  # 定义角色使用的变量的默认值
│   └── main.yml  
├── files  # 定义角色用到的文件
│   └── main.yml  
├── handlers  # 定义角色用到的handlers
│   └── main.yml  
├── meta  # 描述角色属性的元数据
│   └── main.yml  
├── tasks  # 角色需要执行的主任务文件
│   └── main.yml  
├── templates  # 角色用到的jinja2模板文件
│   └── test.conf.j2  
└── vars  # 角色使用的变量值，优先级高于defaults和runRole.yml中定义的变量
    └── main.yml  
```

其中，runRole.yml 的内容如下
```yaml
---
- hosts: webservers
  remote_user: root
  gather_facts: no
  roles:
  - role: testrole
    vars:  # 优先级低于vars/main.yml，高于defaults/main.yml中的同名变量
      var1test: "var_in_yaml"  
  - role: another_role  # 可调用多个角色
```

# 内置变量
## ansible_version
获取ansible版本号

```bash
ansible webserver01 -m debug -a "msg={{ansible_version}}"
```

## ansible_distribution
获取到远程主机的操作系统信息，比如RHEL、CentOS。默认 `gather_facts=true` 时会收集。大版本号为 `ansible_distribution_major_version`。

## ansible_python_version
获取到远程主机安装的python版本号。默认 `gather_facts=true` 时会收集。

## hostvars
获取远程主机的信息

```yaml
---
- name: "play 1: Gather facts of server02"
  hosts: server02
  remote_user: root
  #默认gather_facts: yes
 
- name: "play 2: Get facts of server02 when operating on server01"
  hosts: server01
  remote_user: root
  tasks:
  - debug:
      msg: "{{hostvars['server02'].ansible_ens35.ipv4}}"
      #或者msg: "{{hostvars.server02.ansible_ens35.ipv4}}"
```

## inventory_hostname
获取到被操作的当前远程主机的在inventory中的名称

```bash
ansible servergroup -m debug -a "msg={{inventory_hostname}}"
```

通过 `{{inventory_hostname_short}}` 也可以获取当前play操作的主机在清单中对应的名称，但是这个名称更加简短。

## play_hosts
获取到当前play所操作的所有主机的主机名列表

```yaml
---
- hosts: server01,server02
  remote_user: root
  gather_facts: no
  tasks:
  - debug:
      msg: "{{play_hosts}}"
```

## groups
获取到清单中”所有分组”的分组信息

```bash
ansible server01 -m debug -a "msg={{groups}}"
ansible server01 -m debug -a "msg={{groups.ungrouped}}"
ansible server01 -m debug -a "msg={{groups.webservers}}"
ansible server01 -m debug -a "msg={{groups['webservers']}}"
```

## group_names
获取到当前主机所在分组的组名

```bash
ansible server01 -m debug -a "msg={{group_names}}"
```

## inventory_dir
获取到ansible主机中清单文件的存放路

```bash
$ ansible server01 -m debug -a "msg={{inventory_dir}}"
server01 | SUCCESS => {
    "changed": false, 
    "msg": "/etc/ansible"
}
```

# tests条件判断
Ansible没有使用Linux的test命令，而是使用了jinja2的tests，借助tests，我们可以进行一些判断操作，tests会将判断后的布尔值返回。

## 判断变量
- defined ：判断变量是否已经定义，已经定义则返回真；
- undefind ：判断变量是否已经定义，未定义则返回真；
- none ：判断变量值是否为空，如果变量已经定义，但是变量值为空，则返回真。

```yaml
---
- hosts: server01
  remote_user: root
  gather_facts: no
  vars:
    testvar: "test"
    testvar1:
  tasks:
  - debug:
      msg: "Variable is defined"
    when: testvar is defined
  - debug:
      msg: "Variable is undefined"
    when: testvar2 is undefined
  - debug:
      msg: "The variable is defined, but there is no value"
    when: testvar1 is none
```

## 判断执行结果
- success 或 succeeded：通过任务的返回信息判断任务的执行状态，任务执行成功则返回真；
- failure 或 failed：通过任务的返回信息判断任务的执行状态，任务执行失败则返回真；
- change 或 changed：通过任务的返回信息判断任务的执行状态，任务执行状态为changed则返回真；
- skip 或 skipped：通过任务的返回信息判断任务的执行状态，当任务没有满足条件，而被跳过执行时，则返回真。

```yaml
---
- hosts: server02
  remote_user: root
  gather_facts: no
  vars:
    doshell: "yes"
  tasks:
  - shell: "cat /testdir/abc"
    when: doshell == "yes"
    register: returnmsg
    ignore_errors: true
  - debug:
      msg: "success"
    when: returnmsg is success
  - debug:
      msg: "failed"
    when: returnmsg is failure
  - debug:
      msg: "changed"
    when: returnmsg is change
  - debug:
      msg: "skip"
    when: returnmsg is skip
```

## 判断路径
⚠以下tests的判断均针对于**ansible主机**中的路径，与目标主机无关。

- file : 判断路径是否是一个文件，如果路径是一个文件则返回真；
- directory ：判断路径是否是一个目录，如果路径是一个目录则返回真；
- link ：判断路径是否是一个软链接，如果路径是一个软链接则返回真；
- mount：判断路径是否是一个挂载点，如果路径是一个挂载点则返回真；
- exists：判断路径是否存在，如果路径存在则返回真。

```yaml
---
- hosts: host01  # tests路径与host01无关
  remote_user: root
  gather_facts: no
  vars:
    testpath1: "/testdir/test"
    testpath2: "/testdir/"
    testpath3: "/testdir/testsoftlink"
    testpath4: "/testdir/testhardlink"
    testpath5: "/boot"
  tasks:
  - debug:
      msg: "file"
    when: testpath1 is file
  - debug:
      msg: "directory"
    when: testpath2 is directory
  - debug:
      msg: "link"
    when: testpath3 is link
  - debug:
      msg: "link"
    when: testpath4 is link
  - debug:
      msg: "mount"
    when: testpath5 is mount
  - debug:
      msg: "exists"
    when: testpath1 is exists
```

## 判断字符串
- string：判断对象是否是一个字符串，是字符串则返回真；
- lower：判断包含字母的字符串中的字母是否是纯小写，字符串中的字母全部为小写则返回真；
- upper：判断包含字母的字符串中的字母是否是纯大写，字符串中的字母全部为大写则返回真。

```yaml
---
- hosts: host02
  remote_user: root
  gather_facts: no
  vars:
    str1: "abc"
    str2: "ABC"
  tasks:
  - debug:
      msg: "This string is all lowercase"
    when: str1 is lower
  - debug:
      msg: "This string is all uppercase"
    when: str2 is upper
```

## 判断数字
- number：判断对象是否是一个数字，是数字则返回真；
- even ：判断数值是否是偶数，是偶数则返回真;
- odd ：判断数值是否是奇数，是奇数则返回真;
- divisibleby(num) ：判断是否可以整除指定的数值，如果除以指定的值以后余数为0，则返回真。

```yaml
---
- hosts: host03
  remote_user: root
  gather_facts: no
  vars:
    num1: 4
    num2: 7
    num3: 64
  tasks:
  - debug:
      msg: "An even number"
    when: num1 is even
  - debug:
      msg: "An odd number"
    when: num2 is odd
  - debug:
      msg: "Can be divided exactly by"
    when: num3 is divisibleby(8)
```

## 判断列表
- subset：判断一个list是不是另一个list的子集，是另一个list的子集时返回真；
- superset : 判断一个list是不是另一个list的父集，是另一个list的父集时返回真。

```yaml
---
- hosts: host04
  remote_user: root
  gather_facts: no
  vars:
    a:
    - 2
    - 5
    b: [1,2,3,4,5]
  tasks:
  - debug:
      msg: "A is a subset of B"
    when: a is subset(b)
  - debug:
      msg: "B is the parent set of A"
    when: b is superset(a)
```

## 判断版本号
- version：可以用于对比两个版本号的大小，或者与指定的版本号进行对比，使用语法为 `version('版本号', '比较操作符')`。

```yaml
---
- hosts: host05
  remote_user: root
  vars:
    ver: 7.4.1708
    ver1: 7.4.1707
  tasks:
  - debug:
      msg: "This message can be displayed when the ver is greater than ver1"
    when: ver is version(ver1,">")
  - debug:
      msg: "system version {{ansible_distribution_version}} greater than 7.3"
    when: ansible_distribution_version is version("7.3","gt")
```

