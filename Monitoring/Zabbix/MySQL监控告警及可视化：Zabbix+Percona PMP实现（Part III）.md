---
tags: [监控]
title: MySQL监控告警及可视化：Zabbix+Percona PMP实现（Part III）
created: '2023-05-02T09:46:26.672Z'
modified: '2023-05-02T10:52:19.880Z'
---

MySQL监控告警及可视化：Zabbix+Percona PMP实现（Part III）

# 告警配置
## 配置告警邮箱
在Zabbix Web前端的 **Administration - Media Types - Email** 中配置发送告警信息的邮箱。需要在告警邮箱中开启POP3/SMTP/IMAP，设置第三方授权码。

![img1](https://img-blog.csdnimg.cn/5145a89d143e473cad862a9d8c942e7a.png#pic_center)
![img2](https://img-blog.csdnimg.cn/d3e7c2b25994480e91ba7408b2020e0b.png#pic_center)

:eagle:**注意**：这里的**Password**不是邮箱登录密码，而是邮箱中配置的**第三方授权码**。

![img3](https://img-blog.csdnimg.cn/dd6788cf32ea4d888b4cbbd40791683f.png#pic_center)

配置完成后点击**Test**检查是否能够正常发送邮件。

![img4](https://img-blog.csdnimg.cn/f9b45effb43441ab9c69403eb4696c0e.png#pic_center)


## 配置告警消息模板
然后在 **Administration - Media Types - Email - Message templates** 中配置邮件发送告警消息模板。

告警发生消息模板：
```
Subject: 
故障: {TRIGGER.STATUS}, 服务器: {HOSTNAME1}, 发生{TRIGGER.NAME}故障!

Message:
告警主机: {HOSTNAME1}
告警时间: {EVENT.DATE} {EVENT.TIME}
告警等级: {TRIGGER.SEVERITY}
告警信息: {TRIGGER.NAME}
告警项目: {TRIGGER.KEY1}
问题详情: {ITEM.NAME}:{ITEM.VALUE}
当前状态: {TRIGGER.STATUS}:{ITEM.VALUE1}
事件ID: {EVENT.ID}
```

![img5](https://img-blog.csdnimg.cn/02695b418a834f479153523e36721cda.png#pic_center)

告警恢复消息模板：
```
Subject:
恢复: {TRIGGER.STATUS}, 服务器: {HOSTNAME1}, {TRIGGER.NAME}已恢复!

Message:
告警主机: {HOSTNAME1}
告警时间: {EVENT.DATE} {EVENT.TIME}
告警等级: {TRIGGER.SEVERITY}
告警信息: {TRIGGER.NAME}
告警项目: {TRIGGER.KEY1}
问题详情: {ITEM.NAME}:{ITEM.VALUE}
当前状态: {TRIGGER.STATUS}:{ITEM.VALUE1}
事件ID: {EVENT.ID}
```

![img6](https://img-blog.csdnimg.cn/cbd9e0c8f2aa4396894cd4f166fe0022.png#pic_center)

## 配置告警用户

在 **Administration - User groups**中创建用户组dbreaderg，并对Percona templates添加读权限。

![img7](https://img-blog.csdnimg.cn/d50fce22f37146ca9135d43e9081698f.png#pic_center)
![img8](https://img-blog.csdnimg.cn/5457c0df078c4a128dea4c0a721f442a.png#pic_center)

在 **Administration - Users**中创建用户dbreader，并添加到用户组dbreaderg。

![img9](https://img-blog.csdnimg.cn/889a702605a5440aa4b785a6ec7f2429.png#pic_center)

为dbreader用户添加接收告警的邮箱地址，并授予**User role**普通用户权限。

![img10](https://img-blog.csdnimg.cn/f8e8b4d98d2546d08583dfecee00f740.png#pic_center)
![img11](https://img-blog.csdnimg.cn/5c2d468785994fce881812a4e6101e02.png#pic_center)


## 配置告警规则
在 **Configuration - Templates - Percona MySQL Server Template - Triggers** 中查看Percona PMP插件中已经内置的触发器。

![img12](https://img-blog.csdnimg.cn/921ef7b4b7e64f2c866b0ec7f7490761.png#pic_center)
![img13](https://img-blog.csdnimg.cn/25ff370b3fef467481d95603a5719afb.png#pic_center)

我们选择“**MySQL is down on {HOST.NAME}**”这个触发器来测试告警。

![img14](https://img-blog.csdnimg.cn/c0b37f0c957d4c7d83445ae68801b59e.png#pic_center)

在 **Configuration - Actions - Trigger actions**中创建告警触发规则，在**Conditions**中关联Host group和触发器。

![img16](https://img-blog.csdnimg.cn/83a09763fc7f4b398bc2996491ab359c.png#pic_center)
![img17](https://img-blog.csdnimg.cn/967abe911c7c47b59e3178e1fa9cbb11.png#pic_center)

在Operations中配置将告警消息邮件发送给前面创建的告警接收用户**dbreader**。

![img18](https://img-blog.csdnimg.cn/04c81de9c7734128a5ef31cb525c89c3.png#pic_center)
![img20](https://img-blog.csdnimg.cn/c49b71d4bd384e6ab8d0bef0e1462585.png#pic_center)


## 告警测试
如果我们人为地停掉被监控主机上的MySQL服务：
```bash
systemctl stop mysqld
systemctl start mysqld
```

稍等片刻后应该能够在Zabbix Web首页看到MySQL服务宕机的告警信息，同时配置的告警接收邮件也会受到告警发生和恢复的消息。

![img21](https://img-blog.csdnimg.cn/489942a0c6664f86b0c5aefea187a0b9.png#pic_center)
![img22](https://img-blog.csdnimg.cn/24259a49656347eea7b9c3162660c3ba.jpeg#pic_center)
![img23](https://img-blog.csdnimg.cn/630d09c386734bb6ad3cf966bbc9eaf6.jpeg#pic_center)






