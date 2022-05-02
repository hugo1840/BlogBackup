@[TOC](【漏网之鱼】Ansible AWX调用playbook传参问题)

# 问题背景
通过ansible AWX使用Web接口调用playbook时，出现了playbook在运行时无法读取某些变量、以及读取到的某些变量与定义的值不同的情况。Playbook的结构如下：

```bash
my-test-playbook
|--ansible.cfg
|--fact_files
|--group_vars
   |--all.yml
|--deploy.yml
|--inventory.ini
|--log
  |--ansible.log
|--README
|--retry_files
|--roles
  |--rolePremiere
    |--defaults
      |--main.yml
    |--tasks
      |--main.yml
    |--files
    |--templates
      |--app.conf.j2
    |--vars
  |--roleSecond
  |--roleThird
|--rollback.yml
|--start.yml
|--stop.yml
```

# 问题一：读不到inventory.ini中的变量
在`inventory.ini`中定义如下内容：
```yaml
[appNode]
192.168.x.A ansible_ssh_port=xxx ansible_ssh_user=root
192.168.y.B ansible_ssh_port=xxx ansible_ssh_user=root

[all:vars]
major_version = 5
basedir = /opt/app
is_cluster = true
is_consistent = false
```

在执行`roles/rolePremiere/tasks/main.yml`中的相关任务时，在使用inventory.ini中`[all:vars]`下定义的变量时，竟然提示该变量未定义！但是`[appNode]`中的hosts信息可以读到。

奇怪的是，当我把这些变量移动到`group_vars/all.yml`或者是对应role下的`defaults/main.yml`中后，就可以读取到相应的变量了！


# 问题二：读到的变量与定义值不同
我在`roles/rolePremiere/templates/app.conf.j2`中使用到了变量`is_consistent`作为判断的条件，如下

```yaml
{% if ( is_consistent == "false" ) %}
```
根据上面定义的值（假设该变量定义在`group_vars/all.yml`中），这个判断条件应该成立。但事实上，if后面对应的语句却并没有执行。

通过检查日志和目标机器上的文件发现，在调用该模板文件时，该变量的值变成了`False`，即首字母变成了大写，导致判断条件不成立。

绕行的办法是改写上面的判断语句为
```yaml
{% if ( is_consistent | string | lower == "false" ) %}
```

