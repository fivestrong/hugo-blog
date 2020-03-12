---
title: "Linux 防火墙安全（2）"
date: 2020-03-11T17:06:48+08:00
tags: ["firewall"]
categories: ["linux"]
draft: true
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

## 总结

## 推荐阅读

- nftables wiki: https://wiki.nftables.org/wiki-nftables/index.php/Main_Page
- firewalld documentation for RHEL 7: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/security_guide/sec-using_firewalls
- firewalld documentation for RHEL 8: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/securing_networks/using-and-configuring-firewalls_securing-networks
- The firewalld home page: https://firewalld.org/ 
- nftables examples: https://wiki.gentoo.org/wiki/Nftables/Examples