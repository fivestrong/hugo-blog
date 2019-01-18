---
title: "Zabbix Unreachable Hosts"
date: 2019-01-18T11:24:43+08:00
tags: ["zabbix"]
categories: ["linux"]
draft: false
---
解决zabbix 频繁报错主机不可用
<!--more-->
## 遇到的问题

最近几天有一台zabbix监控主机出现时断时续的主机不可达,具体邮件报警为：
```doc
Trigger: Zabbix agent on myhost is unreachable for 5 minutes
Trigger status: PROBLEM
```
server端日志报警如下：
```doc
2019-01-18T03:11:20.287075137Z    144:20190118:111120.286 resuming Zabbix agent checks on host "PublicDns1": connection restored
2019-01-18T03:11:35.323093142Z    144:20190118:111135.322 Zabbix agent item "net.dns[x.x.x.x,www.xxx.com,A,1,3]" on host "PublicDns1" failed: first network error, wait for 15 seconds
2019-01-18T03:12:05.357013294Z    144:20190118:111205.356 Zabbix agent item "net.dns[x.x.x.x,www.xxx.com,A,1,3]" on host "PublicDns1" failed: another network error, wait for 15 seconds
2019-01-18T03:12:35.396894778Z    144:20190118:111235.396 temporarily disabling Zabbix agent checks on host "PublicDns1": host unavailable
2019-01-18T03:13:35.449995264Z    144:20190118:111335.449 enabling Zabbix agent checks on host "PublicDns1": host became available
```
为了排查出现这种现象的原因，我重新配置了agent端的主动检测模式和被动检测模式，分别排除查看是否是检测模式造成的原因。

zabbix web 时断时续显示错误为 "Get value from agent failed. Error: ZBX_TCP_READ() failed", 我以为是网络问题，于是分别检测了丢包率，server端10051，agent 10050 端口开放情况，以及通过zabbix_get 检测server端是否可以向agent发送请求，结果都没问题。

后来查看了[官网文档](https://www.zabbix.com/documentation/4.0/manual/appendix/items/unreachability)
```doc
A host is treated as unreachable after a failed agent check (network error, timeout).

…

After the UnreachablePeriod ends and the host has not reappeared, the host is treated as unavailable.
```

翻译成人话就是：当一条命令检测失败，整个主机就会被认为是"unavailable"状态，并且将不在被监控。

通过server端日志我们可以看到 'Zabbix agent item "net.dns[x.x.x.x,www.xxx.com,A,1,3]"'这条命令有两次network error。

在这条命令检测成功之前，主机一直会被判定为"host unavailable"。然后看日志，一分钟之后主机状态成了"host became available",这时监控又生效了,
这也就造成了zabbix一直报警。

## 解决方案
1. 关闭这个监控项目，不再使用
2. 调整命令参数，排查出现该现象的原因。