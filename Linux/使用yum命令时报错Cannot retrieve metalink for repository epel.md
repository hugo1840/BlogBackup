@[TOC](使用yum命令时报错Cannot retrieve metalink for repository epel)

# 问题背景
通过Yum安装ftp时报如下错误：
```bash
$ yum search ftp
Loaded plugins: product-id, search-disabled-repos, subscription-manager
This system is not registered with an entitlement server. You can use subscription-manager to register.


 One of the configured repositories failed (Unknown),
 and yum doesn't have enough cached data to continue. At this point the only
 safe thing yum can do is fail. There are a few ways to work "fix" this:

     1. Contact the upstream for the repository and get them to fix the problem.

     2. Reconfigure the baseurl/etc. for the repository, to point to a working
        upstream. This is most often useful if you are using a newer
        distribution release than is supported by the repository (and the
        packages for the previous distribution release still work).

     3. Run the command with the repository temporarily disabled
            yum --disablerepo=<repoid> ...

     4. Disable the repository permanently, so yum won't use it by default. Yum
        will then just ignore the repository until you permanently enable it
        again or use --enablerepo for temporary usage:

            yum-config-manager --disable <repoid>
        or
            subscription-manager repos --disable=<repoid>

     5. Configure the failing repository to be skipped, if it is unavailable.
        Note that yum will try to contact the repo. when it runs most commands,
        so will have to try and fail each time (and thus. yum will be be much
        slower). If it is a very temporary problem though, this is often a nice
        compromise:

            yum-config-manager --save --setopt=<repoid>.skip_if_unavailable=true

Cannot retrieve metalink for repository: epel/x86_64. Please verify its path and try again
```

# 问题分析
检查出现问题环境的服务器源：
```bash
$ ls /etc/yum.repos.d/
epel.repo  epel-testing.repo  redhat.repo  rhel-source.repo  rhel-source.repo.oldyum
```

正常环境的服务器源：
```bash
$ ls /etc/yum.repos.d/
redhat.repo  rhel-source.repo
```

应该是问题环境中安装epel源的问题。

# 问题处理
删掉问题环境中的epel源（可以预先备份到其他地方），然后执行`yum clean all`清理yum缓存。

```bash
$ rm -f /etc/yum.repos.d/epel*
$ rm -f /etc/yum.repos.d/rhel-source.repo.oldyum
```
