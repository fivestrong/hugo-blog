---
title: iptables防火墙规则实例
date: 2016-03-14 14:18:35
draft: true
---
```bash

#!/bin/bash
# Clear any previous rules.
/sbin/iptables -F
# Default drop policy.
/sbin/iptables -P INPUT DROP
/sbin/iptables -P OUTPUT ACCEPT
# Allow anything over loopback and vpn.
/sbin/iptables -A INPUT -i lo -s 127.0.0.1 -d 127.0.0.1 -j ACCEPT
/sbin/iptables -A OUTPUT -o lo -s 127.0.0.1 -d 127.0.0.1 -j ACCEPT
/sbin/iptables -A INPUT -i tun0 -j ACCEPT
/sbin/iptables -A OUTPUT -o tun0 -j ACCEPT
/sbin/iptables -A INPUT -p esp -j ACCEPT
/sbin/iptables -A OUTPUT -p esp -j ACCEPT
# Drop any tcp packet that does not start a connection with a syn flag.
/sbin/iptables -A INPUT -p tcp ! --syn -m state --state NEW -j DROP
# Drop any invalid packet that could not be identified.
/sbin/iptables -A INPUT -m state --state INVALID -j DROP
# Drop invalid packets.
/sbin/iptables -A INPUT -p tcp -m tcp --tcp-flags FIN,SYN,RST,PSH,ACK,URG NONE -j DROP
/sbin/iptables -A INPUT -p tcp -m tcp --tcp-flags SYN,FIN SYN,FIN              -j DROP
/sbin/iptables -A INPUT -p tcp -m tcp --tcp-flags SYN,RST SYN,RST              -j DROP
/sbin/iptables -A INPUT -p tcp -m tcp --tcp-flags FIN,RST FIN,RST              -j DROP
/sbin/iptables -A INPUT -p tcp -m tcp --tcp-flags ACK,FIN FIN                  -j DROP
/sbin/iptables -A INPUT -p tcp -m tcp --tcp-flags ACK,URG URG                  -j DROP
# Reject broadcasts to 224.0.0.1
/sbin/iptables -A INPUT -s 224.0.0.0/4 -j DROP
/sbin/iptables -A INPUT -d 224.0.0.0/4 -j DROP
/sbin/iptables -A INPUT -s 240.0.0.0/5 -j DROP
# Blocked ports
/sbin/iptables -A INPUT -p tcp -m state --state NEW,ESTABLISHED,RELATED --dport 8010 -j DROP
# Allow TCP/UDP connections out. Keep state so conns out are allowed back in.
/sbin/iptables -A INPUT  -p tcp -m state --state ESTABLISHED     -j ACCEPT
/sbin/iptables -A OUTPUT -p tcp -m state --state NEW,ESTABLISHED -j ACCEPT
/sbin/iptables -A INPUT  -p udp -m state --state ESTABLISHED     -j ACCEPT
/sbin/iptables -A OUTPUT -p udp -m state --state NEW,ESTABLISHED -j ACCEPT
# Allow only ICMP echo requests (ping) in. Limit rate in. Uncomment if needed.
/sbin/iptables -A INPUT  -p icmp -m state --state NEW,ESTABLISHED --icmp-type echo-reply -j ACCEPT
/sbin/iptables -A OUTPUT -p icmp -m state --state NEW,ESTABLISHED --icmp-type echo-request -j ACCEPT
# or block ICMP allow only ping out
/sbin/iptables -A INPUT  -p icmp -m state --state NEW -j DROP
/sbin/iptables -A INPUT  -p icmp -m state --state ESTABLISHED -j ACCEPT
/sbin/iptables -A OUTPUT -p icmp -m state --state NEW,ESTABLISHED -j ACCEPT
# Allow ssh connections in.
#/sbin/iptables -A INPUT -p tcp -s 1.2.3.4 -m tcp --dport 22 -m state --state NEW,ESTABLISHED,RELATED -m limit --limit 2/m -j ACCEPT
# Drop everything that did not match above or drop and log it.
#/sbin/iptables -A INPUT   -j LOG --log-level 4 --log-prefix "IPTABLES_INPUT: "
/sbin/iptables -A INPUT   -j DROP
#/sbin/iptables -A FORWARD -j LOG --log-level 4 --log-prefix "IPTABLES_FORWARD: "
/sbin/iptables -A FORWARD -j DROP
#/sbin/iptables -A OUTPUT  -j LOG --log-level 4 --log-prefix "IPTABLES_OUTPUT: "
/sbin/iptables -A OUTPUT  -j ACCEPT
iptables-save > /dev/null 2>&1

```