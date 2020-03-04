---
title: "Restful Web Services With Go (1)"
date: 2020-03-03T15:40:49+08:00
tags: ["restful"]
categories: ["golang"]
draft: true
---

### web 服务的类型

web服务在不断的演变过程中出现了很多服务类型:

- Simple Object Access Protocol(SOAP) 简单对象访问协议
- Universal Description, Discovery, and Integration(UDDI) 统一描述、发现和集成，基于XML
- Web Services Description Language (WSDL) Web服务描述语言
- Representational State Transfer (REST) 表现层状态转换

#### REST API

相交于基于XML传输臃肿的SOAP协议，REST更加的轻量级。REST形式使得多终端通过HTTP进行互相通信成为可能。

#### REST services 的特性

- 基于客户端架构
- 无状态
- 可缓存
- 以资源方式表现
- 实现方式多种多样

### REST 请求以及状态码

rest 常用请求方式

| REST Verb | Action                                               |
| --------- | ---------------------------------------------------- |
| GET       | Fetches a record or set of resources from the server |
| OPTIONS   | Fetches all available REST operations                |
| POST      | Creates a resource or a new set of resources         |
| PUT       | Updates or replaces the given record                 |
| PATCH     | Modifies the given record                            |
| DELETE    | Delete the given resource                            |

http常见响应状态码

| Status Code Type | Number Range                     | Action                                                       |
| ---------------- | -------------------------------- | ------------------------------------------------------------ |
| Success          | 200-226                          | The 2xx family is used for successful responses.             |
| Error            | 400-499(client), 500-599(server) | The 4xx family is used for indicating client errors.The 5xx is for server failures to process the request. |
| Redirect         | 300-308                          | The 3xx family is for URL redirection.                       |

REST API URI 标准格式

http://HostName/APIEndpoint/?key=value(optianal)

#### 请求方式详解

**GET**

GET方法用于请求服务器资源，该请求可以使用以下两种请求参数方法：

- Query parameters
- Path-based parameters

**路径参数举例**

PayPal是国外常使用的支付方式，它提供用户REST API操作交易信息。

通过GET获取 billing agreement 信息的URI为：/v1/payments/billing-agreements/agreement_id.

这里的agreement_id就是一个关于用户的路径请求参数.服务器根据id信息去billing-agreements下面查找信息。

GET也可以获取一个资源列表，比如获取paypal的交易信息，/v1/payments/billing-agreements/transactions.

**请求参数举例**

请求参数通过关键字来操作匹配资源，比如获取一本书出版日期2020，目录为fiction的书：/v1/books/?category=fiction&publish_date=2020

*建议*：当需要获取基于参数的多个资源，可以使用请求参数来进行特定过滤；当需要特定URI的单个资源可以使用路径参数。（Use Path parameters for a single resource and Query parameters for multiple resources in a GET request.）

**POST, PUT, and PATCH**

POST用于创建资源，比如在书籍API中，通过给定信息创建一本书，/v1/books.

POST请求的JSON 格式为

```json
{"name": "射雕英雄传", "year": 1954, "author": "金庸"}
```

POST会在数据库中写入并提供ID，这样我们就能通过ID来GET数据。事实上 射雕英雄传 出版日期为1957年，如果我们想要更新数据，就要用到PUT请求。

PUT跟POST用法类似，它用于更新已存在资源。假如书的API为/v1/books/1234,PUT提交更新的JSON格式为：

```json
{"name": "射雕英雄传", "year": 1957, "author": "金庸"}
```

可以看到为了更新year字段，我们需要将整个ID为1234书的信息全部更新替换，如果是一个数据量很大的信息，这样更新的效率并不高，于是就有了PATCH请求。

PATCH可以更新现有列数据或者添加新的列数据，比如向ID为1234的书添加ISBN信息，/v1/books/1234

```json
{"isbn": "9787807310822"}
```

**DELETE and OPTIONS**

DELETE请求方法会从数据库删除资源，它的请求不需要任何JSON body。

OPTIONS方法会获取服务器提供的所有请求方法。

**跨域资源共享 Cross-Origin Resource Sharing(CORS)**

OPTIONS请求的一个应用就是跨域资源共享。现代浏览器为了安全都会阻止客户端的跨域请求。比如网站www.foo.com只能请求自身域名的API，如果他要请求其他网站的信息，比如www.bar.com，那么bar.com就需要验证foo.com这个域是否有权限获取资源。

具体请求过程如下：

1. foo.com 向 bar.com 发送OPTIONS请求。
2. bar.com 返回响应头信息中包含 Access-Control-Allow-Origin:http://foo.com。
3. 这样foo.com就可以通过任何方法无限制的访问bar.com的资源了。



