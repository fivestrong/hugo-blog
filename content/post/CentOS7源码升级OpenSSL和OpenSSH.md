---
title: CentOS7源码升级OpenSSL和OpenSSH
date: 2019-03-13T14:15:46+08:00
tags: ["openssl", "openssh"]
categories: ["linux"]
---
CentOS7源码升级OpenSSL和OpenSSH
<!--more-->
### 一、安装telnet
这里启用telnet的原因是：

如果通过ssh远程连接服务器后进行的版本升级操作，如果出现意外可能连接会断开并且无法再远程登录上去，如果服务器安装了iDRAC远程管理卡还有救，如果没有iDRAC远程管理卡，则需要提前开启telnet远程登录或是到机房现场进行升级操作比较妥当。


1、查看系统版本
```shell
    cat /etc/centos-release 
    # 临时关闭selinux
    getenforce
    setenforce 0
```
2、安装telnet
```shell
    yum install telnet-server
    # 如果是centos6 需要修改配置
    vim /etc/xinetd.d/telnet
        # default: on
        # description: The telnet server serves telnet sessions; it uses \
        #       unencrypted username/password pairs for authentication.
        service telnet
        {
                flags           = REUSE
                socket_type     = stream
                wait            = no
                user            = root
                server          = /usr/sbin/in.telnetd
                log_on_failure  += USERID
                disable         = no    # 将默认的yes改为no , 否则telnet启动后，23端口就会起不来
        }
    /etc/init.d/xinetd restart
    netstat -antup|grep 23
    # 如果是centos7
    systemctl start telnet.socket
    systemctl status telnet.socket
    ss -tnlp|grep 23
```
3、添加防火墙放行规则
```shell
    # centos6 最好限制一下允许telnet登录的ip
    iptables -I INPUT -p tcp -s 192.168.1.1 --dport 23 -j ACCEPT
    # centos7 
    firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.56.0/24" service name="telnet" log prefix="telnet" level="info" limit value="1/m" accept'
```
4、测试是否可以登录
```shell
    telnet -l username 192.168.1.1
    # 最好在成功升级以后关闭telnet
```
### 二、升级OpenSSL

1、查看ssl版本及下载相关依赖包
```shell
　　openssl version -a
　　yum install -y gcc openssl-devel pam-devel rpm-build
```
2、下载安装包（查询最新安装包）
```shell
   mkdir downloads && cd downloads
　　wget https://distfiles.macports.org/openssl/openssl-1.0.2r.tar.gz
```
3、卸载当前openssl
```shell
　　rpm -qa | grep openssl
　　rpm -qa |grep openssl|xargs -i rpm -e --nodeps {}
```
4、解压openssl源码并编译安装
```shell
　　tar -xzvf openssl-1.0.2r.tar.gz
　　cd openssl-1.0.2r
　　./config --prefix=/usr --openssldir=/etc/ssl --shared zlib
    make 
    make install
```
5、创建库文件软链接并查看版本
　　由于OpenSSL不提供libcrypto.so.10和libssl.so.10这两个库，而yum、wget等工具又依赖此库，需要创建软连接使用
```shell
    cd /usr/lib64/
    ll /usr/lib64/libssl.so*
    ll /usr/lib64/libcrypto.so*
    ln -s /usr/lib64/libssl.so.1.0.0  libssl.so.10
    ln -s /usr/lib64/libcrypto.so.1.0.0  libcrypto.so.10
    openssl version -a
```

### 三、升级OpenSSH
1、查看当前ssh版本
```shell
　　ssh -V
```
2、下载安装包（查询最新安装包）
```shell
    cd downloads
    wget http://ftp.openbsd.org/pub/OpenBSD/OpenSSH/portable/openssh-7.9p1.tar.gz
```
3、卸载原Openssh
```shell
　　rpm -qa |grep openssh
　　for i in `rpm -qa |grep openssh`;do rpm -e $i --nodeps;done
　　rm -rf /etc/ssh  # 删除操作请谨慎，最好提前看一下
```
4、解压openssh安装包
```shell
　　tar -zxvf openssh-7.9p1.tar.gz
　　cd openssh-7.9p1
```
5、编译安装
```shell
    ./configure --prefix=/usr --sysconfdir=/etc/ssh --with-md5-passwords --with-pam --with-tcp-wrappers --without-hardening --with-zlib
    make
    make install
```
6、安装完成，执行配置
```shell 
    rm -rf /etc/init.d/sshd
    cp contrib/redhat/sshd.init /etc/init.d/sshd
    chkconfig --add sshd
    chkconfig --list|grep sshd
    echo "PermitRootLogin yes" >> /etc/ssh/sshd_config # 一般不推荐
    systemctl enable sshd
    systemctl restart sshd
    systemctl status sshd
    ssh -V
```