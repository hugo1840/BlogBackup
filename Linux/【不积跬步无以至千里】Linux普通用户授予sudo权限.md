@[TOC](Linux普通用户授予sudo权限)

# 问题背景
因为中间件问题，某个应用目录下生成的日志owner和group都是root用户，导致应用用户定期清理日志的脚本报错。

# 问题分析
通常我们只需要通过`setfacl`命令来添加ACL权限即可，但是该应用目录为NAS挂载目录，可能由于挂载方式的原因，在使用`setfacl`命令时会报错。

最初的想法是在root用户下添加应用用户日志清理脚本的crontab任务。考虑到root用户删除应用日志可能存在合规风险，最终的解决方法是为应用用户添加sudo权限，使得应用用户能以root身份来执行`rm`和`find`命令来删除日志。

# 问题处理
假设应用用户为appuser，应用日志清理脚本中需要以root身份执行`find`和`rm`命令。

使用root用户按如下方式修改`/etc/sudoers`文件，添加如下两行：

```bash
$ visudo
appuser  ALL=(root)  NOPASSWD: /usr/bin/rm, /usr/bin/find
Defaults:appuser !requiretty
#注释Defaults requiretty 
```

其中`Defaults:appuser !requiretty`这一行是为了`rm`和`find`命令能够在脚本中使用。

最后，使用appuser用户进行有效性测试即可。

