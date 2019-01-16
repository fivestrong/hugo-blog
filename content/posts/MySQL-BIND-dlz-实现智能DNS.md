---
title: MySQL+BIND-dlz 实现智能DNS
date: 2017-03-17 13:12:03
draft: true
---
### 配置环境
```
系统：centos 6.8
Mysql: 5.7
BIND: 9.11.0 
```
### CentOS6编译环境安装
```
yum groupinstall "Development Tools"
yum install openssl-devel
```

### mysql 安装
```
这里直接使用官方yum源安装
1. 找对应系统版本的rpm包，https://dev.mysql.com/downloads/repo/yum/
2. sudo yum localinstall mysql57-community-release-el6-{version-number}.noarch.rpm
3. 查看开启的mysql是哪个版本的yum repolist enabled | grep "mysql.*-community.*"
4. 官方默认5.7直接安装　sudo yum install mysql-community-server mysql-community-devel
5. 启动　sudo service mysqld start
6. 查看状态　sudo service mysqld status
7. 找到临时密码　sudo grep 'temporary password' /var/log/mysqld.log
8. 登录并修改密码 mysql -uroot -p  
   ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass4!';
其他需求查看官网教程:https://dev.mysql.com/doc/refman/5.7/en/linux-installation-yum-repo.html
```
### Bind编译安装dlz插件
下载:  <https://www.isc.org/downloads/>
```
1. 添加用户,和编译安装bind 
# tar xvf bind-9.11.0-P1.tar.gz
# cd bind-9.11.0-P1
# groupadd -r named
# useradd -s /sbin/nologin -M -r -g named named
# ./configure --with-dlz-mysql --enable-largefile --enable-threads=yes --prefix=/usr/local/bind --with-openssl
# make -j 4
# make install 
注: 这里的--enable-threds一般建议为no,dlz开启mysql多线程会崩溃，我为了测试所以编译时开了多线程，结果不行.
再注:后面有开启多线程的方法，所以推荐开启多线程。
2. 这里编译引用libmysqlclient.so可能会报错，我这里该库文件所在位置为/usr/lib64/mysql/libmysqlclient.so 需要在/usr/lib/下做个软链接
＃ln -s /usr/lib64/mysql/libmysqlclient.so /usr/lib/libmysqlclient.so
3. 配置bind 环境变量
# chown -R named:named /usr/local/bind 
# echo "export PATH=${PATH}:/usr/local/bind/sbin/:/usr/local/bind/bin/" >> /etc/profile
# source /etc/profile
4. 配置rndc  配置named.conf
# cd /usr/local/bind/etc/
# rndc-confgen -r /dev/urandom >rndc.conf
# 添加其他配置
#  options {
        directory "/var/named/";
        recursion yes;
        listen-on port 53    { any; };
        dump-file "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        allow-query { any; };
        blackhole { none; };
 };
# mkdir /var/named/
# wget -O /var/named/named.ca  http://www.internic.net/domain/named.root
# chown -R named:named /var/named/
5. 启动named,查看根递归解析域名是否成功
＃named -u named -n 1 -c /usr/local/bind/etc/named.conf
＃dig www.baidu.com @127.0.0.1
如果这一步成功的话，一个基本的dns就搭建成功了。
```
### 配置dlz数据库查询
```
1. 创建单独的数据库
＃ mysql -uroot -p
# > create database bind;
2. 建表
＃> CREATE TABLE IF NOT EXISTS `dns_records` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `zone` varchar(255) NOT NULL,
  `host` varchar(255) NOT NULL DEFAULT '@',
  `type` enum('A','MX','CNAME','NS','SOA','PTR','TXT','AAAA','SVR','URL') NOT NULL,
  `data` varchar(255) DEFAULT NULL,
  `ttl` int(11) NOT NULL DEFAULT '3600',
  `mx_priority` int(11) DEFAULT NULL,
  `view`  enum('any', 'Telecom', 'Unicom', 'CMCC', 'ours') NOT NULL  DEFAULT "any" ,
  `priority` tinyint UNSIGNED NOT NULL DEFAULT '255',
  `refresh` int(11) NOT NULL DEFAULT '28800',
  `retry` int(11) NOT NULL DEFAULT '14400',
  `expire` int(11) NOT NULL DEFAULT '86400',
  `minimum` int(11) NOT NULL DEFAULT '86400',
  `serial` bigint(20) NOT NULL DEFAULT '2015050917',
  `resp_person` varchar(64) NOT NULL DEFAULT 'ddns.net',
  `primary_ns` varchar(64) NOT NULL DEFAULT 'ns.ddns.net.',
  PRIMARY KEY (`id`),
  KEY `type` (`type`),
  KEY `host` (`host`),
  KEY `zone` (`zone`)
) ENGINE=MyISAM  DEFAULT CHARSET=utf8 AUTO_INCREMENT=1 ;
view:是区分不同网络区域的字段.
Priority:是区分不同优先级的字段.
3. 创建单独用户,并授权
# grant all privileges on bind.* to named@'%' identified by "named_passwd";
# flush privileges;
4. 创建view, 在named.conf中加入
＃view "ours_domain" {
        match-clients           {127.0.0.1; };
        allow-query-cache           {any; };
        allow-recursion          {any; };
        allow-transfer          {none; };
 
        dlz "Mysql zone" {
               database        "mysql
               {host=localhost dbname=bind ssl=false port=3306 user=named pass=named}
               {select zone from dns_records where zone='$zone$'}
               {select ttl, type, mx_priority, case when lower(type)='txt' then concat('\"', data, '\"') when lower(type) = 'soa' then concat_ws(' ', data, resp_person, serial, refresh, retry, expire, minimum) else data end from dns_records where zone = '$zone$' and host = '$record$'}"; 
        };
        zone "."  IN {
            type hint;
            file "named.ca";
        };
 
};
5. 插入数据
> insert into named.dns_records (zone, host, type, data, ttl) VALUES ('test.info', 'www', 'A', '1.1.1.1', '60');
> insert into named.dns_records (zone, host, type, data, ttl) VALUES ('test.info', 'mail', 'CNAME', 'www', '60');
> insert into named.dns_records (zone, host, type, data, ttl) VALUES ('test.info', '@', 'NS', 'ns', '60');
> insert into named.dns_records (zone, host, type, data, ttl) VALUES ('test.info', 'ns', 'A', '127.0.0.1', '60');
６.测试结果
# dig  @127.0.0.1
# dig mail.test.info @127.0.0.1
# dig -t NS test.info @127.0.0.1 
# dig -t ANY test.info @127.0.0.1
```
### 启动脚本
```sh
#!/bin/bash
# named a network name service.
# chkconfig: 345 87 75
# description: a name server

[ -r /etc/rc.d/init.d/functions ] && . /etc/rc.d/init.d/functions

Builddir=/usr/local/bind
PidFile=/usr/local/bind/var/run/named/named.pid
LockFile=/var/lock/subsys/named
Sbindir=${Builddir}/sbin
Configfile=${Builddir}/etc/named.conf
CheckConf=${Builddir}/sbin/named-checkconf
named=named

if [ ! -f ${Configfile} ]
then
    echo "Can't find named.conf "
    exit 1
fi

if [ ! -d /var/run/named/ ]
then
    echo "could not open directory '/var/run/named/': Permission denied "
    exit 1
elif [ ! -w /var/run/named/ ]
    then
        echo "could not open directory '/var/run/named/': Permission denied "
        exit 1
fi


if [ ! -r ${Configfile} ]
then
    echo "Error: ${Configfile} is not readfile!"
    exit 1
else
    $CheckConf
    if [ $? != 0 ]
    then
        echo -e "Please check config file in \033[31m${Configfile} \033[0m!"
        exit 2
    fi
fi

start() {
    [ -x ${Builddir}/sbin/$named ] ||   exit 4
    if [ -f $LockFile ]; then
        echo -n "$named is already running..."
        echo_failure
        echo
        exit 5
    fi

    echo -n "Starting $named: "
    daemon --pidfile "$PidFile" ${Sbindir}/$named -u named -n 1 -c ${Configfile}
    RETVAL=$?
    echo
    if [ $RETVAL -eq 0 ]; then
        touch $LockFile
        return 0
    else
        rm -f $LockFile $PidFile
        return 1
    fi
}

stop() {
    if [ ! -f $LockFile ];then
        echo "$named is not started."
        echo_failure
    fi

    echo -n "Stopping $named: "
    killproc $named
    RETVAL=$?
    echo
    [ $RETVAL -eq 0 ] && rm -f $LockFile
    return 0
}

restart() {
    stop
    sleep 1
    start
}

reload() {
    echo -n "Reloading $named: "
    killproc $named -HUP
    RETVAL=$?
    echo
    return $RETVAL
}


status() {
    if pidof $named > /dev/null && [ -f $PidFile ]; then
        echo "$named is running..."
    else
        echo "$named is stopped..."
    fi
}

case $1 in
start)
    start ;;
stop)
    stop ;;
restart)
    restart ;;
reload)
    reload ;;
status)
    status ;;
*)
    echo "Usage:named {start|stop|status|reload|restart}"
    exit 2;;
esac

```
## 关于dlz默认开启多线程，Mysql崩溃问题
```
默认编译开启多线程支持后,每次启动后，查询一次，dlz对mysql的接口就会崩溃。
其实官方已经解决了这个问题，https://source.isc.org/cgi-bin/gitweb.cgi?p=bind9.git;a=commit;h=5ba1d3dcc5739a1f77ec2875b276b163a42ef1e8

首先在源码包下面找到bind-9.11.0-P1/contrib/dlz/modules/mysql　目录
里面有dlz_mysql_dynamic.c 源码执行make 编译一下,可能会报错，还是之前mysql库文件那个，如果之前设置过应该没问题。
在contrib/dlz/modules/mysql/testing 下面有给出的配置文件和数据库文件，按照READEME导入测试即可。
其中最重要的是加一句 database "dlopen ../dlz_mysql_dynamic.so
这个.so就是刚才编译出来的库文件，把它放在你的配置文件目录下载，加载之后就可以了。以前的配置可以不变，即可支持多线程。
终于不崩溃了。其实数据库也可以使用postgresql,这个亲测dlz支持多线程,也不崩溃。
```
