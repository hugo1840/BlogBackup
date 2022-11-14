---
tags: [ansible]
title: ansible playbook实现磁盘格式化及文件系统挂载
created: '2022-11-14T09:48:35.433Z'
modified: '2022-11-14T10:13:30.005Z'
---

ansible playbook实现磁盘格式化及文件系统挂载

# 案例一：格式化整个磁盘并挂载文件系统

将磁盘sdb格式化为xfs文件系统：
```yaml
- name: Broker server - make filesystem
  filesystem:
    fstype: xfs
    dev: /dev/sdb
    opts: -n ftype=1 -L "data"
  when: ('broker_az1' in group_names) or ('broker_az2' in group_names)
```

创建要挂载的文件系统目录`/data`：
```yaml
- name: Broker server - make datadir
  file:
    path: /data
    state: directory
  when: ('broker_az1' in group_names) or ('broker_az2' in group_names)
```

获取磁盘sdb的UUID并保存到变量中：
```yaml
- name: Broker server - get sdb uuid
  shell: 'blkid -s UUID /dev/sdb | cut -d " " -f2'
  register: uuid_sdb
  when: ('broker_az1' in group_names) or ('broker_az2' in group_names)
  
- name: check uuid_sdb variable
  debug:
    msg: "uuid: {{uuid_sdb.stdout}}"
  when: ('broker_az1' in group_names) or ('broker_az2' in group_names)
```

在`/etc/fstab`中配置文件系统开机自动挂载：
```yaml
- name: Broker server - auto-mount config in /etc/fstab
  lineinfile:
    line: "{{uuid_sdb.stdout}} /data xfs defaults 0 0"
    state: present
    path: /etc/fstab
  when: ('broker_az1' in group_names) or ('broker_az2' in group_names)
```

挂载文件系统并检查挂载情况：
```yaml
- name: Broker server - mount datadir & display info
  shell: "mount -a && df -h"
  when: ('broker_az1' in group_names) or ('broker_az2' in group_names)
```

# 案例二：创建LV并挂载文件系统

利用磁盘vdb创建VG（此过程中会自动创建PV）：
```yaml
- name: Other servers - create vg_tdmq using /dev/vdb
  lvg:
    vg: vg_tdmq
    state: present
    pvs: /dev/vdb
  when: ('broker_az1' not in group_names) and ('broker_az2' not in group_names) and ('bookie_az1' not in group_names) and ('bookie_az2' not in group_names)
```

利用创建好的VG来创建LV：
```yaml
- name: Other servers - create LV data using vg_tdmq
  lvol:
    vg: vg_tdmq
    lv: lv_data
    size: 100%FREE
    state: present
  when: ('broker_az1' not in group_names) and ('broker_az2' not in group_names) and ('bookie_az1' not in group_names) and ('bookie_az2' not in group_names)
```

将创建好的LV格式化为xfs文件系统：
```yaml
- name: Other servers - make filesystem 
  filesystem:
    fstype: xfs
    dev: /dev/vg_tdmq/lv_data
    opts: -n ftype=1 -L "data"
  when: ('broker_az1' not in group_names) and ('broker_az2' not in group_names) and ('bookie_az1' not in group_names) and ('bookie_az2' not in group_names)
```

创建要挂载的文件系统目录`/data`：
```yaml
- name: Other servers - make datadir
  file:
    path: /data
    state: directory
  when: ('broker_az1' not in group_names) and ('broker_az2' not in group_names) and ('bookie_az1' not in group_names) and ('bookie_az2' not in group_names)
```

获取`lv_data`的UUID并保存到变量中：
```yaml
- name: Other servers - get vdb uuid
  shell: 'blkid -s UUID /dev/mapper/vg_tdmq-lv_data | cut -d " " -f2'
  register: uuid_vdb_lvdata
  when: ('broker_az1' not in group_names) and ('broker_az2' not in group_names) and ('bookie_az1' not in group_names) and ('bookie_az2' not in group_names)
```

在`/etc/fstab`中配置文件系统开机自动挂载：
```yaml
- name: Other servers - auto-mount config in /etc/fstab
  lineinfile:
    line: "{{uuid_vdb_lvdata.stdout}} /data xfs defaults 0 0"
    state: present
    path: /etc/fstab
  when: ('broker_az1' not in group_names) and ('broker_az2' not in group_names) and ('bookie_az1' not in group_names) and ('bookie_az2' not in group_names)
```

挂载文件系统并检查挂载情况：
```yaml
- name: Other servers - mount datadir & display info
  shell: "mount -a && df -h"
  when: ('broker_az1' not in group_names) and ('broker_az2' not in group_names) and ('bookie_az1' not in group_names) and ('bookie_az2' not in group_names)
```




