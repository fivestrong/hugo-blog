---
title: PostgresSQL安装与使用
date: 2017-03-23 16:30:33
draft: true
---
### 系统版本
CentOS6.8
### 安装官方yum源
```
yum install https://download.postgresql.org/pub/repos/yum/9.6/redhat/rhel-6-x86_64/pgdg-centos96-9.6-3.noarch.rpm
#查看postgres版本
yum list postgres*
```
### 安装软件
```
yum install postgresql96-server postgresql96-devel
```
### 初始化数据库
```
service postgresql-9.6 initdb
这里使用默认初始化位置：/var/lib/pgsql/9.6/data/
```
### 启动数据库
```
chkconfig postgresql-9.6 on #设置开机自启动，可选
service postgresql-9.6 start
```
### 登录数据库
```
su postgres
psql
```
### 创建数据库
```
CREATE DATABASE villains;
CREATE DATABASE villains ENCODING 'UTF-8' #指定数据库编码，默认为utf-8
```
### 创建用户
```
CREATE USER batman WITH PASSWORD 'Extremly-Secret-Password';  
ALTER user postgres with password 'foobar'; ＃修改密码
```
### 给用户赋予权限
```
ALTER DATABASE villains owner to batman;　#改变数据库所有者
GRANT ALL PRIVILEGES ON DATABASE villains to batman; 

GRANT SELECT ON DATABASE villains to alfred;  #或者仅赋予部分权限
可选权限如下：
SELECT
INSERT
UPDATE
DELETE
RULE
REFERENCES
TRIGGER
CREATE
TEMPORARY
EXECUTE
USAGE
```
```
可以在创建数据库的时候使用一个命令给予用户(create database, create user and grant all privileges)
CREATE DATABASE villains OWNER batman;
```
### 查看用户 使用数据库
```
# \du 查看用户
# \c villains 进入数据库
# \l 查看当前所有数据库
```
### 创建表
```
CREATE TABLE super_villains (id serial PRIMARY KEY, name character varying(100), super_power character varying(100), weakness character varying(100));  
CREATE TABLE equipment (id serial PRIMARY KEY, name character varying(100), status character varying(100), special_move character varying(100)); 
```
### 查看表
```
# \d 查看当前数据库所有表
# \d+ equipment; 查看具体表，以及字段
# SELECT *  FROM information_schema.columns  WHERE table_name = 'equipment';  查询语句
```
### 修改表字段自增
```
#我们创建表的时候id字段添加了serial,系统会为我们自动创建两个序列表equipment_id_seq and super_villains_id_seq
#我们可以将他们绑定到id字段以实现自增(应该默认就绑定了)
#ALTER table equipment alter id set default nextval('equipment_id_seq');  
#ALTER table super_villains alter id set default nextval('super_villains_id_seq');
```
### 插入数据
```
INSERT INTO equipment(name, status, special_move) VALUES('Utility Belt', 'Nice and yellow', 'All kind of cool stuff in your waist'); 
INSERT INTO super_villains(name, super_power, weakness) VALUES('The Joker', 'Extra Crazy', 'Super punch'); 
```
### 查询数据
```
SELECT * from equipment;  
SELECT * from super_villains; 
```

### 配置允许远程登录
```
#编辑/var/lib/pgsql/9.6/data/postgresql.conf
#listen_addresses = 'localhost'  替换为　listen_addresses = '*'  
```
### 配置用户访问权限
```
在配置文件pg_hba.conf中,将peer, ident均改成md5，这样就可以像mysql一样使用用户名密码登录
# "local" is for Unix domain socket connections only
local   all             all                                     md5
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
# IPv6 local connections:
host    all             all             ::1/128                 md5

```
### 命令行登录
```
# psql -U postgres postgres -h 127.0.0.1
-U 指定登录用户 postgres
第二个postgres指明登录的数据库
-h 登录主机地址
```
