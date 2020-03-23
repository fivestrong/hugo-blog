---
title: "Restful Web Services With Go (5)"
date: 2020-03-19T20:39:52+08:00
tags: ["restful"]
categories: ["golang"]
draft: false
---

学习目标：

- REST API 框架 go-restful
- SQLite3 基础以及CRUD操作
- 使用go-restful 创建 MEtro Rail API
- 通过 Gin 框架创建RESTful API

  

## go-restful 简介

1. 首先安装需要用到的数据库包

   ```shell
   # Ubuntu 
   apt-get install sqlite3 libsqlite3-dev
   # MAC OS X
   brew install sqlite3
   ```

2. 安装 go-restful 包

   ```
   go get github.com/emicklei/go-restful
   ```

接下来我们通过一个实现一个简单的 ping server 来简单的了解一下 go-restful 的使用

1. 创建项目

   ```shell
   mkdir basicExample && cd basicExample
   touch main.go
   go mod init example
   ```

2. 创建一个函数，将服务器时间作为返回响应。

   ```go
   func pingTime(req *restful.Request, resp *restful.Response){
   	// write to the response
   	io.WriteString(resp, fmt.Sprintf("%s", time.Now()))
   }
   ```

3. 主要逻辑

   ```go
   package main
   
   import (
   	"fmt"
   	"io"
   	"net/http"
   	"time"
   
   	"github.com/emicklei/go-restful"
   )
   
   func main() {
   	// Create a web service
   	webservice := new(restful.WebService)
   	
   	// Create a route and attach it to handler in the service
   	webservice.Route(webservice.GET("/ping").To(pingTime))
   
   	// Add the service to application
   	restful.Add(webservice)
   	http.ListenAndServe(":8000", nil)
   }
   ```

4. 运行并验证

   ```shell
   go run main.go
   
   ➜  ~ curl -X GET "http://localhost:8000/ping"
   2020-03-22 14:55:00.553986 +0800 CST m=+24.083065640
   ```

可以看到 go-restful 从创建到添加路由都很简单，并且 http.ListenAdnServer 中没有添加 ServeMux ，框架已经帮我们处理好了。

## SQLite3 基本操作

SQLite3 是一个轻量级，以文件为基础的SQL数据库

Go语言中对SQLite3的调用我们可以使用 go-sqlite3 这个库

```shell
go get github.com/mattn/go-sqlite3
```

接下来我们通过例子，来学习简单使用

1. 创建项目目录

   ```shell
   mkdir sqliteExample && sqliteExample 
   touch main.go
   go mod init example
   ```

2. 通多代码创建数据库

   ```go
   package main
   
   import (
   	"database/sql"
   	"log"
   
   	_ "github.com/mattn/go-sqlite3"
   )
   
   
   // Book is a placeholder for book
   type Book struct {
   	id int 
   	name string 
   	author string
   }
   
   
   func main() {
   	db, err := sql.Open("sqlite3", "./books.db")
   	if err != nil {
   		log.Println(err)
   	}
   
   	// Create table
   	statement, err := db.Prepare("CREATE TABLE IF NOT EXISTS books (id INTEGER PRIMARY KEY, isbn INTEGER, author VARCHAR(64), name VARCHAR(64) NULL)")
   	if err != nil {
   		log.Println("Error in creating table")
   	} else {
   		log.Println("Successfully created table books!")
   	}
   	statement.Exec()
   	dbOperations(db)
   
   }
   ```

   这里首先需要注意的是我们导入了 `go-sqlite3` 但是却忽略了它，真正使用的还是Go自带的 `database/sql`。这相当于Go自带的 sql 是一个接口，我们实际需要的的数据库操作为对它的实现；这样就可以统一数据库操作的接口，方便切换调用不同数据库引擎。

   同时这里用到了数据库创建以及表格操作，使用 exec 方法实际执行操作。

3. 上面还有一个自定义的函数 dbOperations 这是对数据库的一些具体操作，比如创建一本书，读取，更新，删除等一系列操作

   ```go
   func dbOperations (db *sql.DB) {
   	// Create 
   	statement, _ := db.Prepare("INSERT INTO books (name, author, isbn) VALUES (?,?,?)")
   	statement.Exec("A Tale of Two Cities", "Charles Dickens", 140430547)
   	log.Println("Inserted the book into database!")
   
   	// Read 
   	rows, _ := db.Query("SELECT id, name, author FROM books")
   	var tempBook Book
   	for rows.Next() {
   		rows.Scan(&tempBook.id, &tempBook.name, &tempBook.author)
   		log.Printf("ID: %d, Book:%s, Author:%s\n", tempBook.id, tempBook.name, tempBook.author)
   	}
   
   	// Update
   	statement, _ = db.Prepare("update books set name=? where id=?")
   	statement.Exec("双城记", 1)
   	log.Printf("Successfully updated the book in database!")
   
   	// Read Again
   	rows, _ = db.Query("SELECT id, name, author FROM books")
   	var tempBook2 Book
   	for rows.Next() {
   		rows.Scan(&tempBook2.id, &tempBook2.name, &tempBook2.author)
   		log.Printf("ID: %d, Book:%s, Author:%s\n", tempBook2.id, tempBook2.name, tempBook2.author)
   	}
   
   	//Delete
   	statement, _ = db.Prepare("delete from books where id=?")
   	statement.Exec(1)
   	log.Println("Successfully deleted the book in database!")
   }
   ```

4. 运行并验证

   ```shell
   ➜  sqliteExample go run main.go
   2020/03/22 15:54:26 Successfully created table books!
   2020/03/22 15:54:26 Inserted the book into database!
   2020/03/22 15:54:26 ID: 1, Book:A Tale of Two Cities, Author:Charles Dickens
   2020/03/22 15:54:26 Successfully updated the book in database!
   2020/03/22 15:54:26 ID: 1, Book:双城记, Author:Charles Dickens
   2020/03/22 15:54:26 Successfully deleted the book in database!
   ```

从代码中我们看到，每次执行SQL语句都先调用 Prepare 函数，然后再传入需要的值；这样做的好处是避免应传入非法的字符串而导致SQL注入，因为数据库引擎默认已经做了一些防注入的措施。

### 使用go-restful 创建 Metro Rail API 

### API 设计说明

| HTTP verb | Path                                  | Action | Resource |
| --------- | ------------------------------------- | ------ | -------- |
| POST      | /v1/train (details as JSON body)      | Create | Train    |
| POST      | /v1/station (details as JSON body)    | Create | Station  |
| GET       | /v1/train/id                          | Read   | Train    |
| GET       | /v1/station/id                        | Read   | Station  |
| POST      | /v1/schedule (source and destination) | Create | Route    |

除了这些，如果有需要还可以添加 UPDATE 和 DELETE 方法。

### 创建数据库 models

数据库需要创建的表为 `train`, `station`, `route` 。项目目录结构为:

```shell
├── dbutils
│   ├── init-tables.go
│   └── models.go
├── go.mod
└── main.go
```

1. 我们先编写 models.go

```go
package dbutils

const train = `
	CREATE TABLE IF NOT EXISTS train (
		ID INTEGER PRIMARY KEY AUTOINCREMENT,
		DRIVER_NAME VARCHAR(64) NULL,
		OPERATING_STATUS BOOLEAN
	)
`
const station = `
	CREATE TABLE IF NOT EXISTS station (
		ID INTEGER PRIMARY KEY AUTOINCREMENT,
		NAME VARCHAR(64) NULL,
		OPENING_TIME TIME NULL,
		CLOSING_TIME TIME NULL
	)
`

const schedule = `
	CREATE TABLE IF NOT EXISTS schedule (
		ID INTEGER PRIMARY KEY AUTOINCREMENT,
		TRAIN_ID INT, 
		STATION_ID INT,
		ARRIVAL_TIME TIME,
		FOREIGN KEY (TRAIN_ID) REFERENCES train(ID),
		FOREIGN KEY (STATION_ID) REFERENCES station(ID)
	)
`
```

2. 接下来是数据库的初始化操作, 编辑 init-tables.go

   ```go
   package dbutils
   
   import (
   	"database/sql"
   	"log"
   )
   
   func Initialize(dbDriver *sql.DB) {
   	statement, driverError := dbdriver.Prepare(train)
   	if driverError != nil {
   		log.Println(driverError)
   	}
   
   	// Create train table
   	_, statementError := statement.Exec()
   	if statementError != nil {
   		log.Println("Table already exists!")
   	}
   	statement, _ = dbDriver.Prepare(station)
   	statement.Exec()
   	statement, _ = dbDriver.Prepare(schedule)
   	statement.Exec()
   	log.Println("All tables created/initialized successfully!")
   }
   
   ```

   上面的代码创建了三张表，主函数调用的时候需要向函数传入相应的数据库驱动。

3. 接下来编辑主程序 main.go

   ```go
   package main
   
   import (
   	"database/sql"
   	"log"
   	"railapi/dbutils"
   
   	_ "github.com/mattn/go-sqlite3"
   )
   
   func main() {
   	// Connect to Database
   	db, err := sql.Open("sqlite3", "./railapi.db")
   	if err != nil {
   		log.Println("Driver creation failed")
   	}
   
   	// Create tables
   	dbutils.Initialize(db)
   }
   ```

4. 运行程序

   ```shell
   ➜  railAPI go run main.go
   2020/03/22 17:39:06 All tables created/initialized successfully!
   ```

接下来就是根据之前的API设计说明文档来编写路由以及处理函数了

1. 导入必要的包

   ```go
   package main
   
   import (
   	"database/sql"
   	"encoding/json"
   	"log"
   	"net/http"
   	"time"
   
   
   	"github.com/emicklei/go-restful"
   	_ "github.com/mattn/go-sqlite3"
   	"railapi/dbutils"
   )
   ```

2. 创建接收数据库信息的结构体

   ```go
   // DB Driver visible to whole program 
   var DB *sql.DB
   
   // TrainResource is the model for holding rail information
   type TrainResource struct {
   	ID int 
   	DriverName string
   	OperatingStatus bool
   }
   
   // StationResource holds infomation about locations
   type StationResource struct {
   	ID int 
   	Nmae string 
   	OpeningTime time.Time
   	ClosingTime time.Time
   }
   
   // ScheduleResource links both trains and stations
   type ScheduleResource struct {
   	ID int
   	TrainID int 
   	StationID int 
   	ArrivalTime time.Time
   }
   ```

3. 对每个资源编写路由及处理函数操作，并注册到 container 

   ```go
   // Register adds paths and routes to a new service instance
   func (t *TrainResource) Register( container *restful.Container) {
   	ws := new(restful.WebService)
   	ws.Path("/v1/trains").Consumes(restful.MIME_JSON).Produces(restful.MIME_JSON)
   	ws.Route(ws.GET("/{train-id}").To(t.getTrain))
   	ws.Route(ws.POST("").To(t.createTrain))
   	ws.Route(ws.DELETE("/{train-id}").To(t.removeTrain))
   	container.Add(ws)
   }
   ```

   这段代码创建了一个 server 并向它添加了路由与操作函数的对应。这里需要注意的是ws.Path这个方法，请求必须为json格式，如果不是会报错 `415--Media Not Supported`。这里响应也返回了 json 格式，但是也可以自定义为其他比如说 XML。

4. 定义上面提到的路由处理函数

   ```go
   // GET http://localhost:8000/v1/trains/1
   func (t TrainResource) getTrain(request *restful.Request, response *restful.Response){
   	id := request.PathParameter("train-id")
   	err := DB.QueryRow("select ID, DRIVER_NAME, OPERATING_STATUS FROM train where id=?", id).Scan(&t.ID, &t.DriverName, &t.OperatingStatus)
   	if err != nil {
   		log.Println(err)
   		response.AddHeader("Content-type", "text/plain")
   		response.WriteErrorString(http.StatusNotFound, "Train could not be found.")
   	} else {
   		response.WriteEntity(t) // write a struct as JSON to a response
   	}
   }
   
   // POST http://localhost:8000/v1/trains
   func (t TrainResource) createTrain(request *restful.Request, response *restful.Response){
   	log.Println(request.Request.Body)
   	decoder := json.NewDecoder(request.Request.Body)
   	var b TrainResource
   	err := decoder.Decode(&b)
   	log.Println(b.DriverName, b.OperatingStatus)
   
   	// Error handling is obvious here. So omitting...
   	statement, _ := DB.Prepare("insert into train (DRIVER_NAME, OPRATING_STATUS) values (?, ?)")
   	result, err := statement.Exec(b.DriverName, b.OperatingStatus)
   	if err == nil {
   		newID, _ := result.LastInsertId()
   		b.ID = int(newID)
   		response.WriteHeaderAndEntity(http.StatusCreated, b)
   	} else {
   		response.AddHeader("Content-Type", "text/plain")
   		response.WriteErrorString(http.StatusInternalServerError, err.Error())
   	}
   }
   
   // DELETE http://localhost:8000/v1/trains/1
   func (t TrainResource) removeTrain(request *restful.Request, response *restful.Response) {
   	id := request.PathParameter("train-id")
   	statement, _ := DB.Prepare("delete from train where id=?")
   	_, err := statement.Exec(id)
   	if err == nil {
   		response.WriteHeader(http.StatusOK)
   	} else {
   		response.AddHeader("Content-Type", "text/plain")
   		response.WriteErrorString(http.StatusInternalServerError, err.Error())
   	}
   }
   ```

5. 实现主函数

   ```go
   func main() {
   	var err error
   	// Connect to Database
   	DB, err = sql.Open("sqlite3", "./railapi.db")
   	if err != nil {
   		log.Println("Driver creation failed")
   	}
   
   	// Create tables
   	dbutils.Initialize(DB)
   
   	wsContainer := restful.NewContainer()
   	wsContainer.Router(restful.CurlyRouter{})
   	t := TrainResource{}
   	t.Register(wsContainer)
   	log.Printf("start listening on localhost:8000")
   	server := &http.Server{Addr: ":8000", Handler: wsContainer}
   	log.Fatal(server.ListenAndServe())
   }
   ```

6. 运行测试

   ```shell
   go run main.go 
   
   curl -X POST  http://127.0.0.1:8000/v1/trains -H 'cache-control: no-cache' -H 'content-type: application/json' -d '{"driverName": "Veronica", "operatingStatus": true}'
   {
    "ID": 1,
    "DriverName": "Veronica",
    "OperatingStatus": true
   }
   
   curl -X GET "http://localhost:8000/v1/trains/1"
   {
    "ID": 1,
    "DriverName": "Veronica",
    "OperatingStatus": true
   }
   
   curl -X DELETE "http://localhost:8000/v1/trains/1"
   curl -X GET "http://localhost:8000/v1/trains/1"
   Train could not be found.
   ```

这样我们就完成了对 TrainResource 的增删改查操作，类似的剩下的 Station 和 Schedule 表的操作请自行尝试完成。

go-restful 同样提供内置的功能支持 swagger ，用于生成API文档。具体使用可以查看文档：https://github.com/emicklei/go-restful-swagger12



## 使用Gin框架编写RESTful API

Gin-Gonic 框架基于 `httprouter`，这个路由管理器的一大特点就是快。Gin 提供了更高阶的API，这使得创建REST services 更加高效。

安装Gin包的命令是：

```shell
go get -u github.com/gin-gonic/gin
```

接下来通过看看Gin的简单使用：

1. 创建项目，初始化

   ```shell
   mkdir ginExample && cd ginExample
   touch main.go
   go mod init example
   ```

2. 编写 main.go

   ```go
   package main
   
   import (
   	"time"
   
   	"github.com/gin-gonic/gin"
   )
   
   func main() {
   	r := gin.Default()
   	r.GET("/pingTime", func(c *gin.Context) {
   		// JSON serializer is available on gin context
   		c.JSON(200, gin.H{
   			"serverTime": time.Now().UTC(),
   		})
   	})
   	r.Run(":8000")
   }
   ```

3. 运行并检测

   ```shell
   ➜  ginExample go run main.go
   [GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.
   
   [GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
    - using env:	export GIN_MODE=release
    - using code:	gin.SetMode(gin.ReleaseMode)
   
   [GIN-debug] GET    /pingTime                 --> main.main.func1 (3 handlers)
   [GIN-debug] Listening and serving HTTP on :8000
   [GIN] 2020/03/23 - 16:11:32 | 200 |     101.572µs |       127.0.0.1 | GET      "/pingTime"
   
   
   curl http://localhost:8000/pingTime
   {"serverTime":"2020-03-23T08:11:32.719289Z"}
   ```

我们再来写一遍之前的 Metro Rail API 来看看换不同框架是怎么的体验。结构目录跟之前的一样 railAPIGin 目录，包含dbutils包

1. 导入需要使用到的包

   ```GO
   import (
   	"database/sql"
   	"log"
   	"net/http"
   
   
   	"example/dbutils"
   
   	"github.com/gin-gonic/gin"
   	_ "github.com/mattn/go-sqlite3"
   )
   ```

2. 定义数据库驱动，表示 station 的 struct

   ```go
   // DB Driver visible to whole program
   var DB *sql.DB
   
   // StationResource holds information about locations
   type StationResource struct {
   	ID int `json:"id"`
   	Name string `json:"name"`
   	OpeningTime string `json:"opening_time"`
   	ClosingTime string `json:"closing_time"`
   }
   ```

   StationResource 有两个作用，可以接收通过HTTP POST过来的请求数据，第二种是接收来自数据的数据

3. 接下来分别实现基于 GET, POST, DELETE 的路由函数

   ```go
   // Getstation returns the station detail
   func GetStation(c *gin.Context){
   	var station StationResource
   	id := c.Param("station_id")
   	err := DB.QueryRow( 
   		"select ID, NAME, CAST(OPENING_TIME as CHAR), CAST(CLOSING_TIME as CHAR) from station where id=?", 
   		id).Scan(&station.ID, &station.Name, &station.OpeningTime, &station.ClosingTime)
   	if err != nil {
   		log.Println(err)
   		c.JSON(500, gin.H{
   			"error": err.Error(),
   		})
   	} else {
   		c.JSON(200, gin.H{
   			"result": station,
   		})
   	}
   }
   
   // CreateStation handles the POST
   func CreateStation(c *gin.Context){
   	var station StationResource
   	// Parse the body into our resource
   	if err := c.BindJSON(&station); err == nil {
   		// Format Time to Go time format
   		statement, _ := DB.Prepare("insert into station (NAME, OPENING_TIME, CLOSING_TIME) values (?, ?, ?)")
   		result, _ := statement.Exec(station.Name, station.OpeningTime, station.ClosingTime)
   		if err == nil {
   			newID, _ := result.LastInsertId()
   			station.ID = int(newID)
   			c.JSON(http.StatusOK, gin.H{
   				"result": station,
   			})
   		} else {
   			c.String(http.StatusInternalServerError, err.Error())
   		}
   	} else {
   		c.String(http.StatusInternalServerError, err.Error())
   	}
   }
   
   // RemoveStation handles removing of resource
   func RemoveStation(c *gin.Context){
   	id := c.Param("station-id")
   	statement, _ := DB.Prepare("delete from station where id=?")
   	_, err := statement.Exec(id)
   	if err != nil {
   		log.Println(err)
   		c.JSON(500, gin.H{
   			"error": err.Error(),
   		})
   	} else {
   		c.String(http.StatusOK, "")
   	}
   }
   ```

   第一个函数作用于GET请求，其中 `c.Param`用于提取路径参数 station_id，我们用这个ID值去数据库取值；这里的原生SQL语句使用到了 CAST ,它的作用是将 SQL TIME字段转换为Go语言的字符串，否则会报类型错误。

   第二个函数对应POST请求，这需要向数据库插入数据。使用 `c.BindJSON` 方法从 request body 中取出数据，将它加载到传入的参数的结构体中。

   第三个函数对应DELETE请求，使用了SQL语言的 DELETE 语句。

4. 最后完成主函数

   ```go
   func main() {
   	var err error
   	DB, err = sql.Open("sqlite3", "./railapi.db")
   	if err != nil {
   		log.Println("Driver creation failed!")
   	}
   	dbutils.Initialize(DB)
   
   
   	r := gin.Default()
   	// Add routes to REST verbs
   	r.GET("/v1/stations/:station_id", GetStation)
   	r.POST("/v1/stations", CreateStation)
   	r.DELETE("/v1/stations/:station_id", RemoveStation)
   
   
   	r.Run(":8000")
   }
   ```

5. 运行并检测

   ```go
   curl -X POST  http://127.0.0.1:8000/v1/stations -H 'cache-control: no-cache' -H 'content-type: application/json' -d '{"name": "Brooklyn", "opening_time": "8:12:00", "closing_time": "18:23:00"}'
   {"result":{"id":1,"name":"Brooklyn","opening_time":"8:12:00","closing_time":"18:23:00"}}
   
   curl -X GET "http://localhost:8000/v1/stations/1"
   {"result":{"id":1,"name":"Brooklyn","opening_time":"8:12:00","closing_time":"18:23:00"}}
   
    curl -X DELETE "http://localhost:8000/v1/stations/1"
   
   ➜  ginExample go run main.go
   2020/03/23 21:02:43 All tables created/initialized successfully!
   [GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.
   
   [GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
    - using env:	export GIN_MODE=release
    - using code:	gin.SetMode(gin.ReleaseMode)
   
   [GIN-debug] GET    /v1/stations/:station_id  --> main.GetStation (3 handlers)
   [GIN-debug] POST   /v1/stations              --> main.CreateStation (3 handlers)
   [GIN-debug] DELETE /v1/stations/:station_id  --> main.RemoveStation (3 handlers)
   [GIN-debug] Listening and serving HTTP on :8000
   [GIN] 2020/03/23 - 21:05:01 | 200 |     913.033µs |       127.0.0.1 | POST     "/v1/stations"
   [GIN] 2020/03/23 - 21:06:11 | 200 |      176.74µs |       127.0.0.1 | GET      "/v1/stations/1"
   [GIN] 2020/03/23 - 21:07:06 | 200 |      111.08µs |       127.0.0.1 | DELETE   "/v1/stations/1"
   ```



