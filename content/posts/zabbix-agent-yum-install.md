---
title: 'zabbix-agent yum install '
date: 2017-10-17 08:39:53
draft: true
---
Step 1 – Add Required Repository
```shell
CentOS/RHEL 7:
rpm -Uvh http://repo.zabbix.com/zabbix/3.4/rhel/7/x86_64/zabbix-release-3.4-2.el7.noarch.rpm

CentOS/RHEL 6:
rpm -Uvh http://repo.zabbix.com/zabbix/3.4/rhel/6/x86_64/zabbix-release-3.4-1.el6.noarch.rpm
```

Step 2 – Install Zabbix Agent
```shell
yum install zabbix-agent
```
Step 3 – Edit Zabbix Agent Configuration
```shell

# vim /etc/zabbix/zabbix_agentd.conf

#Server=[zabbix server ip]
#Hostname=[ Hostname of client system ]

Server=192.168.1.11
Hostname=Server1
```

Step 4 – Restarting Zabbix Agent
```shell
# systemctl restart zabbix-agent
```
Step 5 – open the firewall port 
```shell
firewall-cmd --permanent --zone=public --add-port=10050/tcp
firewall-cmd --reload
firewall-cmd --list-all
```