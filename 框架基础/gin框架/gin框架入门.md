## 介绍

Gin 是一个用 Golang编写的 高性能的web 框架, 由于http路由的优化，速度提高了近 40 倍。 Gin的特点就是封装优雅、API友好。

Gin的一些特性：

- 快速
  基于 Radix 树的路由，小内存占用。没有反射。可预测的 API 性能。
- 支持中间件
  传入的 HTTP 请求可以由一系列中间件和最终操作来处理。 例如：Logger，Authorization，GZIP，最终操作 DB。
- Crash 处理
  Gin 可以 catch 一个发生在 HTTP 请求中的 panic 并 recover 它。这样，你的服务器将始终可用。例如，你可以向 Sentry 报告这个 panic！

- JSON 验证
  Gin 可以解析并验证请求的 JSON，例如检查所需值的存在。
- 路由组
  更好地组织路由。是否需要授权，不同的 API 版本…… 此外，这些组可以无限制地嵌套而不会降低性能。
- 错误管理
  Gin 提供了一种方便的方法来收集 HTTP 请求期间发生的所有错误。最终，中间件可以将它们写入日志文件，数据库并通过网络发送。
- 内置渲染
  Gin 为 JSON，XML 和 HTML 渲染提供了易于使用的 API。
- 可扩展性
  新建一个中间件非常简单。



## 安装

```shell
go get -u github.com/gin-gonic/gin
```

## 简单的http server例子

```go
package main
// 导入gin包
import "github.com/gin-gonic/gin"

// 入口函数
func main() {
    // 初始化一个http服务对象
	r := gin.Default()
        
    // 设置一个get请求的路由，url为/ping, 处理函数（或者叫控制器函数）是一个闭包函数。
	r.GET("/ping", func(c *gin.Context) {
    	// 通过请求上下文对象Context, 直接往客户端返回一个json
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
	
	r.Run() // 监听并在 0.0.0.0:8080 上启动服务
}

```

## 目录结构

> Gin框没有对项目结构做出限制，我们可以根据自己项目需要自行设计。

这里给出一个典型的MVC框架大致的项目结构的例子，大家可以参考下：

```shell
├── conf                    #项目配置文件目录
│   └── config.toml         #大家可以选择自己熟悉的配置文件管理工具包例如：toml、xml等等
├── controllers             #控制器目录，按模块存放控制器（或者叫控制器函数），必要的时候可以继续划分子目录。
│   ├── food.go
│   └── user.go
├── main.go                 #项目入口，这里负责Gin框架的初始化，注册路由信息，关联控制器函数等。
├── models                  #模型目录，负责项目的数据存储部分，例如各个模块的Mysql表的读写模型。
│   ├── food.go
│   └── user.go
├── static                  #静态资源目录，包括Js，css，jpg等等，可以通过Gin框架配置，直接让用户访问。
│   ├── css
│   ├── images
│   └── js
├── logs                    #日志文件目录，主要保存项目运行过程中产生的日志。
└── views                   #视图模板目录，存放各个模块的视图模板，当然有些项目只有api，是不需要视图部分，可以忽略这个目录
    └── index.html
```

## gin框架运行模式

为方便调试，Gin 框架在运行的时候默认是debug模式，在控制台默认会打印出很多调试日志，上线的时候我们需要关闭debug模式，改为release模式。

### 通过环境变量设置

```
export GIN_MODE=release
```

GIN_MODE环境变量，可以设置为debug或者release

### 通过代码设置

```shell
在main函数，初始化gin框架的时候执行下面代码
// 设置 release模式
gin.SetMode(gin.ReleaseMode)
// 或者 设置debug模式
gin.SetMode(gin.DebugMode)
```

