---
title: NFS服务配置
date: 2015-11-02 21:43:56
draft: true
---
#### 一、NFS的相关概念
  NFS（Network File System）即网络文件系统的缩写，由Sun公司研发，其目的是为了解决网络文件共享的问题。用户可以实现像挂载本地文件系统一样挂载NFS服务器的共享目录；其具有配置简单、使用高效的特点，但只能在Linux系统使用，不能跨平台使用。

  NFS服务占用2049端口，但其对于不同的功能使用小于1024的随机端口来传输数据，但如果是随机端口客户端如何知晓要访问哪个端口呢？这就要借助于RPC协议了。

RPC（Remote Procedure Call)即远程过程调用，其作用是向客户端告知NFS的端口信息；NFS服务启动时会主动向RPC注册所使用的端口，而RPC使用111端口来响应客户端的请求，所以客户端可以借助于RPC来完成NFS的访问。
#### 二、NFS文件访问权限

NFS服务本身没有身份验证的功能，权限是遵循共享目录在NFS服务器上的权限设置，而且只识别UID和GID。假如现在有一个共享的目录share其属主、属组及权限信息如下：
![](http://upload-images.jianshu.io/upload_images/452132-e1cfb4add111a4d0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
用户和组为mysql，但当客户端访问此目录时，NFS是判定访问者的uid是否为400，如果uid相符，那么访问者就有可能拥有与mysql用户相同的权限，这还要取决于/share设置共享时所分配的权限；如果访问者的uid对应了NFS服务器上的另一个用户，则访问者就对应拥有other权限，但是否能够完全对应用other权限也要取决于\share的共享权限；如果访问者的uid恰好在NFS服务器上不存在，则服务器用自动将其压缩成为匿名用户，其uid为65534，而CentOS将其显示为nfsnobody。

由于在绝大部分Linux系统中root用户的uid为0，也就是说客户端可以轻易的获得NFS的root权限来访问共享目录，这样是极不安全的，所以NFS默认会将root的身份压缩成匿名用户。
#### 三、NFS服务端的配置
1、安装NFS服务
![](http://upload-images.jianshu.io/upload_images/452132-772d40315b69554d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    在安装nfs-utils的同时会安装rpcbind程序。
![](http://upload-images.jianshu.io/upload_images/452132-a3c7d365ff7884f4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
配置NFS服务
    2、 NFS服务使用/etc/exports配置文件进行设置，其语法格式如下：
![](http://upload-images.jianshu.io/upload_images/452132-a43ac94681e08af7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
/share：表示共享的文件系统；       
192.168.2.0/24：允许连接共享文件系统的客户端；
(rw)：客户端对于共享文件系统所拥有的权限；
*.test.com(ro)：表示test.com为后缀的主机都可以对/share目录有只读的权限；
### 3、客户端的设置方式：
（1）IP地址，如192.168.2.10；
（2）网络地址，如192.168.2.0/24，或192.168.2.0/255.255.255.0；
（3）主机名，如client.test.com，也可以使用通配符，“*”或“？”。
    常用权限参数：
        rw：可读可写；
        ro：只读；
        root_squash：将root用户压缩成为匿名用户（默认选项）；
        no_root_squash：访问共享目录时保持root用户身份；
        all_squash：将所有访问NFS的用户身份全部压缩成为匿名用户；
        sync：将数据同步写入到内存和硬盘中；
        async：将数据暂存于内存中。
        anonuid：指定匿名访问用户的UID；
        anongid：指定匿名访问用户组的GID。
        更多的参数可自行man exports来进行查阅。
#### 四、启动NFS服务
先要启动rpcbind：service rpcbind start
再启动nfs: service nfs start
![](http://upload-images.jianshu.io/upload_images/452132-8de1109e1d373c96.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)    查看NFS开启的端口信息：
![](http://upload-images.jianshu.io/upload_images/452132-1eaa0f590f05c270.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](http://upload-images.jianshu.io/upload_images/452132-268a310732b48d3e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
NFS服务本身启动在2049端口，rpcbind启动在111端口。
可以使用rpcinfo命令来查看rpc的相关信息，其格式如下：
rpc [option] [IP|hostname]
option:   
-p：显示所有的port与program信息。
![](http://upload-images.jianshu.io/upload_images/452132-7c3a6795c70d85cf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
#### 五、NFS的查看命令
    下面来介绍两个经常用到的查看命令。
    （1）showmount命令
        格式：showmount [option] [IP|hostname]
            option:
                -a：显示当前主机与客户端的NFS连接共享的状态；
                -e：显示某台主机的/etc/exports所共享的目录信息。
![](http://upload-images.jianshu.io/upload_images/452132-55587723e87e3d1f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

   （2）exportfs命令

      格式：exportfs [option]
      option:
                -a：全部挂载（或卸载）/etc/exports文件中的设置；
                -r：重新挂载/etc/exports中的设置；
                -u：卸载某一目录；
                -v：将命令输出显示到屏幕。
![](http://upload-images.jianshu.io/upload_images/452132-c154bba7b1e337ca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](http://upload-images.jianshu.io/upload_images/452132-6c29606b956eee3d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
#### 六、NFS客户端设置
六、NFS客户端设置
    （1）手动挂载NFS共享目录
![](http://upload-images.jianshu.io/upload_images/452132-13bdc120d8e6205b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    （2）开机自动挂载NFS共享目录
        1）/etc/fstab
![](http://upload-images.jianshu.io/upload_images/452132-b2bf05539a6cd1a0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 _netdev：此选项表示在NFS服务器宕机时，也不会影响本地系统的启动。 
       2）/etc/rc.d/rc.local
![](http://upload-images.jianshu.io/upload_images/452132-9dc865efa91210ca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
      3) 使用autofs
使用autofs可以实现按需挂载，当用户访问共享目录时，目录才会被自动挂载上，过一段时间没有使用又会被自动卸载。
            安装autofs服务：
![](http://upload-images.jianshu.io/upload_images/452132-55b5688b419d4326.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
            配置主配置文件/etc/auto.master：
![](http://upload-images.jianshu.io/upload_images/452132-4d0677a546a9fec4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
            定义/etc/auto.nfs文件，此文件中指时挂载信息即可：
![](http://upload-images.jianshu.io/upload_images/452132-68b248d334a6c689.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
        注意本地的挂载目录/auto/nfs不需要事先建立，autofs会自动建立。
        启动autofs服务：
![](http://upload-images.jianshu.io/upload_images/452132-0ff9ca2ef9ecb6ca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
            验证：直接切换到/auto/nfs目录中；
![](http://upload-images.jianshu.io/upload_images/452132-f1319f82b3621d4b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 安装步骤总结
### NFS服务端配置步骤：
1. 安装软件
yum install nfs-utils rpcbind -y 
2. 启动服务（注意先后顺序）
/etc/init.d/rpcbind start
rpcinfo -p localhost 
/etc/init.d/nfs start 
rpcinfo -p localhost 
3. 设置开机自启动
chkconfig nfs on
chkconfig rpcbind on

```
 /*企业中一般建议将启动写入/etc/rc.local中*/
/#!/bin/sh
/#
/# This script will be executed *after* all the other init scripts.
/# You can put your own initialization stuff in here if you don't
/# want to do the full Sys V style init stuff.
touch /var/lock/subsys/local
/etc/init.d/rpcbind start
/etc/init.d/nfs start

/*然后将chkconfig关闭*/
chkconfig nfs off
chkconfig rpcbind off 
```

4     . 配置nfs服务
```
echo "/data 192.168.230.*(rw,sync,all_squash)" >> /etc/exports //该文件默认为空
mkdir -p /data
chown -R nfsnobody.nfsnobody /data 
//查看nfs默认使用的用户以及共享的参数  cat /var/lib/nfs/etab
```
5      . 重新加载服务（平滑重启）
```
/etc/init.d/nfs reload  等价于 /usr/sbin/exportfs -r 
```
6      . 检查或测试挂载
```
showmount -e localhost 
mount -t nfs 127.0.0.1:/data /mnt
```

### NFS客户端配置：
1. 安装软件
    yum install nfs-utils rpcbind -y 
2. 启动rpcbind
    /etc/init.d/rpcbind start 
3. 配置开机自启动
    chkconfig rpcbind on 
    或	写入/etc/rc.local  /etc/init.d/rpcbind start  
4. 测试服务端共享情况
    showmount -e server_ip (yum install nfs-utils -y 才有此命令)
5. 挂载
    mkdir -p /data 
	mount -t nfs server_ip:/data  /data 
6. 测试读写
7. 开机自动挂载
   vim /etc/rc.local
   /etc/init.d/rpcbind start 
   /bin/mount -t nfs server_ip:/data  /data 
   
### NFS 排错
1. 前提：NFS原理以及部署的步骤很熟练
2. 先在客户端排查
   ping server_ip
   telnet server_ip 111
   showmount -e server_ip
NFS遇到的问题
1.服务端没有关闭防火墙导致客户端无法连接
  /etc/init.d/iptables stop 
  chkconfig iptables off 
2. 注意服务启动的顺序，先启动rpcbind,再启动nfs
3. 查看具体配置文件帮助  man exports

### 企业生产场景NFS共享存储优化小结


1. 硬件：sas/ssd 磁盘，买多块，raid0/raid10。网卡吞吐量要大，至少千兆多弄几块进行bond绑定。
2. NFS服务器端配置：
/data 192.168.230.*(rw,sync,all_squash,anonuid=65534,anongid=65534)
3. NFS客户端挂载：
mount -t nfs -o nosuid,noexec,nodev,noatime,nodiratime,rsize=131072,wsize=131072 192.168.230.132:/data /data 

4.有关NFS服务器的所有服务器内核优化：
```
   cat >> /etc/sysctl.conf <<EOF	
   net.core.wmen_default = 8388608
   net.core.rmen_default = 8388608
   net.core.rmen_max = 16777216
   net.core.wmen_max = 16777216
   EOF
   执行sysctl -p 生效
```


5. 如果卸载的时候提示：

```
umount:/mnt:device is busy 
```

需要退出挂载目录在进行卸载或者是NFS Server宕机了需要强制卸载 

```
mount -lf /mnt 
```

6. 大型网站NFS网络文件系统代替软件：
分布式文件系统Moosefs(mfs),glusterfs,FastDFS