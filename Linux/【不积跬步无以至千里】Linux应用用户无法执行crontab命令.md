@[TOC](Linux应用用户无法执行crontab命令)

# 问题背景
应用反馈脚本报错，经检查发现应用用户appuser执行crontab命令时会报如下错误：
```bash
$ crontab -l
You (appuser) are not allowed to access to (crontab) because of pam configuration.
```

# 问题分析
应用同时反馈登录时会提示修改密码。

使用root用户执行以下命令查看用户是否过期：
```bash
$ chage -l appuser
```

发现用户果然已经过期。可以确定是因为用户过期导致无法执行crontab命令。

# 问题处理
使用root用户执行下面的命令将应用用户设置为永不过期：
```bash
$ chage -M 99999 appuser
```

应用用户无需进行改密操作。

如果应用修改了appuser的密码，现在需要将新密码重置为以前的旧密码，由于密码策略限制，应用用户在使用`passwd appuser`命令修改回原来的密码时，会收到如下报错：

```bash
Password has been already used. Choose another.
```

使用root用户将应用用户密码重置为旧密码（不修改密码策略）：
```bash
$ echo 'appuser:oldPassword' | chpasswd
```
