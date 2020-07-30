---
title: "Openssl Update for Centos6or7"
date: 2020-07-30T16:45:30+08:00
tags: ["openssl"]
categories: ["linux"]
draft: false
---

学习目标：

- 编译最新版本openssl
- 解决编译软件无法找到openssl lib 库的问题

总所周知，CentOS自带软件包老到不行，编译新的软件好多需要高版本的openssl库，yum包提供的版本很低。所以我们需要手动将依赖升级到最新版，还要解决一些编译中的坑。

1. 安装编译环境

查看当前OpenSSL版本

```shell
OpenSSL 1.0.1e-fips 11 Feb 2013
built on: Wed Aug 14 16:32:19 UTC 2019
platform: linux-x86_64
options:  bn(64,64) md2(int) rc4(16x,int) des(idx,cisc,16,int) idea(int) blowfish(idx)
compiler: gcc -fPIC -DOPENSSL_PIC -DZLIB -DOPENSSL_THREADS -D_REENTRANT -DDSO_DLFCN -DHAVE_DLFCN_H -DKRB5_MIT -m64 -DL_ENDIAN -DTERMIO -Wall -O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector --param=ssp-buffer-size=4 -m64 -mtune=generic -Wa,--noexecstack -DPURIFY -DOPENSSL_IA32_SSE2 -DOPENSSL_BN_ASM_MONT -DOPENSSL_BN_ASM_MONT5 -DOPENSSL_BN_ASM_GF2m -DSHA1_ASM -DSHA256_ASM -DSHA512_ASM -DMD5_ASM -DAES_ASM -DVPAES_ASM -DBSAES_ASM -DWHIRLPOOL_ASM -DGHASH_ASM
OPENSSLDIR: "/etc/pki/tls"
engines:  rdrand dynamic
```

先更新 zlib（提供压缩传输支持）:

```shell
yum install -y zlib
```

安装编译环境：

```shell
sudo yum install @development zlib-devel bzip2 bzip2-devel readline-devel sqlite \
sqlite-devel openssl-devel xz xz-devel libffi-devel findutils
```

2. 编译安装

下载解压

```shell
wget https://www.openssl.org/source/old/1.1.1/openssl-1.1.1f.tar.gz
tar xzvf openssl-1.1.1f.tar.gz
cd openssl-1.1.1f
```

编译安装

```shell
./config --prefix=/usr/local/openssl shared zlib
make
make install 
```

3. 配置动态库

备份旧文件

```shell
mv /usr/bin/openssl /usr/bin/openssl.old
mv /usr/include/openssl /usr/include/openssl.old
```

为新版本建立软链接

```shell
ln -s /usr/local/openssl/bin/openssl /usr/bin/openssl
ln -s /usr/local/openssl/include/openssl /usr/include/openssl
```

找到系统库位置（各种 Linux 版本都不同）

```shell
# CentOS 6 位置：`/usr/lib64/`
-rwxr-xr-x. 1 root root 339232 Oct  9  2018 /usr/lib64/libssl3.so
lrwxrwxrwx. 1 root root     16 May 30 22:15 /usr/lib64/libssl.so -> libssl.so.1.0.1e
lrwxrwxrwx. 1 root root     16 May 30 22:14 /usr/lib64/libssl.so.10 -> libssl.so.1.0.1e
-rwxr-xr-x. 1 root root 446344 Aug 15  2019 /usr/lib64/libssl.so.1.0.1e

lrwxrwxrwx. 1 root root      19 May 30 22:15 /usr/lib64/libcrypto.so -> libcrypto.so.1.0.1e
lrwxrwxrwx. 1 root root      19 May 30 22:14 /usr/lib64/libcrypto.so.10 -> libcrypto.so.1.0.1e
-rwxr-xr-x. 1 root root 1974048 Aug 15  2019 /usr/lib64/libcrypto.so.1.0.1e
```

建立软链接

```shell
# 将软链接指向/usr/local/openssl/lib/   文件
ln -sf /usr/local/openssl/lib/libssl.so /usr/lib64/libssl.so
ln -sf /usr/local/openssl/lib/libssl.so /usr/lib64/libssl.so.10

ln -sf /usr/local/openssl/lib/libcrypto.so /usr/lib64/libcrypto.so
ln -sf /usr/local/openssl/lib/libcrypto.so /usr/lib64/libcrypto.so.10

# 因为我们编译的是最新的1.1.1版本所以还需要链接一下软链接，1.0.*版本可能不用
ln -s /usr/local/openssl/lib/libssl.so /usr/lib64/libssl.so.1.1
ln -s /usr/local/openssl/lib/libcrypto.so /usr/lib64/libcrypto.so.1.1

# 可以通过一下命令查看版本是否正确
strings /usr/local/lib64/libssl.so | grep OpenSSL
```

编辑加载so共享库文件

```shell
vim /etc/ld.so.conf.d/openssl.conf
/usr/local/openssl/lib/

# ldconfig 命令会重建缓存文件 `/etc/ld.so.cache`
ldconfig -v  
```

检测是否成功

```shell
# openssl version -a
OpenSSL 1.1.1f  31 Mar 2020
built on: Thu Jul 30 09:07:13 2020 UTC
platform: linux-x86_64
options:  bn(64,64) rc4(16x,int) des(int) idea(int) blowfish(ptr)
compiler: gcc -fPIC -pthread -m64 -Wa,--noexecstack -Wall -O3 -DOPENSSL_USE_NODELETE -DL_ENDIAN -DOPENSSL_PIC -DOPENSSL_CPUID_OBJ -DOPENSSL_IA32_SSE2 -DOPENSSL_BN_ASM_MONT -DOPENSSL_BN_ASM_MONT5 -DOPENSSL_BN_ASM_GF2m -DSHA1_ASM -DSHA256_ASM -DSHA512_ASM -DKECCAK1600_ASM -DRC4_ASM -DMD5_ASM -DAESNI_ASM -DVPAES_ASM -DGHASH_ASM -DECP_NISTZ256_ASM -DX25519_ASM -DPOLY1305_ASM -DZLIB -DNDEBUG
OPENSSLDIR: "/usr/local/openssl/ssl"
ENGINESDIR: "/usr/local/openssl/lib/engines-1.1"
Seeding source: os-specific
```

大功告成！现在可以编译一下最新版的Python试试是否成功了