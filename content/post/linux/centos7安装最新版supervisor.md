---
title: "Centos7安装最新版supervisor"
date: 2019-10-11T17:27:19+08:00
tags: ["supervisor"]
categories: ["linux"]

---

由于centos7 yum安装supervisor版本太老，所以需要自己通过pip安装最新版本的supervisor.
<!--more-->

### 1.安装supervisor
centos7 系统yum安装的supervisor是3.*.*版本，需要使用python的pip安装最新版本
```shell
pip3 install supervisor
```
由于本机python是编译安装，所以supervisor安装后所在目录为:
```shell
/usr/local/python3/bin/supervisorctl  (客户端（用于和守护进程通信，发送管理进程的指令）)
/usr/local/python3/bin/supervisord  (是supervisor的守护进程服务（用于接收进程管理命令))
/usr/local/python3/bin/echo_supervisord_conf (生成初始配置文件程序)
```
添加路径到环境变量
```shell
将 PATH=$PATH:$HOME/bin:/usr/local/python3/bin
添加到用户目录文件 .bash_profile 里面, 其他能够执行环境变量初始化的文件也可以。
```

### 2.配置supervisor
生成配置文件:
```shell
mkdir -p /etc/supervisor/conf.d/
echo_supervisord_conf > /etc/supervisor/supervisord.conf
```
在/etc/supervisor/supervisord.conf 的[include]下添加:
```shell
[include]
files = /etc/supervisor/conf.d/*.conf
```
修改 /etc/supervisor/supervisord.conf 配置文件路径/tmp为其他，防止pid sock文件被系统清除。
```shell
logfile=/var/log/supervisord.log
file=/var/run/supervisor.sock
pidfile=/var/run/supervisord.pid
serverurl=unix:///var/run/supervisor.sock
```

### 3.配置开机自启动

1. 官方脚本

```shell
wget -O /usr/lib/systemd/system/supervisord.service  https://github.com/Supervisor/initscripts/raw/master/centos-systemd-etcs
```

2. 手动编写

创建 /usr/lib/systemd/system/supervisord.service, 内容根据自己安装程序的位置进行修改。
```
# supervisord service for systemd (CentOS 7.0+)
[Unit]
Description=Supervisor daemon
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/python3/bin/supervisord -c /etc/supervisor/supervisord.conf
ExecStop=/usr/local/python3/bin/supervisorctl $OPTIONS shutdown
ExecReload=/usr/local/python3/bin/supervisorctl $OPTIONS reload
KillMode=process
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target
```
设置开机子启动
```
systemctl enable supervisord.service
systemctl daemon-reload 
systemctl list-unit-files|grep enabled #查看是否开启成功

chmod 766 supervisord.service # 修改文件权限
```

### 4. 编辑监控程序文件
在/etc/supervisor/conf.d 下新建程序文件，比如cmd.conf
```text
[program:somecmd]
command=/usr/local/python3/bin/python cmd.py
numprocs=1
autostart=true
autorestart=true
startretries=3
user=nfuser
redirect_stderr=true
stdout_logfile=/var/log/cmd-stdout.log
stdout_logfile_maxbytes=10MB
stdout_logfile_backups=20
```

如果有多个程序，可以配置为组:
```shell
; The below sample group section shows all possible group values,
; create one or more 'real' group: sections to create "heterogeneous"
; process groups.

;[group:thegroupname]
;programs=php-fpm,nginx,redis,mysql  ; 定义该组有哪些程序，程序之间用逗号分隔，注意，这些程序必须在前面用“[program:theprogramname]”模块定义过。
;priority=999                        ; 启动优先级，假设有多个组，每个组一个优先级，越小越优先执行 (默认 999)
```

### 5. 启动supervisor

1. 手动启动supervisor
```shell
supervisord -c /etc/supervisor/supervisord.conf
ps -ef | grep supervisord # 查看服务是否已启动
cat /var/log/cmd-stdout.log # 通过日志查看
```

2. 通过systemd命令启动
```shell
systemctl status supervisord # 查看supervisord状态
systemctl start supervisord
systemctl restart supervisord
systemctl stop supervisord
```

3. 修改cmd配置文件后重新载入
```shell
supervisorctl reread
supervisorctl update
supervisorctl restart somecmd
```

4. 修改/etc/supervisor/supervisord.conf后重新载入
```shell
supervisorctl reload
```

5. 一些其他可能用到的命令
```shell
supervisorctl start programname      启动某个进程  
supervisorctl stop programname       停止某个进程
supervisorctl restart programname    重启某个进程
supervisorctl start all     启动全部进程
supervisorctl stop all      停止全部进程，注：start、restart、stop都不会载入最新的配置文件。
supervisorctl status        查看状态
supervisorctl shutdown      关闭supervisor
```

