---
title: "Python Code Snippets"
date: 2021-01-28T15:32:54+08:00
tags: ["snippet"]
categories: ["python"]
draft: false
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

### accept extra arguments

```python
import argparse

parser = argparse.ArgumentParser(description='Calculate your BMI.')
parser.add_argument("-w", "--weight", type=int, help='Your weight in kg')
parser.add_argument("-l", "--length", type=int, help='Your length in cm')
parser.parse_args(["-w", '80', "--length", '186'])


try:
    parser.parse_args(["-w", '80', "--length", '186', "--years", "23"])
except SystemExit:
    pass
  
/*out 
usage: argparse_default.py [-h] [-w WEIGHT] [-l LENGTH]
argparse_default.py: error: unrecognized arguments: --years 23
*/

know, unknown = parser.parse_known_args(["-w", '80', "--length", '186', "--years", "23"])
print(know)
print(unknown)

/*out
Namespace(length=186, weight=80)
['--years', '23']
*/

```

### choose a random item

range(start, stop,[step])

randrange 结合了choice, range的功能

```python
>>> from random import choice, randrange
>>> list(range(10,101,10))
[10, 20, 30, 40, 50, 60, 70, 80, 90, 100]
>>> choice(range(10,101,10))
90
>>> choice(range(10,101,10))
20
>>> randrange(10,101,10)
50
>>> randrange(10,101,10)
100
```

### str mapping of replacements

```python
>>> vowels = 'aeiou'
>>> text = "It's Friday evening, which means X-FILES night"
>>> table = {c: c.swapcase() for c in vowels + vowels.upper()}
>>> table
{'a': 'A', 'e': 'E', 'i': 'I', 'o': 'O', 'u': 'U', 'A': 'a', 'E': 'e', 'I': 'i', 'O': 'o', 'U': 'u'}
>>> translation = text.maketrans(table)
# keys are outputs of ord() = integer representation of (Unicode) characters
# so ord('A') -> 65, ord('E') -> 69, etc.
>>> translation
{97: 'A', 101: 'E', 105: 'I', 111: 'O', 117: 'U', 65: 'a', 69: 'e', 73: 'i', 79: 'o', 85: 'u'}

# apply the ranslation mapping, effectively "swapcasing" all vowels
>>> text.translate(translation)
"it's FrIdAy EvEnIng, whIch mEAns X-FiLeS nIght"
# an alternative: using join + a generator expression
>>> "".join(table.get(c, c) for c in text)
"it's FrIdAy EvEnIng, whIch mEAns X-FiLeS nIght"
```

