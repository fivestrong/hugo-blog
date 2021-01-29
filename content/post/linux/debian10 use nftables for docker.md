---
title: "Debian 10 Use Nftables for Docker"
date: 2020-12-29T15:44:07+08:00
tags: ["nftables"]
categories: ["linux"]
draft: false
---

众所周知，辣鸡Docker在防火墙上做了自定义规则，导致我们通常添加的限制规则不起作用。这样的话，本来仅仅是一个自己使用的Docker服务直接就全部暴露在互联网上了。

虽然官网给出了解决方法，可以将限制规则添加到 `DOCKER-USER` 这个链中，但是这是基于iptables的。而Debian 10 默认防火墙已经切换到了nftables，我试着向`DOCKER-USER`中添加规则，可惜并不生效。具体可以参考[官方文档](https://docs.docker.com/network/iptables/)

查了好多文档最后探索出了一个有效的方法，记录一下以便之后使用

### 创建基本的nftables配置

配置文件位于 /etc/nftables.conf，直接创建包含docker注入的基本规则，这里将表、链名都于docker需求做了统一

```bash
#!/usr/sbin/nft -f

flush ruleset

# List all IPs and IP ranges of your traffic filtering proxy source.
define SAFE_TRAFFIC_IPS = {
    x.x.x.x/24
}

table ip filter {
	chain INPUT {
    type filter hook input priority 0; policy drop;
		ct state invalid counter drop comment "early drop of invalid packets"
		ct state {established, related} counter accept comment "accept all connections related to connections made by us"
		iif lo accept comment "accept loopback"
		iif != lo ip daddr 127.0.0.1/8 counter drop comment "drop connections to loopback not coming from loopback"
		ip protocol icmp limit rate 4/second accept
		ip protocol igmp limit rate 4/second accept
		tcp dport 22 ip saddr $SAFE_TRAFFIC_IPS accept # 向特定ip开放 ssh 端口
		tcp dport {80, 443}  accept # 开放80,443端口
    counter comment "count dropped packets"
	}

	chain FORWARD {
		type filter hook forward priority 0; policy accept;
		counter jump DOCKER-USER
		counter jump DOCKER-ISOLATION-STAGE-1
		oifname "docker0" ct state established,related counter accept
		oifname "docker0" counter jump DOCKER
		iifname "docker0" oifname != "docker0" counter accept
		iifname "docker0" oifname "docker0" counter accept
	}

	chain OUTPUT {
		type filter hook output priority 0; policy accept;
	}

	chain DOCKER {
	}

	chain DOCKER-ISOLATION-STAGE-1 {
		iifname "docker0" oifname != "docker0" counter jump DOCKER-ISOLATION-STAGE-2
		counter return
	}

	chain DOCKER-ISOLATION-STAGE-2 {
		oifname "docker0" counter drop
		counter return
	}

	chain DOCKER-USER {
		counter return
	}
}

table ip nat {
	chain PREROUTING {
		type nat hook prerouting priority -100; policy accept;
	}

	chain INPUT {
		type nat hook input priority 100; policy accept;
	}

	chain POSTROUTING {
		type nat hook postrouting priority 100; policy accept;
		oifname != "docker0" ip saddr 172.17.0.0/16 counter masquerade
	}

	chain OUTPUT {
		type nat hook output priority -100; policy accept;
	}

	chain DOCKER {
		iifname "docker0" counter return
	}
}


table ip6 filter {
	chain INPUT {
		type filter hook input priority 0; policy drop;
		ct state invalid counter drop comment "early drop of invalid packets"
		ct state {established, related} counter accept comment "accept all connections related to connections made by us"
		iif lo accept comment "accept loopback"
		iif != lo ip6 daddr ::1/128 counter drop comment "drop connections to loopback not coming from loopback"
		ip6 nexthdr icmpv6 limit rate 4/second counter accept comment "accept all ICMP types"
		# tcp dport 22 counter accept comment "accept SSH"
		counter comment "count dropped packets"
	}

	chain FORWARD {
		type filter hook forward priority 0; policy drop;
		counter comment "count dropped packets"
	}

	# If you're not counting packets, this chain can be omitted.
	chain OUTPUT {
		type filter hook output priority 0; policy accept;
		counter comment "count accepted packets"
	}
}
```

### 启动docker服务

这里需要注意，如果在Docker服务启动之前就存在 `ip nat` `ip filter`这两个表，docker服务会启动报错，所以可以先清除当前规则

```bash
# 清除当前规则
nft flush ruleset
# 启动docker
systemctl restart docker
# 让我们的规则生效
nft -f /etc/nftables.conf
```

查看表 现在有3个

```bash
nft list tables
table ip nat
table ip filter
table ip6 filter
```

### 启动docker容器

这时候我们启动docker容器的时候，docker会在这些表的默认基础上添加新的容器规则。我们只需要找到合适的位置修改这些规则就能限制对特定的容器进行访问，我找的注入点是 `ip nat DOCKER`这个链，比如启动容器后表变成

```bash
table ip nat {
	chain PREROUTING {
		type nat hook prerouting priority -100; policy accept;
		fib daddr type local counter packets 4353 bytes 217872 jump DOCKER
	}

	chain INPUT {
		type nat hook input priority 100; policy accept;
	}

	chain POSTROUTING {
		type nat hook postrouting priority 100; policy accept;
		oifname != "br-d5710cd4cef7" ip saddr 172.18.0.0/16 counter packets 0 bytes 0 masquerade
		oifname != "docker0" ip saddr 172.17.0.0/16 counter packets 0 bytes 0 masquerade
		meta l4proto tcp ip saddr 172.18.0.4 ip daddr 172.18.0.4 tcp dport 443 counter packets 0 bytes 0 masquerade
		meta l4proto tcp ip saddr 172.18.0.4 ip daddr 172.18.0.4 tcp dport 80 counter packets 0 bytes 0 masquerade
		meta l4proto tcp ip saddr 172.18.0.4 ip daddr 172.18.0.4 tcp dport 22 counter packets 0 bytes 0 masquerade
	}

	chain OUTPUT {
		type nat hook output priority -100; policy accept;
		ip daddr != 127.0.0.0/8 fib daddr type local counter packets 0 bytes 0 jump DOCKER
	}

	chain DOCKER {
		iifname "br-d5710cd4cef7" counter packets 0 bytes 0 return
		iifname "docker0" counter packets 0 bytes 0 return
		iifname != "br-d5710cd4cef7" meta l4proto tcp tcp dport 443 counter packets 4 bytes 256 dnat to 172.18.0.4:443
		iifname != "br-d5710cd4cef7" meta l4proto tcp tcp dport 80 counter packets 2 bytes 124 dnat to 172.18.0.4:80
		iifname != "br-d5710cd4cef7" meta l4proto tcp tcp dport 10022 counter packets 2 bytes 120 dnat to 172.18.0.4:22
	}
}
```

### 手动添加规则

```bash
nft list table ip nat -a # 这个命令会显示表中的handle，方便我们插入和修改规则

....
chain DOCKER { # handle 5
		iifname "br-d5710cd4cef7" counter packets 0 bytes 0 return # handle 13
		iifname "docker0" counter packets 0 bytes 0 return # handle 11
		iifname != "br-d5710cd4cef7" meta l4proto tcp tcp dport 443 counter packets 4 bytes 256 dnat to 172.18.0.4:443 # handle 14
		iifname != "br-d5710cd4cef7" meta l4proto tcp tcp dport 80 counter packets 2 bytes 124 dnat to 172.18.0.4:80 # handle 16
		iifname != "br-d5710cd4cef7" meta l4proto tcp tcp dport 10022 counter packets 2 bytes 120 dnat to 172.18.0.4:22 # handle 18
	}
```

```bash
nft insert rule ip nat DOCKER handle 14 ip saddr != {x.x.x.x/24} tcp dport 22 drop
# 这个规则限制除了指定ip外所有访问22端口的ip都会被丢弃，不会执行到下面的端口转发规则
```

```bash
nft replace rule ip nat DOCKER handle 14 iifname != "br-d5710cd4cef7" meta l4proto tcp tcp dport 443 ip saddr x.x.x.x counter dnat to 172.18.0.4:443
# 也可以通过修改规则替换docker生成的规则,添加额外的限制条件，这里限制443端口只能通过指定ip访问，80、10022不受影响
# 但是不建议这个这样做，修改后规则没法被docker自动删除，建议使用上面的规则直接阻断。
```



### 注意事项

这种方法其实并不灵活，因为一旦重启`nftables`服务所有规则都会重新初始化，还需要重新启动容器。不过可以做到了将阻止规则写入配置，每次重启nftables规则都会自动写入，这时候只需要重启docker 容器服务就可以了，省去了手动添加阻止规则的步骤。

nftables官方[帮助文档](https://wiki.nftables.org/wiki-nftables/index.php/Main_Page)

