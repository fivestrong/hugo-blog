---
title: zabbix 3 配置邮件报警
date: 2017-04-05 15:58:50
draft: true
---
zabbix 配置邮件报警,其实网上有一堆教程，但是在按着他们的教程配置好了之后，或多或少有些问题，比如邮件发不出去，没内容等等。
我整理了一下自己配置成功之后需要注意的几点。
### 1. 安装mailx 服务 通过mailx 配置好外部SMTP服务器相关信息发送邮件（这里是配置发信的地址）：
```
yum -y install mailx
vim /etc/mail.rc   增加以下内容：
set bsdcompat
set from=test@163.com smtp=smtp.163.com   #这里是邮局服务器和SMTP 服务器信息，这里使用163的，其他邮箱自行修改一下
set smtp-auth-user=test@163.com smtp-auth-password=yourpassword  #smtp-auth-user 自然是指邮局用户，需要写完整地址，然后是密码
set smtp-auth=login
```
```
使用命令行测试一下是否配置成功
echo “zabbix test mail” | mail -s “zabbix” test@163.com

```
### 2.创建示警媒介
1. 进入 【管理】-【示警媒介类型】-【创建媒体类型】
　　注意我们选择使用脚本方式，名称可自定义，脚本名称设定需要和以后创建的脚本相同，这里还需要添加参数，否则无法接受到系统传递的信息进行发送：
{ALERT.SENDTO}
{ALERT.SUBJECT}
{ALERT.MESSAGE}
![](http://upload-images.jianshu.io/upload_images/452132-adaced77dbb85ca2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](http://upload-images.jianshu.io/upload_images/452132-913648e8f64171d9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2.创建用户接收邮箱：
　　【管理】-【用户】-选择对应的用户默认Admin -切换到【示警媒介】选项卡-类型处选择为刚才我们创建的示警媒介名称，收件人填写为需要接收邮件的地址
![](https://www.cnyunwei.cc/wp-content/uploads/2016/06/zabbix-3.png)
![](http://upload-images.jianshu.io/upload_images/452132-b1f59e84be10d0c5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 3.创建触发动作：
【组态】-【动作】-【创建动作】
![](http://upload-images.jianshu.io/upload_images/452132-810929d138368a68.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
名称：Action-Email
默认接收人：Problem：{TRIGGER.NAME}
默认信息：
告警主机:{HOSTNAME1}
告警时间:{EVENT.DATE} {EVENT.TIME}
告警等级:{TRIGGER.SEVERITY}
告警信息: {TRIGGER.NAME}
告警项目:{TRIGGER.KEY1}
问题详情:{ITEM.NAME}:{ITEM.VALUE}
当前状态:{TRIGGER.STATUS}:{ITEM.VALUE1}
事件ID:{EVENT.ID}

恢复主旨：Recover：{TRIGGER.NAME}
恢复信息：
告警主机:{HOSTNAME1}
告警时间:{EVENT.DATE} {EVENT.TIME}
告警等级:{TRIGGER.SEVERITY}
告警信息: {TRIGGER.NAME}
告警项目:{TRIGGER.KEY1}
问题详情:{ITEM.NAME}:{ITEM.VALUE}
当前状态:{TRIGGER.STATUS}:{ITEM.VALUE1}
事件ID:{EVENT.ID}
```
## 注意: 这里改了图片里面的默认接收人和恢复主旨，之前的太长邮件显示不全。还有：设置后不要点击【添加】，这里点击更新是无法保存的，切换到【操作】选项卡
添加用户，按照下图设置勾选即可。
![](http://upload-images.jianshu.io/upload_images/452132-4abe654ce75bda08.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### 4.创建邮件发送脚本：
1.先查看一下/usr/local/etc/zabbix 中的AlertScriptsPath 是否已经指定了，这里是指定zabbix程序调用脚本的目录，设置为：
AlertScriptsPath=/usr/local/share/zabbix/alertscripts
2.在该目录/usr/local/share/zabbix/alertscripts 下创建脚本文件

```
vim sendmail.sh
#!/bin/bash
file=/tmp/zabbix_mail.txt
echo "$3" > $file
dos2unix -k $file
/bin/mail -s "$2" $1 < $file
# echo "$3" | mail -s "$2" $1 #如果发送邮件完全是英文的，可以只使用这一条   
:wq 保存退出
设置权限以及所属用户：
chown zabbix.zabbix /usr/local/share/zabbix/alertscripts/sendmail.sh
chmod +x /usr/local/share/zabbix/alertscripts/sendmail.sh
```
```
yum install dos2unix -y

注：使用dos2unix工具是为解决zabbix发送邮件出现乱码和收到的邮件是*.bin的情况。
#$3 代表邮件内容，也就是对应参数{ALERT.MESSAGE}
#$2 代表邮件主题，也就是对应参数{ALERT.SUBJECT}
#$1 代表收件人，也就是对应参数{ALERT.SENDTO}
```
接下来测试一下，看看成不成功，祝好运。

参考文章：

[1:Zabbix使用外部邮箱服务器发送邮件报警](https://yq.aliyun.com/articles/38837)

[2:zabbix 配置邮件报警](https://www.cnyunwei.cc/archives/242)

[3:zabbix 邮件内容为附件](https://www.cnyunwei.cc/archives/249)