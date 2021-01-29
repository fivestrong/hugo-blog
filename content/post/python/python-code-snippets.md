---
title: "Python Code Snippets"
date: 2021-01-28T15:32:54+08:00
tags: ["snippet"]
categories: ["python"]
draft: true
---

### validate IP addresses

```python
import ipaddress
import socket

ip_addresses = ("192.168.0.2", "127.0.0.1",
               "192.168.1.257", "1.0x2.3.4",
                "2001:0bd8:85a3:0000:0000:8a2e:0370:7334"
               )

def valid_ip(ip: str) -> bool:
  try:
    ipaddress.ip_address(ip)
    return True
  except ValueError:
    return False
  
def valid_ipv4_addr(ip: str) -> bool:
  try:
    socket.inet_pton(socket.AF_INET, ip)
    return True
  except socket.error:
    return False

for ip in ip_addresses:
    print(f"{ip:<40} | {str(valid_ip(ip)):<5} | {valid_ipv4_addr(ip)}")

```

```code
192.168.0.2                              | True  | True
127.0.0.1                                | True  | True
192.168.1.257                            | False | False
1.0x2.3.4                                | False | False
2001:0bd8:85a3:0000:0000:8a2e:0370:7334  | True  | False
```

### make iterator

```python
class Halving:
    def __init__(self, start):
        self.number = start
    def __iter__(self):
        return self
    def __next__(self):
        self.number /= 2
        print(self.number)
        if self.number < 10:
            raise StopIteration

ha = Halving(50)
next(ha)
next(ha)
next(ha)

for _ in Halving(50): pass
```

```text
(venv) ➜ python make_an_iterator.py
25.0
12.5
6.25
Traceback (most recent call last):
  File "make_an_iterator.py", line 15, in <module>
    next(ha)
  File "make_an_iterator.py", line 10, in __next__
    raise StopIteration
StopIteration
(venv) ➜ python make_an_iterator.py
25.0
12.5
6.25
```

