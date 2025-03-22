* **对外：** REST + JSON 的组合。因为 API 接口规范、数据格式直观、易懂、开发调试方法，再加上客户端和服务端通过 HTTP 协议通信时，无需使用相同的编程语言，所以 REST + JSON 更适合对外提供 API 接口

- **对内：** RPC + Protobuf 的组合。因为 RPC 协议调用方便，Protobuf 格式数据传输效率更高，从而使得 API 接口性能更好，所以 RPC + Probobuf 的组合更适合对内提供接口。

REST 规范把所有内容都视为资源，网络上一切皆资源。REST 架构对资源的操作包括获取、创建、修改和删除，资源的操作正好对应 HTTP 协议提供的 `GET`、`POST`、`PUT` 和 `DELETE` 方法。HTTP 动词与 REST 风格 CRUD 对应关系：

对资源的操作应该满足安全性和幂等性。

- **安全性：** 不会改变资源状态，可以理解为只读的；
- **幂等性：** 执行 1 次和执行 N 次，对资源状态改变的效果是等价的。

### miniblog 实现一个最简单的 REST Web Server

只需要在 `internal/miniblog/miniblog.go` 文件中的 `run()` 函数中，添加以下代码即可：

```go
// run 函数是实际的业务代码入口函数.
func run() error {
    // 设置 Gin 模式
    gin.SetMode(viper.GetString("runmode"))

    // 创建 Gin 引擎
    g := gin.New()

    // 注册 404 Handler.
    g.NoRoute(func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"code": 10003, "message": "Page not found."})
    })

    // 注册 /healthz handler.
    g.GET("/healthz", func(c *gin.Context) {        c.JSON(http.StatusOK, gin.H{"status": "ok"})
    })
    // 创建 HTTP Server 实例
    httpsrv := &http.Server{Addr: viper.GetString("addr"), Handler: g}

    // 运行 HTTP 服务器
    // 打印一条日志，用来提示 HTTP 服务已经起来，方便排障
    log.Infow("Start to listening the incoming requests on http address", "addr", viper.GetString("addr"))
    if err := httpsrv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
        log.Fatalw(err.Error())
    }

    return nil
}

```

#### 编译并测试

```go
$ make
$ _output/miniblog -c configs/miniblog.yaml
[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:        export GIN_MODE=release
 - using code:        gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /healthz                  --> github.com/marmotedu/miniblog/internal/miniblog.run.func2 (1 handlers)

```

```shell
$ curl http://127.0.0.1:8080/healthz
{"status":"ok"}

```

### 添加中间件（Middleware）

**全局中间件**

```go
router := gin.New()
//一次设置多个中间件 
router.Use(Logger(), Recovery())
//一次设置一个中间件
router.Use(gin.Logger())
router.Use(gin.Recovery())

```

**路由组中间件：路由组中间件仅对该路由组下面的路由起作用。**

```go
 authorized := r.Group("/users", AuthRequired())

```

**单个路由中间件：单个路由中间件仅对一个路由起作用。**

```go
authorized.POST("/login", loginEndpoint)
```

#### 给请求添加 X-Request-ID

如何开发一个 Gin 中间件，将 `X-Request-ID` 注入到请求头中呢？这里，我们先来看下 `r.Use` 方法的入参格式：

```go
Use(middleware ...HandlerFunc) IRoutes

```

所以，我们只需要返回一个 `gin.HandlerFunc` 类型的 function 即可。也可以这么来实现：

```go
package middleware

import (
    "github.com/gin-gonic/gin"    
    "github.com/google/uuid"        
)                      

func RequestID() gin.HandlerFunc {        
    return func(c *gin.Context) {                    
        requestID := c.Request.Header.Get("X-Request-ID")                              

        if requestID == "" {                            
            requestID = uuid.New().String()                                                  
        }                                                              
				// 将 RequestID 保存在 gin.Context 中，方便后边程序使用   
         c.Set("X-Request-ID", requestID)                                                                         // 将 RequestID 保存在 HTTP 返回头中，Header 的键为 `X-Request-ID`
      	c.Writer.Header().Set("X-Request-ID", requestID) 
                                                                                  
    }                                                                                                      
}  

```

#### 在日志中打印 X-Request-ID

```go
// C 解析传入的 context，尝试提取关注的键值，并添加到 zap.Logger 结构化日志中.
func C(ctx context.Context) *zapLogger {                              
    return std.C(ctx)                               
}                                                   
          
func (l *zapLogger) C(ctx context.Context) *zapLogger {                    
    lc := l.clone()                                    
                                                               
    if requestID := ctx.Value(known.XRequestIDKey); requestID != nil {
        lc.z = lc.z.With(zap.Any(known.XRequestIDKey, requestID))
    }                                                                 
                                                    
    return lc                                                                
}                          
                                                                              
// clone 深度拷贝 zapLogger.            
func (l *zapLogger) clone() *zapLogger {                       
    lc := *l           
    return &lc                     
}

```

因为 `log` 包被多个请求并发调用，**为了防止 `X-Request-ID` 污染，针对每一个请求，我们都深拷贝一个 `*zapLogger` 对象，然后再添加 `X-Request-ID`。**

改造 `/healthz` 路由方法，当 `/healthz` 被调用时，打印一条调用日志：

```go
    // 注册 /healthz handler.
    g.GET("/healthz", func(c *gin.Context) {
        log.C(c).Infow("Healthz function called")

        c.JSON(http.StatusOK, gin.H{"status": "ok"})
    })

```

### 跨域功能实现

CORS 是一个 W3C 标准，全称是"跨域资源共享"（Cross-origin resource sharing）。它允许浏览器向跨域服务器，发出 AJAX 请求，从而克服了 AJAX 只能同源使用的限制。例如：当一个请求 URL 的协议、域名、端口三者之间任意一个与当前页面的 URL 不同即为跨域。

简单请求会在发送时自动在 HTTP 请求头加上 `Origin` 字段，来标明当前是哪个源(协议 + 域名 + 端口)，服务端来决定是否放行。

CORS 需要浏览器和服务器同时支持。目前，所有浏览器都支持该功能。浏览器一旦发现 AJAX 请求跨源，就会自动添加一些附加的头信息，复杂请求还会多出一次附加的预检请求，但用户不会有感觉。因此，实现 CORS 通信的关键是服务器。**只要服务器实现了 CORS 接口，就可以跨源通信（在 Header 中设置：`Access-Control-Allow-Origin`）**

对于简单请求，浏览器直接发出 CORS 请求。具体来说，就是在头信息之中，增加一个 `Origin` 字段：

服务器需要处理这个头部，并填充返回头 `Access-Control-Allow-Origin`：

```go
access-control-allow-origin: https://wetv.vip

```

此头部也可填写为 `*`，表示接受任意域名的请求。如果不返回这个头部，**浏览器会抛出跨域错误。**（注：服务器端不会报错）

#### miniblog 跨域功能实现

```go
func Cors(c *gin.Context) {
    if c.Request.Method != "OPTIONS" {
        c.Next()
    } else {
        c.Header("Access-Control-Allow-Origin", "*")
        c.Header("Access-Control-Allow-Methods", "GET,POST,PUT,PATCH,DELETE,OPTIONS")
        c.Header("Access-Control-Allow-Headers", "authorization, origin, content-type, accept")
        c.Header("Allow", "HEAD,GET,POST,PUT,PATCH,DELETE,OPTIONS")
        c.Header("Content-Type", "application/json")
        c.AbortWithStatus(200)
    }
}

```

- 如果 HTTP 请求不是 `OPTIONS` 跨域请求，则继续处理 HTTP 请求；
- 如果 HTTP 请求时 `OPTIONS` 跨域请求，则设置跨域 Header，并返回。

### 添加优雅关停功能

先来说下，为什么要添加优雅关停能力。在应用程序的生命周期中，新功能发布、Bug 修复、配置变更等，都需要重启服务。在服务进程停止的时候，可能需要做一些处理工作，例如:

1. 正在执行的 HTTP 请求，要等待请求执行完并返回，否则该请求会报错，并产生一些脏数据；
2. 异步处理任务，也需要将缓存中的数据处理完成，否则可能会造成一些数据丢失或者不一致；
3. 关闭数据库连接，否则数据库连接池会保存一个无用的连接，造成宝贵的连接资源浪费。

我们通过给应用发送标准的系统信号来终止服务进程，例如最常见的 3 种终止进程的方式如下：

- `CTRL + C` 实际是发送 `SIGINT` 信号；
- `kill <pid>` 命令向指定的进程发送 `SIGTERM` 信号；
- `kill -9 <pid>` 命令向指定进程发送 `SIGKILL` 信号，`SIGKILL` 信号既不能被应用程序捕获，也不能被阻塞或忽略。

Go 提供 `os/signal` 包可以用来监听并反馈收到的信号。所以，我们基于以上思路，使用 `os/signal` 即可实现优雅关停功能

```go
      // 创建 HTTP Server 实例
    httpsrv := &http.Server{Addr: viper.GetString("addr"), Handler: g}

    // 运行 HTTP 服务器
    // 打印一条日志，用来提示 HTTP 服务已经起来，方便排障
    log.Infow("Start to listening the incoming requests on http address", "addr", viper.GetString("addr"))
    go func() {
        if err := httpsrv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            log.Fatalw(err.Error())
        }
    }()

    // 等待中断信号优雅地关闭服务器（10 秒超时)。
    quit := make(chan os.Signal, 1)
    // kill 默认会发送 syscall.SIGTERM 信号
    // kill -2 发送 syscall.SIGINT 信号，我们常用的 CTRL + C 就是触发系统 SIGINT 信号
    // kill -9 发送 syscall.SIGKILL 信号，但是不能被捕获，所以不需要添加它
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM) // 此处不会阻塞
    <-quit                                               // 阻塞在此，当接收到上述两种信号时才会往下执行
    log.Infow("Shutting down server ...")

    // 创建 ctx 用于通知服务器 goroutine, 它有 10 秒时间完成当前正在处理的请求
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    // 10 秒内优雅关闭服务（将未处理完的请求处理完再关闭服务），超过 10 秒就超时退出
    if err := httpsrv.Shutdown(ctx); err != nil {
        log.Errorw("Insecure Server forced to shutdown", "err", err)
        return err
    }

    log.Infow("Server exiting")

    return nil


```

1. 如果系统收到 `SIGINT` 和 `SIGTERM` 信号，就会往 `quit` channel 中写入一条 `os.Signal` 类型的数据；
2. `quit` 读取到数据，解除阻塞状态，执行后续的清理工作。清理工作执行完成后，正常终止进程。清理工作根据业务逻辑可以执行不同的清理函数

### 测试优雅关停功能

```sh
$ kill `pgrep miniblog`
```



















