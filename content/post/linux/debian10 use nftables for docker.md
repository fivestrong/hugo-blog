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

配置文件位于 /etc/nftables.conf

```bash
#!/usr/sbin/nft -f

flush ruleset

# List all IPs and IP ranges of your traffic filtering proxy source.
define SAFE_TRAFFIC_IPS = {
    x.x.x.x/24
}
table inet filter {
	chain input {
		type filter hook input priority 0; policy drop;
		ct state established,related accept
		ct state invalid drop
		iifname "lo" accept
		ip protocol icmp limit rate 4/second accept
		ip6 nexthdr ipv6-icmp limit rate 4/second accept
		ip protocol igmp limit rate 4/second accept
		tcp dport 22 ip saddr $SAFE_TRAFFIC_IPS accept # 向特定ip开放 ssh 端口
		icmpv6 type { nd-router-advert, nd-neighbor-solicit, nd-neighbor-advert } ip6 hoplimit 1 accept
		icmpv6 type { nd-router-advert, nd-neighbor-solicit, nd-neighbor-advert } ip6 hoplimit 255 counter packets 0 bytes 0 accept
		counter packets 0 bytes 0 reject with icmpx type admin-prohibited
	}

	chain forward {
		type filter hook forward priority 0; policy accept;
	}

	chain outbound {
		type filter hook output priority 0; policy accept;
	}
}
```

### 启动docker服务

```bash
systemctl restart docker
```

这一步能够让docker生成需要的基本规则，并且不会改变我们的默认表 `inet filter`,查看表 现在有3个

```bash
nft list tables
table inet filter
table ip nat
table ip filter
```

我们分别将新生成的表的内容添加到`nftables.conf`配置文件中

```bash
nft list table nat 

table ip nat {
	chain PREROUTING {
		type nat hook prerouting priority -100; policy accept;
		fib daddr type local counter packets 0 bytes 0 jump DOCKER
	}

	chain INPUT {
		type nat hook input priority 100; policy accept;
	}

	chain POSTROUTING {
		type nat hook postrouting priority 100; policy accept;
		oifname != "docker0" ip saddr 172.17.0.0/16 counter packets 0 bytes 0 masquerade
	}

	chain OUTPUT {
		type nat hook output priority -100; policy accept;
		ip daddr != 127.0.0.0/8 fib daddr type local counter packets 0 bytes 0 jump DOCKER
	}

	chain DOCKER {
		iifname "docker0" counter packets 0 bytes 0 return
	}
}
```

```bash
nft list table filter

table ip filter {
	chain INPUT {
		type filter hook input priority 0; policy accept;
	}

	chain FORWARD {
		type filter hook forward priority 0; policy accept;
		counter packets 0 bytes 0 jump DOCKER-USER
		counter packets 0 bytes 0 jump DOCKER-ISOLATION-STAGE-1
		oifname "docker0" ct state related,established counter packets 0 bytes 0 accept
		oifname "docker0" counter packets 0 bytes 0 jump DOCKER
		iifname "docker0" oifname != "docker0" counter packets 0 bytes 0 accept
		iifname "docker0" oifname "docker0" counter packets 0 bytes 0 accept
	}

	chain OUTPUT {
		type filter hook output priority 0; policy accept;
	}

	chain DOCKER {
	}

	chain DOCKER-ISOLATION-STAGE-1 {
		iifname "docker0" oifname != "docker0" counter packets 0 bytes 0 jump DOCKER-ISOLATION-STAGE-2
		counter packets 0 bytes 0 return
	}

	chain DOCKER-ISOLATION-STAGE-2 {
		oifname "docker0" counter packets 0 bytes 0 drop
		counter packets 0 bytes 0 return
	}

	chain DOCKER-USER {
		counter packets 0 bytes 0 return
	}
}
```

这样我们每次重置配置的时候都能生成docker所需的默认表

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
nft insert rule ip nat DOCKER handle 14 ip saddr != {x.x.x.x/24} drop
# 这个规则限制除了指定ip段都不能通过nat访问到容器端口
```

```bash
nft replace rule ip nat DOCKER handle 14 iifname != "br-d5710cd4cef7" meta l4proto tcp tcp dport 443 ip saddr x.x.x.x counter dnat to 172.18.0.4:443
# 这条规则可以替换docker生成的规则,添加额外的限制条件，这里限制443端口只能通过指定ip访问，80、10022不受影响
```



### 注意事项

这种方法其实并不灵活，因为一旦重启`nftables`服务所有规则都会重新初始化，还需要重新启动容器，并重新添加规则。这给向 `inat filter`中加规则增加了麻烦，只能动态添加之后配置文件再添加一次，以便重启防火墙生效。

而且有一点，如果在Docker服务启动之前就存在 `ip nat` `ip filter`这两个表，docker服务会启动报错，只能先启动docker再启动nftables。

还有就是我们手动添加或者修改过的规则，docker不会自动为我们删除，所以重置之前记得清除规则，否则可能会有问题。

nftables官方[帮助文档](https://wiki.nftables.org/wiki-nftables/index.php/Main_Page)

