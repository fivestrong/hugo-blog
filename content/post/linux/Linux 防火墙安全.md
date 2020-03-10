---
title: "Linux 防火墙安全"
date: 2020-03-10T08:57:09+08:00
tags: ["firewall"]
categories: ["linux"]
draft: true
---

## Firewalld 概述

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

假设我们现在默认设置 ACCEPT ，添加 DROP 规则。