---
title: "Linux 防火墙安全（1）"
date: 2020-03-10T08:57:09+08:00
tags: ["firewall"]
categories: ["linux"]
draft: false
---

## 防火墙概述

在工作环境中，我们可能需要多种多样的防火墙设备以满足各种各样的防护需求。

- 一些边界设备将互联网与内网分离，并维护路由表以提供公有IP到私有IP的转换。同时它也提供各种各样的访问控制，以避免非法访问。其他功能还有数据包检测功能，防止网络攻击、恶意软件以及过滤敏感信息。
- 一个大型网络经常被划分为许多的子网络，通过防火墙设备能够做到只有通过认证的用户才能访问子网络的资源。
- 同时，除了硬件防火墙，服务器系统中也存在防火墙服务。这种防火墙拥有访问控制权限，可以避免在黑客攻破外部防火墙之后轻松的攻击内部的各个相关服务器。也可以配置对于本机的端口扫描、DoS攻击等防护。

前两项有专门的厂商以及团队会提供硬件以及技术上的支持，作为一个服务器维护者，我们应该着重关注系统当中的防火墙软件以及技术的使用。

## Iptables 概述

我们经常会误以为linux 提供的防火墙就是 `iptables`，事实上各个linux发行版都会内置的防火墙叫做 `netfilter`。`Iptables` 仅仅是用于管理`netfilter`众多的命令行工具之一。它作为Linux 2.6版本内核的新功能加入，一直到现在，它的优势很明显：

- 历史悠久，大多数管理员都会使用。
- 方便在shell脚本中调用
- 它的灵活性很强，可以设置简单的端口过滤，路由规则或者虚拟的私有网络
- 即便有些发行版系统中没有对它进行配置，但是大多数系统都预装了iptables
- 文档资料齐全

当然，只有有点没有缺点是不可能的：

- IPv4 和 IPv6 需要分别实现规则，如果你启用了IPv4、IPv6双栈，那么你得同时维护两套规则，运行两套守护进程
- 如果需要做MAC 桥接，需要安装ebtables，不仅需要单独安装，还需要使用它独有的语法规则
- arptables 应用同理
- 添加一条新的规则，整个规则表需要重新加载，如果规则多的话会影响效率

直到现在，多数的Linux发行版默认的防火墙管理软件都是普通的iptables。但是Red Hat/CentOS 7 版本以后，它默认使用`firewalld`作为前端管理iptables规则。Ubuntu 也开发了 `Uncomplicated Firewall(ufw)` 用于管理iptables。更新的工具是`nftables`，在Debian/Ubuntu系统中作为可选软件，在Red Hat 8/CentOS8 系统中nftables 更是作为 firewalld 的后端服务取代了 iptables。（是的，你没看错）

### iptables 的基本使用

iptables 由5个表组成，每个表有自己的作用：

- Filter table: 最常用的表，一般基本规则都在创建在这个表上
- Network Address Translation(NAT) table: 用于公网到私网的通信
- Mangle table: 用于更改通过网络数据包的状态
- Raw table: 用于不需要连接跟踪的包
- Security table: 仅用于安装有SELinux的系统

从基本开始，我们学习使用 `filter table` 的使用。每个表由包含多条规则的链组成，`filter table` 包含 `INPUT`, `FORWARD`, `OUTPUT` 三条链。

`注意`：虽然 Red Hat Enterprise Linux 7/8 默认安装了 iptables 服务，但是默认是禁用的，取而代之是 firewalld。你也可以使用 iptables 服务，那么就必须停止使用 firewalld ，因为两者是冲突的。

这里先以Ubuntu系统作为演示，

使用 `sudo iptables -L` 查看当前配置

```shell
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```

同理，查看IPv6规则命令是 `sudo ip6tables -L` :

```shell
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```

可以看到，Ubuntu默认规则都是空的，主机对外完全开放。现在来创建一条规则

```shell
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

- -A INPUT: `-A` 将一条规则添加到指定链的最后(这里是INPUT),如果需要将规则放到最顶端，可以使用 `-I` 。
- -m: 启用 iptables 模块， 这里使用了 conntrack 模块用来追踪连接状态。
- --ctstate: 后面的两个参数做了两件事，第一，寻找客户端已经与服务器建立的连接；第二，查找服务器返回给这个客户端数据的相关连接。所以如果一个用户通过浏览器连接到了我们的网站，这条规则允许网站返回的数据包通过。
- -j: 表示jump.规则要跳到特殊的目标，这里是ACCEPT。

最后添加结果是：

```shell
ubuntu@VM-38-8-ubuntu:~$ sudo iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
ACCEPT     all  --  anywhere             anywhere             ctstate RELATED,ESTABLISHED
Chain FORWARD (policy ACCEPT)
target     prot opt source               destination
Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```

接着我们开放22端口，允许远程登录:

```shell
sudo iptables -A -INPUT -p tcp --dport ssh -j ACCEPT
```

- -A INPUT: 跟之前一样，将规则添加到INPUT链最后。
- -p tcp: -p 指明这条规则应用的协议。这里是ssh要用的TCP协议。
- --dport ssh: 当选项不是单词选项时，需要使用双线。 --dport 表示开放的目标端口是哪个(这里使用了22端口服务的名称ssh)
- -j ACCEPT: 前面所有的规则和这个选项一同使用，表示允许其他机器通过ssh连接到本机。

假设现在是台 DNS 服务， 需要开放 TCP UDP 的53端口：

```shell
sudo iptables -A INPUT -p tcp --dport 53 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 53 -j ACCEPT
```

除了这些，还需要特别注意，一定要允许本地回环网卡。现在将这个规则添加到 INPUT 链的最上端。

```shell
sudo iptables -I INPUT 1 -i lo -j ACCEPT
```

一般来说查看 iptables 规则信息最好加上 -n 选项，它以数字的形式表示主机地址以及端口。

虽然有了以上规则，即便应用了，也还是`允许所有`的请求进入，因为我们没有创建默认的阻止规则。

### 利用 iptables 阻止 ICMP 协议

其实为了服务器安全，我们需要阻止 ICMP 请求，这是因为：

- 黑客可以通过僵尸网络利用多台主机同时向你发起 ping 请求，这样会耗尽服务器资源。
- 某些ICMP协议相关的漏洞可能会导致黑客获取系统权限，定向流量到恶意服务器，甚至会导致系统崩溃。

但是，一刀切的阻止全部ICMP数据包是不可取的，我们应该阻止一些特定类型的ICMP包，因为有些类型的ICMP包对于某些网络功能是必须的。下面是一些需要被允许的ICMP类型

```shell
sudo iptables -A INPUT -m conntrack -p icmp --icmp-type 3 --ctstate NEW,ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -m conntrack -p icmp --icmp-type 11 --ctstate NEW,ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -m conntrack -p icmp --icmp-type 12 --ctstate NEW,ESTABLISHED,RELATED -j ACCEPT
```

- -m conntrack: 同之前一样，conntrack 模块允许特定状态的数据包，这次除了允许已连接主机的数据包（ESTABLISHED,RELATED ），同时还允许了NEW，允许其他主机像我们发送包。
- -p icmp: 表示 ICMP 协议
- --icmp-type: ICMP 信息的类型
  - Type 3: 目标不可达信息。可以告诉服务器不能到达目标主机，并且注明原因。例如，如果服务器的包过大，交换机处理不了，那么交换机会返回一个ICMP信息告诉服务器去适当的分片。如果没有ICMP信息，服务器就会因为一直传输大包而导致网络连接问题。
  - Type 11: 超时信息可以让服务器知道包到达目标之前是否超出了它的 Time-to-Live(TTL)值，或者碎片包重组超出了TTL时间。
  - Type 12: 参数错误信息表明服务器发送了有错误 IP 头信息的包。换句话说，IP 头要么缺失了可选 flag, 要么长度不合法。
  - Type 0 and Type 8: 它们代表 ping 包。Type 8 是发送 ping 到主机的请求包，Type 0 是对方主机返回的是否存活的响应包。允许 ping 包的好处是可以方便网络调试，在这种情况下，可以零时添加规则放行ping。
  - Type 5: 重定向信息。如果你有路由器有高效的访问路劲供服务器去使用，那么开启这个信息会有很大的帮助。否则黑客可能会将你重定向到你没想去的目标，所以封掉为好。

还有很多类型的ICMP信息没有提到，不过目前这些够用了

### 阻止所有没有被允许的请求

有两种阻止的方式：

- 为 INPUT 设置默认 DROP 或者 REJECT 规则。
- 将规则设置为ACCEPT，然后在INPUT链底部设置 DROP 或者 REJECT 规则

DROP 和 REJECT 的区别是，DROP封锁包并且不会回送任何信息给发送者。而 REJECT 封锁包，会返回给发送者具体原因。

在INPUT链底部创建DROP规则

```shell
sudo iptables -A INPUT -j DROP
```

也可以将它设置为默认的 DROP 规则

```shel
sudo iptables -P INPUT DROP
```

将DROP 或者 REJECT 设置成为默认规则的好处是可以更加简单的添加新的 ACCEPT 规则。如果我们设置了 ACCEPT为默认规则，之后创建的 DROP 或者 REJECT 规则就必须要一直在链的最下面。

iptables 规则生效的原则是`由上到下`按顺序执行的。所以你需要将任何的ACCEPT规则放在最后的DROP或者REJECT规则之上，这比每次都将它们插入到规则表最下面方便一些。

通过命令行写入的规则会在重启之后消失，Ubuntu 上最简单的方法是安装 `iptables-persistent` 包

```shell
sudo apt install iptables-persistent
```

安装过程中会询问你是否将当前的 ipv4 | ipv6 规则保存到文件中，之后这些文件会被开机自动加载。这些文件位于 `/etc/iptables/rules.v*`

如果之后再添加了新的规则，可以使用 `iptables-save` 来保存覆盖这些文件

### 阻止无效的包

TCP的传输需要事先建立`三次握手`，当你用浏览器访问网站的时候，两者建立连接大致需要以下步骤：

- 浏览器发送一个仅包含 `SYN flag` 的包，说："你好，服务器。我想要深入的了解你"
- 当服务器接到这个包，会返回一个带有 `SYN` 和 `ACK` flag 的包，回答:"~~滚犊子~~你好啊，小哥哥，好的呢，我这就去准备"
- 当客户端接到服务器的SYN-ACK包之后，它会再发一个ACK包确认，"得了，i'm on my way。"
- 当接到最后的ACK包，服务器就开始建立连接通道，双方展开深入的交流。

基本上任何的TCP连接都是这一套流程。然后有一些聪明人就开始利用各种工具创建带有特殊flags的包，这些包就被称为 `invalid packets`，这种包有些问题

- 这种包可以诱导目标主机得到它运行的系统，服务，甚至软件版本。
- 这种包可以诱发目标主机某些安全漏洞。
- 这种包需要比其他正常包更多的资源去处理，可以用于DoS攻击。

在 INPUT 链最后添加 DROP ALL 能够阻止大部分的无效包，但是或多或少会有遗漏，并且这种做法也不够高效。这是因为这些包会遍历整个 INPUT 链上的规则，并依次匹配，只有当找不到一条能够 ALLOW 的规则，才会最终被 DROP ALL挡掉。

所以，更有效的方法是在它进入到 INPUT 链之前就将这些包过滤掉。这时候我们需要用到 PREROUTING 链，而这个链不在 `filter` 表中。这就要用到 `mangle` 表中的 PREROUTING。

```shell
sudo iptables -t mangle -A PREROUTING -m conntrack --ctstate INVALID -j DROP
sudo iptables -t mangle -A PREROUTING -p tcp ! --syn -m conntrack --ctstate NEW -j DROP
```

第一条规则会阻挡大部分的 `invalid` 包，还有遗漏的用第二条补刀，这条将所有不是 SYN 并状态为NEW的包阻挡。

使用 `iptables -L` 是看不到添加结果的，因为它默认只显示 `filter` 表内容，这里需要使用

```shell
sudo iptables -t mangle -L 
```

这样添加的规则系统重启就失效，可以使用 iptables-save 命令将规则导入文件，然后替换掉 /etc/iptables/下面的文件

```shell
sudo iptbales-save > rules.v4
sudo cp rules.v4 /etc/iptables/
```

为了测试这条规则是否生效，可以使用Nmap扫描软件，接下来我们对这台机器进行 XMAS 扫描

```shell
[root@centos8 ~]# nmap -sX 192.168.50.54
Starting Nmap 7.70 ( https://nmap.org ) at 2020-03-11 02:32 EDT
Nmap scan report for ubuntu-2004 (192.168.50.54)
Host is up (0.00033s latency).
All 1000 scanned ports on ubuntu-2004 (192.168.50.54) are open|filtered
MAC Address: 00:1C:42:76:A5:22 

Nmap done: 1 IP address (1 host up) scanned in 21.51 seconds
```

默认情况下，Nmap只扫描最常用的1000个端口，这个 XMAS 会发送带有FIN，PSH，URG 标识的包。这里的 `open|filtered` 表明扫描被阻断，无法判断端口状态。（事实上这台机器的22端口是开着的）可以通过查看防火墙规则来确定是被哪个规则所阻断的。

```shell
ubuntu@ubuntu-2004:~$ sudo iptables -t mangle -L -v
Chain PREROUTING (policy ACCEPT 847 packets, 184K bytes)
 pkts bytes target     prot opt in     out     source               destination
 2000 80000 DROP       all  --  any    any     anywhere             anywhere             ctstate INVALID
    0     0 DROP       tcp  --  any    any     anywhere             anywhere             tcp flags:!FIN,SYN,RST,ACK/SYN ctstate NEW
```

可以看到第一条规则阻断了 2000个包共80000 bytes。接着将统计清零，试试第二条规则。

```shell
ubuntu@ubuntu-2004:~$ sudo iptables -t mangle -Z
ubuntu@ubuntu-2004:~$ sudo iptables -t mangle -L -v
Chain PREROUTING (policy ACCEPT 19 packets, 1320 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 DROP       all  --  any    any     anywhere             anywhere             ctstate INVALID
    0     0 DROP       tcp  --  any    any     anywhere             anywhere             tcp flags:!FIN,SYN,RST,ACK/SYN ctstate NEW
```

下面是一个 Window 扫描，用ACK包去轰炸目标主机

```shell
[root@centos8 ~]# nmap -sW 192.168.50.54
Starting Nmap 7.70 ( https://nmap.org ) at 2020-03-11 02:46 EDT
Nmap scan report for ubuntu-2004 (192.168.50.54)
Host is up (0.00028s latency).
All 1000 scanned ports on ubuntu-2004 (192.168.50.54) are filtered
MAC Address: 00:1C:42:76:A5:22 

Nmap done: 1 IP address (1 host up) scanned in 21.46 seconds
```

查看结果，同样被阻断了。

```shell
ubuntu@ubuntu-2004:~$ sudo iptables -t mangle -L -v
Chain PREROUTING (policy ACCEPT 288 packets, 57626 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 DROP       all  --  any    any     anywhere             anywhere             ctstate INVALID
 2000 80000 DROP       tcp  --  any    any     anywhere             anywhere             tcp flags:!FIN,SYN,RST,ACK/SYN ctstate NEW
```

如果你想要看看不阻断的效果，可以使用选项 -D 删除规则

```shell
sudo iptables -t mangle -D PREROUTING 1
sudo iptables -t mangle -D PREROUTING 1
```

再次使用 XMAS 扫描

```shell
[root@centos8 ~]# nmap -sX 192.168.50.54
Starting Nmap 7.70 ( https://nmap.org ) at 2020-03-11 02:52 EDT
Nmap scan report for ubuntu-2004 (192.168.50.54)
Host is up (0.00012s latency).
Not shown: 999 closed ports
PORT   STATE         SERVICE
22/tcp open|filtered ssh
MAC Address: 00:1C:42:76:A5:22 

Nmap done: 1 IP address (1 host up) scanned in 2.77 seconds
```

### IPv6 防护

随着IPv6越来越普及，很多服务器都再服务器端配置了IPv6地址。但是正如之前所说，iptables 的一个缺点就是，v4和v6的防火墙是两个服务，你所有在IPv4上的操作都得在IPv6上再来一次。唯一的区别就是使用命令变成了 `ip6tables`

```shell
sudo ip6tables -A INPUT -i lo -j ACCEPT
sudo ip6tables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo ip6tables -A INPUT -p tcp --dport ssh -j ACCEPT
sudo ip6tables -A INPUT -p tcp --dport 53 -j ACCEPT
sudo ip6tables -A INPUT -p udp --dport 53 -j ACCEPT
```

还有一个很大的不同就是，相比IPv4你需要放行更多的ICMP协议，因为：

- 在IPv6中新的ICMP类型取代了之前的 Address Resolution Protocol(ARP)
- IPv6地址分配通过与其他主机交换ICMP信息，而不是传统的DHCP。
- 如果需要通过IPv4作为IPv6的通信隧道，那么ping 包就必须被允许

```shell
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 1 -j ACCEPT // 目标不可达
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 2 -j ACCEPT // 包太大
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 3 -j ACCEPT // 超时
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 4 -j ACCEPT // 包头信息错误

sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 128 -j ACCEPT // echo requests
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 129 -j ACCEPT // echo responses

// Link-local Multicast Receiver Notification messages
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 130 -j ACCEPT // Listener query
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 131 -j ACCEPT // Listener report
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 132 -j ACCEPT // Listener down
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 143 -j ACCEPT // Listener report v2

// neiphbor and router discoverry message types
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 134 -j ACCEPT  // Router solicitation
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 135 -j ACCEPT  // Router advertisement
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 136 -j ACCEPT  // Neiphbor solicitation
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 141 -j ACCEPT  // Neiphbor advertisement
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 142 -j ACCEPT  // Inverse neiphbor discovery solicitation

```

允许 Secure Neiphbor Discovery(SEND) 

```shell
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 148 -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 149 -j ACCEPT
```

允许 Multicast Router Discovery messages

```shel
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 151 -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 152 -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 153 -j ACCEPT
```

最后 添加 DROP 规则

```shell
sudo ip6tables -A INPUT -j DROP
```

这些仅仅是成功配置IPv6防火墙规则的冰山一角，这里 [IptablesHowTo](https://help.ubuntu.com/community/IptablesHowTo) 可以了解在Ubuntu上配置 iptables 的整个过程。

## Ubuntu 系统内置防火墙软件 ufw

`ufw` 最新版本的Ubuntu都默认内置这个软件，虽然仍然使用 `iptables service` ，但是在易用性上做了很多努力。只要你激活服务，设置你想要开放的端口，一个最简单的防火墙就开启了。而且 `ufw` 命令会自动配置IPv4和IPv6规则。

安装命令

```shell
sudo apt install ufw
```

### 配置ufw

在Ubuntu 18.04 之后，ufw 的系统服务是默认开启的，但是需要你手动去激活启用规则。

启动 ufw 命令

```shell
sudo systemctl enable --now ufw
```

激活 ufw 命令

```shell
sudo ufw enable
```



启动之后第一件事当然是打开ssh端口

```shell
sudo ufw allow 22/tcp
```

查看结果

```shell
sudo iptables -L 

Chain ufw-user-input (1 references)
target     prot opt source               destination
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ssh
```

上面结果是节选的，整个表非常的长，基本的配置ufw都帮我们默认做了。比如说：默认添加了防护DoS攻击的规则，记录被封锁包的日志规则。

上面的规则是添加了 tcp 的 22端口，如果你希望同时开启 tcp udp 

```shell
sudo ufw allow 53

Chain ufw-user-input (1 references)
target     prot opt source               destination
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ssh
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:domain
ACCEPT     udp  --  anywhere             anywhere             udp dpt:domain
```

这时候你通过 sudo ip6tables -L 查看规则，会发现不仅你自定义的规则被添加了进去，之前特别繁琐的 ICMP 规则也默认被添加了。

快速查看你所添加的防火墙信息，可以使用 status 选项。

```shell
ubuntu@ubuntu-2004:~$ sudo ufw status
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       Anywhere
53                         ALLOW       Anywhere
22/tcp (v6)                ALLOW       Anywhere (v6)
53 (v6)                    ALLOW       Anywhere (v6)
```

### 直接编辑 ufw 规则文件

`ufw` 的规则文件存放在 `/etc/ufw` 下面，不同的规则存放在不同的文件中

```shell
ubuntu@ubuntu-2004:~$ ls -l /etc/ufw/
total 48
-rw-r----- 1 root root  915 Dec 17 07:47 after6.rules
-rw-r----- 1 root root 1126 Jan 17 02:39 after.init
-rw-r----- 1 root root 1004 Dec 17 07:47 after.rules
drwxr-xr-x 2 root root 4096 Jan 31 10:55 applications.d
-rw-r----- 1 root root 6700 Dec 17 07:47 before6.rules
-rw-r----- 1 root root 1130 Jan 17 02:39 before.init
-rw-r----- 1 root root 2537 Dec 17 07:47 before.rules
-rw-r--r-- 1 root root 1391 Aug 16  2019 sysctl.conf
-rw-r--r-- 1 root root  313 Mar 11 04:11 ufw.conf
-rw-r----- 1 root root 1524 Mar 11 04:18 user6.rules
-rw-r----- 1 root root 1517 Mar 11 04:18 user.rules
```

`user6.rules` 和 `user.rules` 文件默认不可更改，即便你修改保存了，使用 `sudo ufw reload` 之后你所有的修改都会被还原。

如果需要保存一些复杂的规则，可以手动将规则添加到 before.rules , before6.rules, after.rules, after6.rules这四个文件中。

查看 before.rules 文件， 有一行规则如下

```shell
# drop INVALID packets (logs these in loglevel medium and higher)
-A ufw-before-input -m conntrack --ctstate INVALID -j ufw-logging-deny
-A ufw-before-input -m conntrack --ctstate INVALID -j DROP
```

这就是之前我们做过的丢弃 INVALID 包，这里多了一个日志记录的规则。出于性能的考虑，我们可以将规则添到 mangle 表中，而不是现在的 filter 表中。接下来将规则写到 before 文件，/etc/ufw/before.rules

```shell
# Mangle table added 
*mangle
:PREROUTING ACCEPT [0:0]
-A PREROUTING -m conntrack --ctstate INVALID -j DROP
-A PREROUTING -p tcp -m tcp ! --tcp-flags FIN,SYN,RST,ACK SYN -m conntrack --ctstate NEW -j DROP
COMMIT
```

同样添加到 /etc/ufw/before6.rules 文件，然后重载配置文件。

```shell
sudo ufw reload
```

基本上 ufw 还是使用最古老的 iptables 技术作为核心引擎，相当于封装了一些好用的API到命令行界面。

## 总结

前面学习了用于管理内核防火墙 `netfilter`，四种前台管理工具中的两种。第一种是最古老的 iptables ，好用但是有不少缺点；第二种是 Ubuntu 系统实现的简化管理工具 `ufw` 。

接下来会介绍 `nftables` 和 `firewalld` ，这两个比较新的 netfilter 前台管理工具。

## 推荐阅读

- 25 iptables netfilter firewall examples: https://www.cyberciti.biz/tips/linux-iptables-examples.html 
- Linux IPv6 how-to: http://tldp.org/HOWTO/html_single/Linux+IPv6-HOWTO/ 

- Recommendations for Filtering ICMPv6 Messages in Firewalls: https://www.ietf.org/rfc/rfc4890.txt 
- Rate-limiting with ufw: https://45squared.com/rate-limiting-with-ufw/ 
- UFW Community Help Wiki: https://help.ubuntu.com/community/UFW 
- How to set up a Linux firewall with UFW on Ubuntu 18.04: https://linuxize.com/post/how-to-setup-a-firewall-with-ufw-on-ubuntu-18-04/

