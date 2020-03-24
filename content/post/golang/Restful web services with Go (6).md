---
title: "Restful Web Services With Go (6)"
date: 2020-03-24T09:16:47+08:00
tags: ["restful"]
categories: ["golang"]
draft: true
---

学习目标：

- MongoDB 简介
- 安装并使用MongoDB
- 官方Go语言驱动-mongo-driver
- 使用gorilla/mux MongoDB 创建RESTful API
- 通过索引提高查询性能
- 设计用于物流配送的MongoDB 文档

## MongoDB简介

MongoDB是使用比较广泛的NoSQL数据库，它和传统的关系型数据库有很大的不同，主要区别就是它存储集合和文档。可以把MongoDB中的 collections 理解为关系型数据库中的表，documents 为数据库中的每一行；但是需要了解的是，MongoDB中 collections 之间没有关系。这种设计使得MongoDB能够采用一种叫 Sharding 的技术水平伸缩。MongoDB采用BSON文件格式存储数据，这种格式是一种二进制格式能够方便的操作以及传输。几乎所有的MongoDB客户端都能够在插入数据或者检索数据时将JSON格式与BSON格式互相转换。

MongoDB的数据存储在 document，而所有的 documents 又存储在collection中；这就类似关系型数据库中数据行与表的关系。一个简单的 document 格式如下:

```json
{
	_id: 5, 
	name: 'Star Trek', 
	year: 2009, 
	directors: ['J.J. Abrams'], 
	writers: ['Roberto Orci', 'Alex Kurtzman'], 
	boxOffice: {
		budget:150000000,
		gross:257704099 
	}
}
```

MongoDB相对于关系型数据库的主要优势是:

- 易于建模
- 高效的查询能力
- 文档结构符合现代web程序(JSON)
- 比关系型数据库容易伸缩容量

## MongoDB安装以及使用

Ubuntu

```shell
sudo apt-get update
sudo apt-get install -y mongodb

systemctl start mongod
```

Mac OS X

```shell
brew tap mongodb/brew
brew install mongodb-community

//  run MongoDB as a macOS service
brew services start mongodb/brew/mongodb-community

// run MongoDB manually as a background process
mongod --config /usr/local/etc/mongod.conf

// MongoDB 的安装位置
the configuration file (/usr/local/etc/mongod.conf)
the log directory path (/usr/local/var/log/mongodb)
the data directory path (/usr/local/var/mongodb)
```

可以通过 mongo 进入shell 环境



