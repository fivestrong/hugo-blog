---
title: CentOS7 安装配置VNC
date: 2017-08-31 13:27:11
---
### 1. 安装图形化界面
```
# yum check-update
# yum groupinstall "X Window System"
# yum install gnome-classic-session gnome-terminal nautilus-open-terminal control-center liberation-mono-fonts
```
或者直接
```
yum groupinstall "GNOME Desktop" "Graphical Administration Tools"
```
```
### 设置默认启动图形界面
# unlink /etc/systemd/system/default.target
# ln -sf /lib/systemd/system/graphical.target /etc/systemd/system/default.target
# reboot
```
### 2.安装 tigervnc server and X11 fonts
```
yum install tigervnc-server xorg-x11-fonts-Type1
```
拷贝VNC server configuration 文件到 /etc/systemd/system 下进行配置。VNC默认端口为5900，你可以直接通过5900端口登录VNC，也可以自己设置一个子端口。比如，我将子端口设成5，那么登录的端口就是5905,你可以通过ipaddress:sub-port(192.168.1.1:5或者192.168.1.1:5905)来登录。
```
# cp /lib/systemd/system/vncserver@.service /etc/systemd/system/vncserver@:5.service
# vi /etc/systemd/system/vncserver@:5.service
```
默认配置文件的内容
```
[Unit]
Description=Remote desktop service (VNC)
After=syslog.target network.target
[Service]
Type=forking
# Clean any existing files in /tmp/.X11-unix environment
ExecStartPre=/bin/sh -c '/usr/bin/vncserver -kill %i > /dev/null 2>&1 || :'
ExecStart=/sbin/runuser -l <USER> -c "/usr/bin/vncserver %i"
PIDFile=/home/<USER>/.vnc/%H%i.pid
ExecStop=/bin/sh -c '/usr/bin/vncserver -kill %i > /dev/null 2>&1 || :'
[Install]
WantedBy=multi-user.target
```
将<USER>替换成你的用户名字，例如 archie
```
[Unit]
Description=Remote desktop service (VNC)
After=syslog.target network.target
[Service]
Type=forking
# Clean any existing files in /tmp/.X11-unix environment
ExecStartPre=/bin/sh -c '/usr/bin/vncserver -kill %i > /dev/null 2>&1 || :'
ExecStart=/sbin/runuser -l archie-c "/usr/bin/vncserver %i"
PIDFile=/home/archie/.vnc/%H%i.pid
ExecStop=/bin/sh -c '/usr/bin/vncserver -kill %i > /dev/null 2>&1 || :'
[Install]
WantedBy=multi-user.target
```
添加防火墙规则
```
# firewall-cmd --permanent --zone=public --add-port=5905/tcp
# firewall-cmd --reload
```
### 3. 配置启动vncserver
切换到archie用户, 启动vncserver 并配置密码
```
[archlie@server ~]$ vncserver
You will require a password to access your desktops.

Password:
Verify:
xauth:  file /home/archie/.Xauthority does not exist

New 'localhost.localdomain:1 (raj)' desktop is server.itzgeek.com:1

Creating default startup script /home/archie/.vnc/xstartup
Starting applications specified in /home/archie/.vnc/xstartup
Log file is /home/archie/.vnc/server.itzgeek.com:1.log

```
切换到root用户，重启systemctl daemon,启动vncserver
```bash
systemctl daemon-reload
systemctl start vncserver@:5.service
systemctl enable vncserver@:5.service
```
然后通过ipaddress:5905 or ipaddress:5登录远程服务器


