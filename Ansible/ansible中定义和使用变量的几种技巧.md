---
tags: [ansible]
title: ansible中定义和使用变量的几种技巧
created: '2022-11-05T07:04:09.062Z'
modified: '2022-11-05T09:07:08.003Z'
---

ansible中定义和使用变量的几种技巧

# Case 1：使用Inventory中自定义的全局变量
Inventory清单文件中`[all:vars]`部分定义的变量为全局变量，可以在同一级目录下的roles中的所有playbook中直接使用。使用的格式为`{{全局变量名}}`。

在Inventory中定义全局变量：
```
[all:vars]
# variables related to versions
version_major_tencentKona=8.0.8
version_minor_tencentKona=312
arch_tencentKona=x86_64 
#arch_tencentKona=aarch64
```

在playbook中使用自定义的全局变量：
```yaml
- name: unarchive TencentKona JDK
  unarchive:
    remote_src: no
    src: "/root/TencentKona{{version_major_tencentKona}}.b1_jdk_linux-{{arch_tencentKona}}_8u{{version_minor_tencentKona}}.tar.gz"
    dest: /usr/local
```

# Case 2：使用ansible内置的主机变量
Ansible内置了许多系统变量，可以直接在ad-hoc命令或者playbook中使用。这些内置变量包括但不限于：
- `ansible_architecture`：操作系统架构，arm或者x86；
- `ansible_distribution`：操作系统发行版本；
- `ansible_distribution_major_version`：操作系统大版本号；
- `ansible_processor_vcpus`：CPU核数；
- `ansible_memtotal_mb`：内存总量；
- `ansible_hostname`：主机名；
- `ansible_devices.vdb.size`：vdb盘的容量（如果磁盘vdb存在）；
- `ansible_devices.sdb.size`：sdb盘的容量（如果磁盘sdb存在）；
- `ansible_devices.nvme0n1.size`：nvme0n1盘的容量（如果磁盘nvme0n1存在）。

在playbook中使用内置变量：
```yaml
- name: check arm OR x86
  debug: 
    msg: "{{ansible_architecture}}"

- name: check OS version
  debug:
    msg: "{{ansible_distribution}}: {{ansible_distribution_major_version}}"

- name: check CPU
  debug: 
    msg: "CPU(s): {{ansible_processor_vcpus}}, type: {{ansible_processor_vcpus | type_debug}}"

- name: check Memory
  debug:
    msg: "Memory: {{ansible_memtotal_mb}} mb, type: {{ansible_memtotal_mb | type_debug}}"
```


# Case 3：使用Inventory中自定义的主机变量
可以在Inventory清单文件中每个主机IP后自定义一个或多个主机变量，并在playbook中引用。

在Inventory中定义主机变量：
```
[broker_az1]
192.168.17.83 myhostname=TBXZ1ATDMQBRK01 mycpu=64 mymem_limit=300000 mydisk_target=1.8T 

[bookie_az1]
192.168.17.177 myhostname=TBXZ1ATDMQBKE01 mycpu=112 mymem_limit=500000 mydisk_target=3.7T

[zk_az1]
192.168.16.41 myhostname=TBXZ1TDMQZK01 mycpu=16 mymem_limit=14000 mydisk_target=500G
192.168.16.42 myhostname=TBXZ1TDMQZK02 mycpu=16 mymem_limit=14000 mydisk_target=500G
192.168.16.43 myhostname=TBXZ1TDMQZK03 mycpu=16 mymem_limit=14000 mydisk_target=500G
```

在playbook中使用自定义的主机变量：
```yaml
# 检查主机名是否等于myhostname
- name: check if hostname already set
  debug: 
    msg: "----------Good Job! HOSTNAME ALREADY SET----------"
  when: ansible_hostname == myhostname 

# 检查CPU核数是否等于mycpu
- name: ABORT if CPU not the same as given in hosts file
  fail:
    msg: "----------Abort playbook Cuz CPU not as Required----------"
  when: ansible_processor_vcpus != mycpu

# 检查内存是否小于mymem_limit
- name: ABORT if Memory not greater than given in hosts file
  fail:
    msg: "----------Abort playbook Cuz Memory not as Required----------"
  when: ansible_memtotal_mb <  mymem_limit

# 检查sdb盘大小是否满足需求
- name: check data disk for broker server
  debug:
    msg: "{{ansible_hostname}}, sdb: {{ansible_devices.sdb.size}}, TARGET: {{mydisk_target}}"
  when: ('broker_az1' in group_names) or ('broker_az2' in group_names)
```


# Case 4：在jinja2文件中引用Inventory变量
在templates下的jinja2模板文件中也可以引用Inventory清单文件中的主机IP和自定义变量。使用`groups['分组名'][i]`可以获取指定分组中第`i+1`台主机的IP地址。

在Inventory文件中定义如下主机IP和变量：
```
[mon_az1]
192.168.16.44 myhostname=TBXZ1TDMQMON01 mycpu=16 mymem_limit=14000 mydisk_target=600G
192.168.16.45 myhostname=TBXZ1TDMQMON02 mycpu=16 mymem_limit=14000 mydisk_target=600G
192.168.16.46 myhostname=TBXZ1TDMQMON03 mycpu=16 mymem_limit=14000 mydisk_target=600G

[mgm_az1]
192.168.16.49 myhostname=TBXZ1TDMQMANAGER01 mycpu=16 mymem_limit=30000 mydisk_target=500G

#####################
[all:vars]
user_redis=redis
pass_redis=Abc@77yy
```

在jinja2模板文件中使用：
```
#[Redis] 
redis={{ groups['mgm_az1'][0] }}     # 表示mgm_az1分组中第一台主机的IP
rsuser={{user_redis}}
rspasswd={{pass_redis}}

# 通过for循环遍历mon_az1和mon_az2分组中所有的主机IP
monitor_bk_servers_node_ip={% for sr in groups['mon_az1'] %}{{sr}} {% endfor %} {% for st in groups['mon_az2'] %}{{st}} {% endfor %}
```


# Case 5：通过set_fact模块自定义变量
可以在playbook中使用`set_fact`模块来定义变量，并在后续的任务中使用。

```yaml
- name: Block for port activity checking
  block:
    - name: check task 1
      ...
    - name: check task 2
      ...
    - name: check task 3
      ...

    - name: set var when all ports checking OK
      set_fact:
        is_port_checking_ok: true
    - name: servers whose port checking OK
      debug:
        msg: "IS_PORT_CHECKING_OK: {{is_port_checking_ok}}"

  rescue:
    - debug:
        msg: "There is port checking FAILED!!!"
    - name: set var when port checking FAILED
      set_fact:
        is_port_checking_ok: false
    - name: servers whose port checking NOT OK
      debug:
        msg: "{{ansible_hostname}}, IS_PORT_CHECKING_OK: {{is_port_checking_ok}}"
```

# Case 6：使用register保存shell命令的输出到变量中
在playbook中使用register关键字将shell模块的输出保存到自定义变量中，并在后续task中使用。

下面的Block可以实现在操作系统中未安装python时安装指定的python版本。
```yaml
- name: check python on RHEL8
  block:
  - name: check if python2 installed
    shell: "python2 -V"
    register: return_msg_python2
    ignore_errors: true

  - name: check if python3 installed
    shell: "python3 -V"
    register: return_msg_python3
    ignore_errors: true
  
  - name: install & link python3
    block:
    - name: install python3.6
      yum:
        name: python36
        state: present
    - name: link python3 binary file
      file:
        path: /usr/bin/python
        src: /usr/bin/python3
        state: link
    when: " return_msg_python2 is failure and return_msg_python3 is failure "
  when: " ansible_distribution_major_version == '8' "
```


# Case 7：限制task仅在个别分组的主机上执行
通过使用`when: '主机分组名' in group_names`在限制单个task仅在指定分组内的主机上执行。也可以使用逻辑运算符`NOT`、`OR`或者`AND`来实现取反以及连接多个分组的目的。

>:chicken:不支持两个以上OR条件连接，但是AND条件不会限制连接分组的个数。

示例：
```yaml
# 表示下面的任务仅在broker_az1和broker_az2两个分组内的主机上执行
- name: Broker server - make filesystem
  filesystem:
    fstype: xfs
    dev: /dev/sdb
    opts: -n ftype=1 -L "data"
  when: ('broker_az1' in group_names) or ('broker_az2' in group_names)

# 表示面的任务仅在除broker_az1、broker_az2、bookie_az1和bookie_az2以外的所有分组中执行
- name: Other servers - create vg_tdmq using /dev/vdb
  lvg:
    vg: vg_tdmq
    state: present
    pvs: /dev/vdb
  when: ('broker_az1' not in group_names) and ('broker_az2' not in group_names) and ('bookie_az1' not in group_names) and ('bookie_az2' not in group_names)
```


# Case 8：在lineinfile正则匹配中使用分组
在使用lineinfile模块时，设置`backrefs: yes`开启反向引用，在`regexp`参数中使用**小括号**来对目标行中的内容进行分组匹配，在line参数中使用`\1`、`\2`、`\3`来从左至右依次引用`regexp`参数中划分的分组。

```yaml
# line后面的\1会被替换为regexp中(\s*[^#]host:\s*)匹配到的内容
- name: config elb vip in application.yml for access-gateway az1
  lineinfile:
    path: /usr/local/cloud-access-gateway/application.yml
    line: '\1http://{{ mgm_vip_az1 }}:7001'
    regexp: '(\s*[^#]host:\s*)http://.*:7001'
    state: present
    backrefs: yes
  when: " 'acgw_az1' in group_names "
```

>:snake: 如果regexp在目标文件中匹配到多行，line后面的操作只会对**匹配到的最后一行**生效。




