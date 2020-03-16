---

title: "Linux 防火墙安全（2）"
date: 2020-03-11T17:06:48+08:00
tags: ["firewall"]
categories: ["linux"]
draft: false
---

## nftables 概述

nftables 引入的新特证：

- 之前不同的网络操作需要安装特定的工具，比如 iptables, ip6tables, ebtables, arptables 等等，现在都被整合到了 nft 这一个工具包中。
- nftables 采用多维树状结构展示规则集，这使得调试数据包匹配了哪些规则变得更加容易。
- 在 iptables 中不管你用不用，默认都建立了 filter, NAT, mangle, security 这些表；而在 nftables 中， 你仅需要创建需要的表，这将拥有更高效的性能。
- nftables 可以在一条规则中指定多个行为，而不用像 iptables 中那样需要为每个行为创建多条规则。
- 添加一条新的规则之后，这条规则会立刻生效，而不用像之前一样需要重新加载整个规则表。
- nftables 有自己内建的脚本引擎，在脚本中调用会更加的高效且易读。
- 对应已有的调用 iptables 的脚本，可以使用相关的工具将它转化成 nftables 格式。

nftables 还在不断的完善中，所以有一些高阶的功能还得基于 iptables 的解决方案，但是随着版本的迭代，功能会越来越完善。所以要时刻留意版本的变更：

```shell
[root@centos8 ~]# nft -v
nftables v0.9.0 (Fearless Fosdick)
```

### 了解nftables 的表和链

- Tables: nftables 中的表指代特定的协议族，拥有类型：ip, ip6, inet, arp, bridge, netdev 。
- Chains： nftables 的链大致相当于 iptables 的表，比如在 nftables 中可以使用 filter, route, 或者 NAT 链。

### nftables 应用

还是以 Ubuntu 系统为例，将 ufw 关闭, 安装 nftables 

```shell
sudo ufw disable
sudo systemctl disable --now ufw
sudo apt install nftables
```

查看 安装的表 ，啥也没有

```shell
sudo nft list tables
```

开启服务

```shell
sudo systemctl start nftables
```



Ubuntu 20.04 中，nftables 的配置文件基本设置为空，需要你手动替换。

我们可以查看一下软件自带的一些样例:

```shell
ubuntu@ubuntu-2004: cd /usr/share/doc/nftables/examples
ubuntu@ubuntu-2004:/usr/share/doc/nftables/examples$ ls -l
total 92
-rw-r--r-- 1 root root 1016 Dec 17 07:49 all-in-one.nft
-rw-r--r-- 1 root root  129 Dec 17 07:49 arp-filter.nft
-rw-r--r-- 1 root root  197 Dec 17 07:49 bridge-filter.nft
-rwxr-xr-x 1 root root 1263 Dec  2 15:00 ct_helpers.nft
-rw-r--r-- 1 root root  187 Dec 17 07:49 inet-filter.nft
-rw-r--r-- 1 root root  251 Dec 17 07:49 inet-nat.nft
-rw-r--r-- 1 root root  182 Dec 17 07:49 ipv4-filter.nft
-rw-r--r-- 1 root root   74 Dec 17 07:49 ipv4-mangle.nft
-rw-r--r-- 1 root root  246 Dec 17 07:49 ipv4-nat.nft
-rw-r--r-- 1 root root  137 Dec 17 07:49 ipv4-raw.nft
-rw-r--r-- 1 root root  186 Dec 17 07:49 ipv6-filter.nft
-rw-r--r-- 1 root root   78 Dec 17 07:49 ipv6-mangle.nft
-rw-r--r-- 1 root root  253 Dec 17 07:49 ipv6-nat.nft
-rw-r--r-- 1 root root  141 Dec 17 07:49 ipv6-raw.nft
-rwxr-xr-x 1 root root 1858 Dec  2 15:00 load_balancing.nft
-rwxr-xr-x 1 root root 1165 Dec  3 08:10 nat.nft
-rw-r--r-- 1 root root  128 Dec 17 07:49 netdev-ingress.nft
-rwxr-xr-x 1 root root 1077 Dec  3 08:10 overview.nft
-rw-r--r-- 1 root root  475 Dec  3 08:10 README
-rwxr-xr-x 1 root root 2413 Dec  2 15:00 secmark.nft
-rwxr-xr-x 1 root root 1278 Dec  2 15:00 sets_and_maps.nft
drwxr-xr-x 2 root root 4096 Mar 11 23:16 sysvinit
-rwxr-xr-x 1 root root  577 Dec  3 08:10 workstation.nft
```

将 `workstation.nft` 替换到 `/etc/nftables.conf` 

```shell
sudo cp workstation.nft /etc/nftables.conf

cat /etc/nftables.conf
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
	chain input {
		type filter hook input priority 0;

		# accept any localhost traffic
		iif lo accept

		# accept traffic originated from us
		ct state established,related accept

		# activate the following line to accept common local services
		#tcp dport { 22, 80, 443 } ct state new accept

		# accept neighbour discovery otherwise IPv6 connectivity breaks.
		ip6 nexthdr icmpv6 icmpv6 type { nd-neighbor-solicit,  nd-router-advert, nd-neighbor-advert } accept

		# count and drop any other traffic
		counter drop
	}
}
```

这个配置文件内容有：

- #! /usr/sbin/nft -f : 类似于 shell 脚本的 #!/bin/sh，使用 nftables 内置的脚本引擎，这样就不需要在每一条需要执行的命令前面再添加 nft 命令
- flush ruleset: 清除所有已经加载的规则。
- table inet filter : 创建一个 inet 协议族的 filter，这将对 IPv4 IPv6都有效。这个表的名字就叫 filter（名称可以自定义）。
- chain input: 定义 input 链（名称可以自定义）
- type filter hook input priority: 定义这个链的类型为 filter ，**hook input** 表示用途是处理入流量的包，由于同时定义了 hook 和 priority ， 这条规则将直接从网络栈中接收包数据。
- iif  lo accept : 允许 本地回环网卡 接收数据包。
- 下一行是标准的 connection tracking ( ct ) 规则，允许已连接请求的响应数据。
- 下一行注释掉的规则表示开放 ssh web 端口，**ct state new** 表示允许其他主机通过这几个端口进行通信。
- 下一行是 ipv6 相关规则，放行用于邻居发现的包，是Ipv6可用。
- 最后一行阻止所有其他的流量并记录阻挡的包数以及字节数。这是一个一条规则实现多个行为的典型规则。 

一般情况下，使用 nftables  设置防火墙，最简单粗暴的方法就是根据需求修改 /etc/nftables.conf 文件。现在假设是一台DNS服务器，那么相关的配置就是

```shelll
tcp dport { 22, 53 } ct state new accept // 开放多个端口
udp dport 53 ct state new accept
```

重载配置

```shell
sudo systemctl reload nftables
ubuntu@ubuntu-2004:~$ sudo nft list ruleset
table inet filter {
	chain input {
		type filter hook input priority filter; policy accept;
		iif "lo" accept
		ct state established,related accept
		tcp dport { 22, 53 } ct state new accept
		udp dport 53 ct state new accept
		ip6 nexthdr ipv6-icmp icmpv6 type { nd-router-advert, nd-neighbor-solicit, nd-neighbor-advert } accept
		counter packets 5 bytes 380 drop
	}
}
```

最后的 counter drop 规则丢弃不合法的数据包，可以将这些记录添加到日志 /var/log/kern.log 中

```shell
counter log prefix "Dropped Packet: " drop
```

现在假设需要限制其他IP登录该主机的22端口，可以添加规则：

```shell
tcp dport 22 ip saddr { 192.168.50.55, 192.168.50.11 } log prefix "Blocked SSH packets: " drop
tcp dport { 22, 53 } ct state new accept
```

这时候如果尝试用被封主机登录就会看到日志

```shell
Mar 12 02:20:17 ubuntu-2004 kernel: [94037.467972] Blocked SSH packets: IN=enp0s5 OUT= MAC=00:1c:42:76:a5:22:00:1c:42:af:3a:a5:08:00 SRC=192.168.50.55 DST=192.168.50.54 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=42006 DF PROTO=TCP SPT=46446 DPT=22 WINDOW=29200 RES=0x00 SYN URGP=0
```

接着允许特定的 ICMP 包，注意，这里IPv4 IPv6 需要分开来写

```shell
ct state new,related,established icmp type { destination-unreachable, time-exceeded, parameter-problem } accept
                ct state established,related,new icmpv6 type { destination-unreachable, time-exceeded, parameter-problem } accept
```

接着在 filter 表中添加一个 prerouting 链来封禁不合法包

```shell
chain prerouting {
				type filter hook prerouting priority 0;
				ct state invalid counter log prefix "Invalid Packets: " drop
				tcp flags & (fin|syn|rst|ack) != syn ct state new counter log drop
}
```

最后完全配置好的文件为

```shell
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
	chain input {
		type filter hook input priority 0;

		# accept any localhost traffic
		iif lo accept

		# accept traffic originated from us
		ct state established,related accept

		# activate the following line to accept common local services
		#tcp dport { 22, 80, 443 } ct state new accept

                tcp dport 22 ip saddr { 192.168.50.55, 192.168.50.11 } log prefix "Blocked SSH packets: " drop

                tcp dport { 22, 53 } ct state new accept
                udp dport 53 ct state new accept
                ct state new,related,established icmp type { destination-unreachable, time-exceeded, parameter-problem } accept
                ct state established,related,new icmpv6 type { destination-unreachable, time-exceeded, parameter-problem } accept

		# accept neighbour discovery otherwise IPv6 connectivity breaks.
		ip6 nexthdr icmpv6 icmpv6 type { nd-neighbor-solicit,  nd-router-advert, nd-neighbor-advert } accept

		# count and drop any other traffic
		counter log prefix "Dropped Packet: " drop
	}
        chain prerouting {
				type filter hook prerouting priority 0;
				ct state invalid counter log prefix "Invalid Packets: " drop
				tcp flags & (fin|syn|rst|ack) != syn ct state new counter log drop
        }
}
```

通过配置文件可以看出，我们可以在同一个文件中同时配置 IPv4 IPv6规则，并且使用 inet 表，这些规则能够同时运用于IPv4，IPv6，除非你有特殊需求，否则 inet 类型的表使用的优先级高于单独的 IPv4， IPv6 类型表。

### 使用 nft 命令行

删除之前的规则，用命令行创建新的规则。

```shell
sudo nft delete table inet filter
sudo nft list tables
sudo nft add table inet ubuntu_filter
sudo nft list tables
```

接着添加新的链，特别注意最后的分号要加转义字符

`注意`： 如果是通过ssh远程连接的千万不要执行这条命令，因为 nft 命令行会立马生效， 导致你被立刻断开连接。

```shell
sudo nft add chain inet ubuntu_filter input { type filter hook input priority 0\; policy drop\;} 
```

每一个 nftables 的协议族都拥有自己的系统勾子，用来在不同阶段处理流量包。只考虑 ip/ip6/inet 协议族的话，一般有这5个勾子：

- Prerouting 
- Input 
- Forward
- Output
- Postrouting

filter 类型的链只需要考虑 input 和 output 勾子就好了。通过在 自定义 链上设置勾子以及优先级，表示这个链接收直接来自网络栈的数据包。在命令的最后还添加了 policy drop 作为默认协议，如果不加则默认为 accept 。

为了不被断开，我们还需要添加一些必要的规则

```shell
sudo nft add rule inet ubuntu_filter input ct state established accept
sudo nft add rule inet ubuntu_filter input tcp dport 22 ct state new accept
```

这样得到的结果是：

```shell
root@ubuntu-2004:~# sudo nft list table inet ubuntu_filter
table inet ubuntu_filter {
	chain input {
		type filter hook input priority filter; policy drop;
		ct state established accept
		tcp dport 22 ct state new accept
	}
}
```

除了这些我们还忘了添加对于本地回环网卡的规则， 可以使用 insert 将规则`添加`到最上面：

```shell
sudo nft insert rule inet ubuntu_filter input iif lo accept
```

结果

```shell
table inet ubuntu_filter {
	chain input {
		type filter hook input priority filter; policy drop;
		iif "lo" accept
		ct state established accept
		tcp dport 22 ct state new accept
	}
}
```

除了能插入规则到最上面，还可以选取任意位置，这时候可以先查看现有规则的 rule handles

```shell
root@ubuntu-2004:~# sudo nft list table inet ubuntu_filter -a
table inet ubuntu_filter { # handle 15
	chain input { # handle 1
		type filter hook input priority filter; policy drop;
		iif "lo" accept # handle 4
		ct state established accept # handle 2
		tcp dport 22 ct state new accept # handle 3
	}
}
```

假设现在需要`添加`一条阻止特定 ip 登录 ssh的规则，这条规则需要在允许ssh登录规则之上才能生效

```shelll
sudo nft insert rule inet ubuntu_filter input position 3 tcp dport 22 ip saddr { 192.168.50.55 } drop
```

结果：

```shell
root@ubuntu-2004:~# sudo nft list table inet ubuntu_filter -a
table inet ubuntu_filter { # handle 15
	chain input { # handle 1
		type filter hook input priority filter; policy drop;
		iif "lo" accept # handle 4
		ct state established accept # handle 2
		tcp dport 22 ip saddr { 192.168.50.55 } drop # handle 6
		tcp dport 22 ct state new accept # handle 3
	}

```

接下来我们`删除`这条规则 

```shell
sudo nft delete rule inet ubuntu_filter input handle 6
```

这些规则会在系统重启之后失效，所以最好写入到 nftables.conf 配置文件中

```shell
sudo sh -c "nft list table inet ubuntu_filter > /etc/nftables.conf"  
```

这样重定向之后的配置文件还缺少了

```shell
#!/usr/sbin/nft -f 
flush ruleset
```

加上之后重新加载配置文件，就能还原我们之前的配置。

##  firewalld 概述

Firewalld 是RHEL/CentOS7和RHEL/CentOS8默认的防火墙管理软件，但是这两个版本还有所差别：7版本的 firewalld 是以 iptables 作为后台引擎，而8版本则换成了 nftables 作为后端引擎。同时 firewalld 与它所用后端引擎的使用方法还不同，它有自己的一套存储规则的方式。

Firewalld 的一个优点是规则动态管理，不用重启 firewall 服务就能加载规则，同时不会断开已有服务连接。

### 管理 firewalld 状态

1. `sudo firewall-cmd --state`
2. `sudo systemctl status firewalld`

### 了解 firewalld zones 

Firewalld 预设了很多 zones，配置文件在 /usr/lib/firewalld/zones 目录下面， 它们是 .xml 格式的

```shell
cd /usr/lib/firewalld/zones
[root@centos8 zones]# ls
block.xml  dmz.xml  drop.xml  external.xml  home.xml  internal.xml  public.xml  trusted.xml  work.xml
```

这里的每个 zone 文件代表一种特定的使用场景，建议了在这种情况下哪些端口开，哪些端口关闭。

以 public zone 为例，我们看一下它的默认设置

```shell
<?xml version="1.0" encoding="utf-8"?>
<zone>
  <short>Public</short>
  <description>For use in public areas. You do not trust the other computers on networks to not harm your computer. Only selected incoming connections are accepted.</description>
  <service name="ssh"/>
  <service name="dhcpv6-client"/>
</zone>
```

可以看到 public zone 默认开了 ssh ipv6-dhcp 服务

也可以使用 firewall-cmd 命令查看系统上所有的 zones 列表，这样就不用进入 zone 目录查看了。

```shell
sudo firewall-cmd --get-zones
[root@centos8 zones]# sudo firewall-cmd --get-zones
block dmz drop external home internal public trusted work
```

查看所有 zone 具体的配置

```shell
sudo firewall-cmd --list-all-zones
block
  target: %%REJECT%%
  icmp-block-inversion: no
  interfaces:
  sources:
  services:
  ports:
  protocols:
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
  ...
  ...
```

查看具体某一个 zone 配置

```shel
sudo firewall-cmd --info-zone=internal
```

现代电脑可能存在多个网卡，每一个网卡都能且仅可以配置一个 zone，查看默认 zone 命令

```shell
sudo firewall-cmd --get-default-zone
```

查看激活的 zone 所对应的网卡

```shell
sudo firewall-cmd --get-active-zones
```

系统安装之后默认激活的是 public zone ，假设我们想要将它更改为 dmz zone

```shell
<?xml version="1.0" encoding="utf-8"?>
<zone>
  <short>DMZ</short>
  <description>For computers in your demilitarized zone that are publicly-accessible with limited access to your internal network. Only selected incoming connections are accepted.</description>
  <service name="ssh"/>
</zone>
```

```shell
sudo firewall-cmd --set-default-zone=dmz
[root@centos8 zones]# sudo firewall-cmd --get-default-zone
dmz
```

默认的 dmz zone  仅仅放开了 ssh 服务，日常工作中我们需要开放更多的服务。除了使用firewall-cmd 添加，我们也可以直接修改配置文件。但是一般情况下不要修改 /usr/lib/firewalld 目录下的文件，正确的修改地方是 /etc/firewalld 目录

```shell
[root@centos8 firewalld]# ls
firewalld.conf  firewalld.conf.old  helpers  icmptypes  ipsets  lockdown-whitelist.xml  services  zones
```

因为我们修改了默认 zone ， 所以 firewalld.conf 会有新旧配置文件

```shell
[root@centos8 firewalld]# sudo diff /etc/firewalld/firewalld.conf /etc/firewalld/firewalld.conf.old
6c6
< DefaultZone=dmz
---
> DefaultZone=public
```

### 向 zone 中添加需要开启的服务

firewalld 默认提供很多常见服务名称，这些名称代表的服务会帮我们开启所需要的端口，这样极大的简化了我们手动添加端口的操作，这些服务器文件存放在 /usr/lib/firewalld/services 目录，可以通过命令查看

```shell
[root@centos8 firewalld]# firewall-cmd --get-services
RH-Satellite-6 amanda-client amanda-k5-client amqp amqps apcupsd audit bacula bacula-client bb bgp bitcoin bitcoin-rpc bitcoin-testnet bitcoin-testnet-rpc bittorrent-lsd ceph ceph-mon cfengine cockpit condor-collector ctdb dhcp dhcpv6 dhcpv6-client distcc dns dns-over-tls docker-registry docker-swarm dropbox-lansync elasticsearch etcd-client etcd-server finger freeipa-4 freeipa-ldap freeipa-ldaps freeipa-replication freeipa-trust ftp ganglia-client ganglia-master git grafana gre high-availability http https imap imaps ipp ipp-client ipsec irc ircs iscsi-target isns jenkins kadmin kdeconnect kerberos kibana klogin kpasswd kprop kshell ldap ldaps libvirt libvirt-tls lightning-network llmnr managesieve matrix mdns memcache minidlna mongodb mosh mountd mqtt mqtt-tls ms-wbt mssql murmur mysql nfs nfs3 nmea-0183 nrpe ntp nut openvpn ovirt-imageio ovirt-storageconsole ovirt-vmconsole plex pmcd pmproxy pmwebapi pmwebapis pop3 pop3s postgresql privoxy prometheus proxy-dhcp ptp pulseaudio puppetmaster quassel radius rdp redis redis-sentinel rpc-bind rsh rsyncd rtsp salt-master samba samba-client samba-dc sane sip sips slp smtp smtp-submission smtps snmp snmptrap spideroak-lansync spotify-sync squid ssdp ssh steam-streaming svdrp svn syncthing syncthing-gui synergy syslog syslog-tls telnet tentacle tftp tftp-client tile38 tinc tor-socks transmission-client upnp-client vdsm vnc-server wbem-http wbem-https wsman wsmans xdmcp xmpp-bosh xmpp-client xmpp-local xmpp-server zabbix-agent zabbix-server
```

在添加新的服务之前，我们可以看看当前开启了哪些服务

```shell
[root@centos8 firewalld]# sudo firewall-cmd --list-services
ssh
```

如果说需要添加服务，比如说 dropbox-lansync 服务，那么可以提前查看该服务所开启的端口

```shell
[root@centos8 firewalld]# sudo firewall-cmd --info-service=dropbox-lansync
dropbox-lansync
  ports: 17500/udp 17500/tcp
  protocols:
  source-ports:
  modules:
  destination:
  includes:
```

现在将默认 zone 设置为 dmz zone，查看它的信息

```shell
[root@centos8 firewalld]# sudo firewall-cmd --info-zone=dmz
dmz (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp0s5
  sources:
  services: ssh
  ports:
  protocols:
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules
```

当前仅有ssh一个服务开启， 我们来添加 web server 服务

```shell
sudo firewall-cmd --add-service=http
```

再次查看结果

```shell
[root@centos8 firewalld]# sudo firewall-cmd --info-zone=dmz
dmz (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp0s5
  sources:
  services: http ssh
  ports:
  protocols:
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
```

但是如果我们加上 --permanent 选项后

```shell
[root@centos8 firewalld]# sudo firewall-cmd --permanent --info-zone=dmz
dmz
  target: default
  icmp-block-inversion: no
  interfaces:
  sources:
  services: ssh
  ports:
  protocols:
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
```

这就表明当前添加的规则没有永久生效，在系统重启之后就会消失。所以为了永久生效，添加命令需要加上 --permanent 选项，但是加了这个选项之后，你需要重载配置才能让它生效；而不加这个选项，配置立马生效。

配置重载命令

```shell
sudo firewall-cmd --reload
```

删除服务只需要将 `--add-service` 换成 `--remove-service` 就行。

### 向 zone 中添加需要开启的端口

有一些服务的端口没有在 firewalld 于定义的服务列表中，这时候就需要我们手动去添加

```shell
sudo firewall-cmd --permanent --add-port=10000/tcp //将端口10000永久添加到默认zone配置中
sudo firewall-cmd --reload // 让配置生效
```

同时添加多个端口

```shell
sudo firewall-cmd --add-port={636/tcp, 637/tcp, 638/udp}
```

同理，删除端口使用 `--remove-port` 选项

如果不想每条命令都加上 --permanent 选项，可以等到全部添加完毕之后，统一设置为 permanent

```shell
sudo firewall-cmd --runtime-to-permanent
```

### ICMP 包阻断

我们查看 dmz zone 的信息

```shell
[root@centos8 firewalld]# sudo firewall-cmd --info-zone=dmz
dmz (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp0s5
  sources:
  services: http ssh
  ports:
  protocols:
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
```

可以看到有一行 icmp-blocks 信息，后面为空，表示允许所有ICMP包。为了安全起见，我们需要阻断一些类型的ICMP包。firewall 可以方便的查看 ICMP 包的类型

```shell
[root@centos8 firewalld]# sudo firewall-cmd --get-icmptypes
address-unreachable bad-header beyond-scope communication-prohibited destination-unreachable echo-reply echo-request failed-policy fragmentation-needed host-precedence-violation host-prohibited host-redirect host-unknown host-unreachable ip-header-bad neighbour-advertisement neighbour-solicitation network-prohibited network-redirect network-unknown network-unreachable no-route packet-too-big parameter-problem port-unreachable precedence-cutoff protocol-unreachable redirect reject-route required-option-missing router-advertisement router-solicitation source-quench source-route-failed time-exceeded timestamp-reply timestamp-request tos-host-redirect tos-host-unreachable tos-network-redirect tos-network-unreachable ttl-zero-during-reassembly ttl-zero-during-transit unknown-header-type unknown-option
```

可以查看这些类型的具体信息

```shell
[root@centos8 firewalld]# sudo firewall-cmd --info-icmptype=network-redirect
network-redirect
  destination: ipv4
[root@centos8 firewalld]# sudo firewall-cmd --info-icmptype=neighbour-advertisement
neighbour-advertisement
  destination: ipv6
```

查看是否阻断特定类型的icmp包

```shell
[root@centos8 firewalld]# sudo firewall-cmd --query-icmp-block=host-redirect
no
```

添加对 redirects 类型的阻断

```shell
[root@centos8 firewalld]# sudo firewall-cmd --add-icmp-block=host-redirect
success
[root@centos8 firewalld]# sudo firewall-cmd --query-icmp-block=host-redirect
yes
[root@centos8 firewalld]# sudo firewall-cmd --info-zone=dmz
dmz (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp0s5
  sources:
  services: http ssh
  ports:
  protocols:
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks: host-redirect
  rich rules:
  
## 同时添加多个类型
[root@centos8 firewalld]# sudo firewall-cmd --add-icmp-block={host-redirect,network-redirect}
Warning: ALREADY_ENABLED: 'host-redirect' already in 'dmz'
success
[root@centos8 firewalld]# sudo firewall-cmd --info-zone=dmz
dmz (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp0s5
  sources:
  services: http ssh
  ports:
  protocols:
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks: host-redirect network-redirect
  rich rules:
[root@centos8 firewalld]# sudo firewall-cmd --runtime-to-permanent
success
```

### pinic 模式

开启关闭命令，这将会立刻关闭所有网络连接，所以如果是远程连接的，千万不要这么做！！！

```shell
sudo firewall-cmd --panic-on
sudo firewall-cmd --panic-off
```

### 日志记录丢弃包的信息

使用的参数为 `--set-log-denied`

```shell
// 查看信息
[root@centos8 firewalld]# sudo firewall-cmd --get-log-denied
off
// 设置
[root@centos8 firewalld]# sudo firewall-cmd --set-log-denied=all
success
```

除了直接设置 all ，也可以分类设置

- unicast
- broadcast
- multicast

```shell
[root@centos8 firewalld]# sudo firewall-cmd --set-log-denied=multicast
success
[root@centos8 firewalld]# sudo firewall-cmd --get-log-denied
multicast
```

红帽系的系统将 包阻断相关日志放在了 /var/log/messages 

### 编写 rich language rules

之前的端口，服务等规则添加能够满足日常简单的需求，但是 更加精细的控制就需要使用到叫做 '富语言规则'的东西。

先来一个例子

```shell
sudo firewall-cmd --add-rich-rule='rule family="ipv4" source address="200.192.0.0/24" service name="http" drop'

[root@centos8 ~]# sudo firewall-cmd --info-zone=dmz
dmz (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp0s5
  sources:
  services: http ssh
  ports:
  protocols:
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks: host-redirect network-redirect
  rich rules:
	rule family="ipv4" source address="200.192.0.0/24" service name="http" drop
```

这是一条阻止一个C类网段访问网站的规则，表示作用于ipv4网络，地址为 200.192.0.0/24 的IP无法访问该服务器的HTTP服务。 可以看到新加的规则显示在了最后。IPv6的规则同理，只需要将 family 换成 ipv6。

有些规则是同时作用于IPv4和IPv6。比如，添加 Network Time Protocol(NTP) 服务规则，并且每分钟记录一条规则。

```shell
sudo firewall-cmd --add-rich-rule='rule service name="ntp" audit limit value="1/m" accept'
```

更详细的规则，可以通过 man firewalld.richlanguage 来查看 。现在添加的这些规则可以在 .xml 文件中看到，因为我的默认 zone 一直是 dmz ，所以可以查看一下 /etc/firewalld/zones/dmz.xml

```shell
[root@centos8 ~]# cat /etc/firewalld/zones/dmz.xml
<?xml version="1.0" encoding="utf-8"?>
<zone>
  <short>DMZ</short>
  <description>For computers in your demilitarized zone that are publicly-accessible with limited access to your internal network. Only selected incoming connections are accepted.</description>
  <service name="ssh"/>
  <service name="http"/>
  <icmp-block name="host-redirect"/>
  <icmp-block name="network-redirect"/>
  <rule family="ipv4">
    <source address="200.192.0.0/24"/>
    <service name="http"/>
    <drop/>
  </rule>
  <rule>
    <service name="ntp"/>
    <audit>
      <limit value="1/m"/>
    </audit>
    <accept/>
  </rule>
</zone>
```

后来添加的富语言规则都是以 rule 为标签。

## 推荐阅读

- nftables wiki: https://wiki.nftables.org/wiki-nftables/index.php/Main_Page
- firewalld documentation for RHEL 7: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/security_guide/sec-using_firewalls
- firewalld documentation for RHEL 8: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/securing_networks/using-and-configuring-firewalls_securing-networks
- The firewalld home page: https://firewalld.org/ 
- nftables examples: https://wiki.gentoo.org/wiki/Nftables/Examples