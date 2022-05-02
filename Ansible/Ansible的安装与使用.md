@[TOC](Ansible的安装与使用)

# 1. Ansible安装部署
## 1.1 前提条件
**管理节点**
-	已安装Python 2.7或Python 3.5或更高的版本；
-	可以是Debian、RHEL、CentOS、macOS、BSDs等操作系统。管理节点不能是Windows操作系统；
-	支持SSH通信方式。

**被管理节点**
-	已安装Python 2.6或Python 3.5或更高的版本；
-	可以是Debian、RHEL、CentOS、macOS、BSDs、Windows等操作系统；
-	支持SSH通信方式。

## 1.2 安装与配置
**#1 安装**
以RHEL和CentOS为例，在管理节点执行

```bash
# 检查是否已安装python
$ python --version
# 先安装epel源
$ yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
# rpm安装
$ yum install -y ansible
$ 检查版本
$ ansible --version
```

**#2 清单配置**
编辑 `/etc/ansible/hosts`，修改要管控的机器清单

```bash
all:
  children:
    group:
      children:
        groupA:
          hosts:
            ansible01:
              ansible_host: 192.168.88.10
        groupB:
          hosts:
            ansible02:
              ansible_host: 192.168.88.11
```

这里用到的是YAML格式，后文中会用到INI格式。其中，group、groupA和groupB为我们自己定义的分组，ansible01和ansible02是机器别名。

**#3 免密配置**
在ansible管控主机执行：

```bash
$ ssh-keygen  # 一路回车即可
# 将公钥拷贝到被管控机器
$ ssh-copy-id -i root@192.168.88.10
$ ssh-copy-id -i root@192.168.88.11
```


**#4 ansible配置**
Ansible的配置文件为`ansible.cfg`。Ansible搜寻配置信息的顺序为：
1.	ANSIBLE_CONFIG环境变量
2.	当前工作目录下的ansible.cfg
3.	家目录下的`~/.ansible.cfg`
4.	`/etc/ansinble/ansible.cfg`

其中，按以上顺序最先找到的配置信息会被使用。

配置文件ansible.cfg的内容大致如下：

```bash
[defaults]  # 通用默认配置
inventory = /etc/ansible/hosts  # 默认库文件位置，或者存放通信主机目录
forks = 5  # 在与主机通信时的默认并行进程数
remote_user = root  # playbooks的默认用户
module_name = command  # 默认使用的功能模块
executable = /bin/sh  # sudo执行命令默认使用的shell

[inventory]  
enabled_plugins = host_list, script, auto, yaml, ini, toml
…

[privilege_escalation]
become = False
become_user = root
become_method = sudo
…

[ssh_connection]  # ssh连接配置
pipelining = False
…

[persistent_connection]
connection_timeout = 30  # 持久化连接超时的秒数
…
```

# 2. 基本概念
**Control node**
控制节点，即安装了ansible的管理节点。在控制节点上可以使用ansible或ansible-playbook命令。控制节点可以有多个，但不能是Windows主机。

**Managed node**
被管理节点，即通过ansible管理的网络设备或服务器。也称为hosts。被管理节点上无需安装ansible。

**Inventory**
被管理节点的清单。一个清单文件也被称为hostfile，其中记录了每个被管理节点的信息，比如IP地址。

**Facts**
用于采集远程主机信息。

**Collections**
集合，Ansible内容的分发格式，其中可以包含playbooks、角色roles、功能模块modules和插件plugins。

**Modules**
功能模块，即ansible执行的代码单元。每个module都有特定的功能。可以通过task调用单个功能模块，也可以在playbook中调用多个功能模块。

**Tasks**
任务，Ansible的动作单元。使用ad-hoc命令可以执行单个task。

**Ad-hoc**
使用自带模块执行的任务。

**Playbooks**
可以重复使用的多个tasks的有序列表。Playbooks可以包含变量和任务。Playbooks使用的是YAML语言。

**Role**
对日常使用playbook的目录结构规范。

**Galaxy**
官方分享role的平台。

# 3. 小试牛刀
## 3.1 创建inventory
Ansible通过inventory获取要管理的工作节点的信息。创建`/etc/ansible/hosts`文件并在其中添加被管理节点信息（IP地址、FQDN全域名，等等）。

```bash
192.0.2.50
mail.example.com

[atlanta]
aserver.example.org
bserver.example.org

[webservers]
web1.example.com
web2.example.com
```

其中，方括号里的是分组名称。未分组的host放在默认的组`ungrouped`。组`all`中包含了所有的hosts。

## 3.2 连接远程节点
Ansible默认使用OpenSSH和当前控制节点登录用户连接远程主机。使用`-u`参数指定远程登录用户，或者在inventory文件中指定。

```bash
other1.example.com   ansible_connection=ssh   ansible_user=asb
other2.example.com   ansible_connection=ssh   ansible_user=root
```

连接远程节点默认使用SSH密钥认证，如果要使用密码认证，需要加上`--ask-pass`选项。

## 3.3 执行功能模块
一旦与被管理节点建立连接，ansible就会把执行ad-hoc命令或playbooks需要的功能模块转移到远程节点上。

```bash
# 使用ping模块ping清单中的所有节点
$ ansible all -m ping
# 在所有被管理节点执行指令
$ ansible all -a "/bin/echo hello"
aserver.example.org | SUCCESS => {
…
}
bserver.example.org | SUCCESS => {
…
```
清单中的每个被管理节点都会返回执行结果。

# 4. ad-hoc命令
Ansible的ad-hoc命令会使用`/usr/bin/ansible`命令行工具执行一个单一的task，适用于简单且无需重复的操作。其命令格式为
`ansible [-i /etc/ansible/hosts] [pattern] -m [module] -a "[module options]" `

其中，`-i`用于指定inventory清单文件位置，默认为`/etc/ansible/hosts`。pattern用于筛选执行命令影响到的hosts或组。

## 4.1 常用的ad-hoc命令
重启服务器

```bash
$ ansible atlanta -a "/sbin/reboot"  # 重启分组atlanta中的所有主机
$ ansible atlanta -a "/sbin/reboot" -f 10  # 将并行进程数提高为10，默认是5
$ ansible atlanta -a "/sbin/reboot" -u username --become  # 使用username连接hosts并使用root用户权限执行重启命令
```

检查远程主机状态

```bash
$ ansible atlanta -m setup  # 查看远程主机基本信息
$ ansible atlanta -m ping  # 测试能否ping通
```

## 4.2 常用的modules
**管理文件：copy, file**

```bash
# 远程拷贝文件
$ ansible atlanta -m copy -a "src=/etc/hosts dest=/tmp/hosts"
# 修改文件权限和拥有者
$ ansible atlanta -m file -a "dest=/srv/foo.txt mode=600 owner=root group=root"
# 逐层递归修改目录及其文件权限
$ ansible atlanta -m file -a "dest=/path/to/c mode=755 state=directory"
# 逐层递归删除目录及其文件
$ ansible atlanta -m file -a "dest=/path/to/c state=absent"
```

**管理安装包：yum**

```bash
# 确保应用已安装（即使不是最新版，也不会进行更新）
$ ansible atlanta -m yum -a "name=httpd state=present"
# 确保应用安装了指定版本
$ ansible atlanta -m yum -a "name=httpd-1.5 state=present"
# 确保应用安装了最新版本
$ ansible atlanta -m yum -a "name=httpd state=latest"
# 确保应用未安装
$ ansible atlanta -m yum -a "name=httpd state=absent"
```
其中，state参数用于声明对象的状态（absent、present、latest）。

**管理用户和组：user**

```bash
# 创建用户
$ ansible all -m user -a "name=mandela password=adm123"
# 删除用户
$ ansible all -m user -a "name=mandela state=absent"
```

**管理服务：service**

```bash
# 确保webservers组内的hosts都运行了httpd服务且已设为开机启动
$ ansible webservers -m service -a "name=httpd state=started enabled=yes"
# 重启服务
$ ansible webservers -m service -a "name=httpd state=restarted"
# 停止服务
$ ansible webservers -m service -a "name=httpd state=stopped"
```

**执行命令：command, shell, raw**

Ansible的ad-hoc命令默认调用的功能模块就是command。模块shell是通过`/bin/sh`来执行指令的，与command的不同之处在于，shell模块支持管道、输入和输出重定向、&等操作。模块raw仅用于极少数场景：一是目标机器未安装python-simplejson时，必须使用raw安装该模块才能使用所有其他模块；二是跟非python环境的网络设备交互时。
```bash
$ ansible atlanta -a "/sbin/reboot"  # 默认使用command模块
$ ansible atlanta -m shell -a "somescript.sh >> somelog.txt"
```

**执行脚本：script**

用于执行linux脚本或者Windows PowerShell脚本。
```bash
$ ansible atlanta -m script -a "/home/user/somescript.sh"
```


**管理计划任务：cron**

```bash
$ ansible atlanta -m cron -a 'name="check dirs" hour="2, 5" job="ls -alh > /dev/null" state=present' 
```

其他常用的功能模块还包括管理块设备的parted、管理文件系统的filesystem、以及挂载文件系统的mount。查看模块的详细用法的指令为 `ansible-doc 模块名`。

# 5. Ansible Playbooks
Ansible Playbooks用于需要重复执行的多个复杂任务。

## 5.1 编写Playbook
Playbook的内容可以分为以下几个部分：
-	Target section：定义将要执行playbook的远程主机组；
-	Variable section：定义playbook运行时需要使用的变量；
-	Task section：定义将要在远程主机组上执行的任务列表；
-	Handler section：定义Task执行完成以后需要调用的任务。

**Target section**
该部分的重要参数包括：
-	hosts：定义远程的主机组；
-	remote_user：ssh认证的用户；
-	become：设置为True时，表示执行tasks时获取root权限；
-	become_method：一般设置为sudo。会覆盖ansible.cfg中的默认值；
-	become_user：一般设置为root。如果设置remote_user=tom，become=True，become_user=steve，则用户tom会获取用户steve的权限。

```bash
- hosts: target-server
  remote_user: test
  name: Ansible Learning Playbook 1
  tasks:
    - name: Test shell module
      shell: df -Th
```

**Variable section**
Playbook中的变量必须以字母开头，可以包含下划线或数字。Playbook基于YAML，因此也支持使用字典形式的变量。

```bash
# 定义变量
remote_install_path: /opt/my_app_config
# 引用变量（加双引号与yaml字典区分）
"{{ remote_install_path }}"

# 定义字典变量
foo:
  filed1: one
  filed2: two
# 引用字典变量
foo['field1']  
foo.field1
```

使用register关键字可以将shell命令的输出注册为变量。

```bash
- host: webservers
  tasks:
     - name: Run a shell command and register its output as a variable
       shell: /usr/bin/foo
       register: foo_result
       ignore_errors: true
       
     - name: Run a shell command using output of the previous task
       shell: /usr/bin/bar
       when: foo_result.rc == 5
```

**Task section**
该部分包含了一个或多个任务，每个任务必须在对应主机上执行完毕才能执行下一个task。当一个task执行到某一个host失败时，此任务以后的tasks都不会在该主机上执行。每一个task相当于引用了一个module，并带上自定义的参数。

```bash
tasks:
   - name: install apache
     yum: name=httpd state=present
     
   - name: configure apache
     copy: src=files/httpd.conf dest=/etc/httpd/conf/httpd.conf
     notify:
        - restart apache
        
   - name: start apache
     service: name=httpd state=started enabled=yes
```

**Handler section**
所有的tasks执行完后才会执行Handlers。需要触发才会执行handler，其名称必须与notify的一致。Handlers的执行顺序只与handler task的顺序有关，与notify的顺序无关。

```bash
- name: Template configuration file
  ansible.builtin.template:
    src: template.j2
    dest: /etc/foo.conf
  notify:
    - Restart memcached
    - Restart apache

  handlers:
    - name: Restart memcached
      ansible.builtin.service:
        name: memcached
        state: restarted

    - name: Restart apache
      ansible.builtin.service:
        name: apache
        state: restarted
```

**Ansible Facts**
Facts是来自目标主机的变量信息，与Puppet的facts相同。Facts可以作为编写playbook的判断条件，或者作为系统巡检的指标信息。

```bash
# 查看
$ ansible hostname -m setup
# 引用
{{ ansible_default_ipv4.address }}
{{ ansible_hostname }}
```

为了提高playbook的重用性，且便于后期维护，可以使用include和Role来组织playbook。

**Playbooks include**
可以在playbook中include的对象包括vars、tasks、handlers、play等等。

```bash
tasks:
- include: tasks/foo.yml
```

**Playbooks Role**
Role允许根据定义的格式对文件进行分组，从本质上讲，它是一个具有一些自动化功能的包含。角色允许将变量、文件、任务、模板、handlers放到一个文件夹中，然后include它们。在建立好一个有效的依赖关系之后，还可以在一个角色中include另外一个角色。与include一样，可以传递变量给角色。它定义了一个特定的组织结构，可以很清晰的将不同的playbooks组织到一起。典型的Role结构如下：

```bash
roles/
  role_name/
     files/
     templates/
     tasks/
     handlers/
     vars/
     defaults/
     meta/
```

通过如下方式引用

```bash
--- 
- hosts: webservers
  roles:
    - role_name
```

每个Role底下的目录中必须要有main.yml文件，其次可以包含其他被include进来的playbooks。Role也支持定义变量和判断条件。

```bash
# 定义变量
- hosts: webservers
  roles:
    - common
    - {role: foo_app_instance, dir: 'path/to/c', port: 8080}

# 判断条件
- hosts: webservers
  roles:
    - {role: some_role, when: "ansible_os_family == 'CentOS'"}
```

在Ansible Galaxy中可以自由发布、共享和下载各种Role。下载的命令为`ansible-galaxy install username.rolename`

## 5.2 执行Playbook
```bash
$ ansible-playbook playbook.yml --verbose  # 执行playbook并输出详细信息
```


References:
[1\] https://docs.ansible.com/ansible/latest/installation_guide/index.html
[2\] https://github.com/ansible/ansible/blob/devel/examples/ansible.cfg
[3\] https://docs.ansible.com/ansible/latest/user_guide/basic_concepts.html
[4\] https://docs.ansible.com/ansible/latest/user_guide/intro_adhoc.html
[5\] https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html
[6\] https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html
[7\] https://docs.ansible.com/ansible/latest/user_guide/playbooks_handlers.html



