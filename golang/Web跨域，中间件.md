### 为什么会出现跨域

* 出于浏览器的同源策略限制。同源策略（Sameoriginpolicy）是一种约定，它是浏览器最核心也最基本的安全功能，如果缺少了同源策略，则浏览器的正常功能可能都会受到影响。可以说 Web 是构建在同源策略基础之上的，浏览器只是针对同源策略的一种实现。同源策略会阻止一个域的 javascript 脚本和另外一个域的内容进行交互；
* 所谓同源（即指在同一个域）就是两个页面具有相同的协议（protocol），主机（host）和端口号（port）；
* 非同源限制：
  * 无法读取非同源网页的 Cookie、LocalStorage 和 IndexedDB；
  * 无法接触非同源网页的 DOM；
  * 无法向非同源地址发送 AJAX 请求。



### **使用跨域资源共享（CORS）来跨域**

解决跨域问题的方法有多种，例如：CORS、Nginx 反向代理、JSONP 跨域等。在 Go 后端服务开发中，通常可以使用 CORS 来解决跨域问题。



CORS 是一个 W3C 标准，全称是"跨域资源共享"（Cross-origin resource sharing）。它允许浏览器向跨域服务器，发出 AJAX 请求，从而克服了 AJAX 只能同源使用的限制。

在使用 CORS 的时候，HTTP 请求会被划分为两类：简单请求和复杂请求



- **简单请求：** 请求方法是 `GET`、`HEAD` 或者 `POST`，并且 HTTP 请求头中只有 `Accept/Accept-Language/Content-Language/Last-Event-ID/Content-Type` 6 种类型，且 `Content-Type` 只能是 `application/x-www-form-urlencoded`, `multipart/form-data` 或着 `text` / `plain` 中的一个值。简单请求会在发送时自动在 HTTP 请求头加上 `Origin` 字段，来标明当前是哪个源(协议 + 域名 + 端口)，服务端来决定是否放行。
- **复杂请求：** 如果一个请求不是简单请求，则为复杂请求。



**CORS 需要浏览器和服务器同时支持**。目前，所有浏览器都支持该功能。**浏览器一旦发现 AJAX 请求跨源，就会自动添加一些附加的头信息**，**复杂请求还会多出一次附加的预检请求，但用户不会有感觉**。因此，实现 CORS 通信的关键是服务器。只要服务器实现了 CORS 接口，就可以跨源通信（在 Header 中设置：`Access-Control-Allow-Origin`）

### 简单请求的 CORS 跨域处理

对于简单请求，浏览器直接发出 CORS 请求。具体来说，就是在头信息之中，增加一个 `Origin` 字段：

```
origin: https://wetv.vip

//服务器需要处理这个头部，并填充返回头 Access-Control-Allow-Origin：
access-control-allow-origin: https://wetv.vip
```

### 复杂请求的 CORS 跨域处理

复杂请求的 CORS 请求，会在正式通信之前，增加一次 HTTP 查询请求，称为**"预检"**请求（`preflight`）。"预检"请求用的请求方法是`OPTIONS`，表示这个请求是用来询问请求能否安全送出的。



当后端收到预检请求后，可以设置跨域相关 Header 以完成跨域请求。支持的 Header 具体如下表所示：

| 返回头                           | 说明                                                         |
| -------------------------------- | ------------------------------------------------------------ |
| Access-Control-Allow-Origin      | 必选，设置允许访问的域名                                     |
| Access-Control-Allow-Methods     | 必选，逗号分隔的字符串，表明服务器支持的所有跨域请求的方法   |
| Access-Control-Allow-Headers     | 逗号分隔的字符串，表明服务器支持的所有头信息字段，不限于浏览器在"预检"中请求的字段。如果浏览器请求包括 `Access-Control-Request-Headers` 字段，则此字段是必选的 |
| Access-Control-Allow-Credentials | 可选，布尔值，默认是`false`，表示不允许发送 Cookie           |
| Access-Control-Max-Age           | 指定本次预检请求的有效期，单位为秒。可以避免频繁的预检请求   |

预检通过后，浏览器就正常发起请求和响应，流程和简单请求一致。

##### gin实现 ####

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

### Go 程序优雅关停能力实现







