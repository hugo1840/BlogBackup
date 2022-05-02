@[TOC](ansible模块之include_tasks：为什么加了tags后导入的任务没有执行？)

本文中ansible的版本为2.9。

# 场景再现
下面是Role中的要测试的任务：

```yaml
# role01.yml
# ...前面的tasks

- name: included task for test
  include_tasks: test01.yml
  tags: 
    - test01

# ... 后面的tasks
```

在执行整个Role时，test01.yml会被正常导入playbook并执行：

```bash
$ ansible-playbook -i hosts.ini all role01.yml
```

但是当我们想通过tags单独测试这个任务时，
```bash
$ ansible-playbook -i hosts.ini all role01.yml --tags "test01"
```

奇怪的事情发生了：include_tasks本身这个任务执行成功了，但是被导入的test01.yml却并没有被执行！

# 原因分析
>在对include_tasks任务使用tags时，只会对include_tasks任务本身生效，而并**不会**对其中包含的任务生效。

那如果我们要对其中包含的任务也生效，该怎么做呢？

# 解决办法
可以通过include_tasks模块的apply参数，为包含的任务添加标签。

将上面的任务改成下面的形式

```yaml
# role01.yml
# ...前面的tasks

- name: included task for test
  include_tasks: 
    file: test01.yml
    apply:
      tags: test01
  tags: always

# ... 后面的tasks
```

然后调用即可：

```bash
$ ansible-playbook -i hosts.ini all role01.yml --tags "test01"
```

注意，上面的 `tags: always` 不能省略，否则 include_tasks本身不会被执行。always标签只对include_tasks本身生效。在调用其他tags时，include_tasks也会always执行，但是其中包含的任务不会被执行。
