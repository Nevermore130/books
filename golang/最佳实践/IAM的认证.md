### 如何设计 IAM 项目的认证功能？ ###

将 iam-apiserver 提供的 API 接口注册到 API 网关中，通过 API 网关中的 Token 认证功能，来实现对 iam-apiserver API 接口的认证。有很多 API 网关可供选择，例如腾讯云 API 网关、Tyk、Kong 等

Basic 认证需要用户名和密码，Bearer 认证则需要密钥，所以 iam-apiserver 需要将用户名 / 密码、密钥等信息保存在后端的 MySQL 中，持久存储起来。

为了将控制流和数据流区分开来，密钥的 CURD 操作也放在了 iam-apiserver 中，但是 iam-authz-server 需要用到这些密钥信息。为了解决这个问题，目前的做法是

<img src="../images/image-20240921112422926.png" alt="image-20240921112422926" style="zoom:50%;" />

* iam-authz-server 通过 gRPC API 请求 iam-apiserver，获取所有的密钥信息；
* 当 iam-apiserver 有密钥更新时，会 Pub 一条消息到 Redis Channel 中。因为 iam-authz-server 订阅了同一个 Redis Channel，iam-authz-searver 监听到 channel 有新消息时，会获取、解析消息，并更新它缓存的密钥信息。这样，我们就能确保 iam-authz-server 内存中缓存的密钥和 iam-apiserver 中的密钥保持一致。
* 将所有密钥都缓存在 iam-authz-server 中，那岂不是要占用很大的内存？8G 的内存大概能保存约 8 千万个密钥信息，完全够用。后期不够用的话，可以加大内存。

### IAM 项目是如何实现 Basic 认证的？ ###

* auto 策略：该策略会根据 HTTP 头Authorization: Basic XX.YY.ZZ和Authorization: Bearer XX.YY.ZZ自动选择使用 Basic 认证还是 Bearer 认证。
* basic 策略：该策略实现了 Basic 认证。
* jwt 策略：该策略实现了 Bearer 认证，JWT 是 Bearer 认证的具体实现。

iam-apiserver 通过创建需要的认证策略，并加载到需要认证的 API 路由上，来实现 API 认证

```go
jwtStrategy, _ := newJWTAuth().(auth.JWTStrategy)
g.POST("/login", jwtStrategy.LoginHandler)
g.POST("/logout", jwtStrategy.LogoutHandler)
// Refresh time can be longer than token timeout
g.POST("/refresh", jwtStrategy.RefreshHandler)
```

上述代码中，通过newJWTAuth函数创建了auth.JWTStrategy类型的变量，该变量包含了一些认证相关函数。

* LoginHandler：实现了 Basic 认证，完成登陆认证。
* RefreshHandler：重新刷新 Token 的过期时间。
* LogoutHandler：用户注销时调用。登陆成功后，如果在 Cookie 中设置了认证相关的信息，执行 LogoutHandler 则会清空这些信息。

来看下 LoginHandler Gin 中间件，该函数定义位于github.com/appleboy/gin-jwt包的[auth_jwt.go](https://github.com/appleboy/gin-jwt/blob/v2.6.4/auth_jwt.go#L431)文件中。

```go
func (mw *GinJWTMiddleware) LoginHandler(c *gin.Context) {
  if mw.Authenticator == nil {
    mw.unauthorized(c, http.StatusInternalServerError, mw.HTTPStatusMessageFunc(ErrMissingAuthenticatorFunc, c))
    return
  }

  data, err := mw.Authenticator(c)

  if err != nil {
    mw.unauthorized(c, http.StatusUnauthorized, mw.HTTPStatusMessageFunc(err, c))
    return
  }

  // Create the token
  token := jwt.New(jwt.GetSigningMethod(mw.SigningAlgorithm))
  claims := token.Claims.(jwt.MapClaims)

  if mw.PayloadFunc != nil {
    for key, value := range mw.PayloadFunc(data) {
      claims[key] = value
    }
  }

  expire := mw.TimeFunc().Add(mw.Timeout)
  claims["exp"] = expire.Unix()
  claims["orig_iat"] = mw.TimeFunc().Unix()
  tokenString, err := mw.signedString(token)

  if err != nil {
    mw.unauthorized(c, http.StatusUnauthorized, mw.HTTPStatusMessageFunc(ErrFailedTokenCreation, c))
    return
  }

  // set cookie
  if mw.SendCookie {
    expireCookie := mw.TimeFunc().Add(mw.CookieMaxAge)
    maxage := int(expireCookie.Unix() - mw.TimeFunc().Unix())

    if mw.CookieSameSite != 0 {
      c.SetSameSite(mw.CookieSameSite)
    }

    c.SetCookie(
      mw.CookieName,
      tokenString,
      maxage,
      "/",
      mw.CookieDomain,
      mw.SecureCookie,
      mw.CookieHTTPOnly,
    )
  }

  mw.LoginResponse(c, http.StatusOK, tokenString, expire)
}
```

LoginHandler 函数会执行**Authenticator函数，来完成 Basic 认证。如果认证通过，则会签发 JWT Token**，并执行 PayloadFunc函数设置 Token Payload。如果我们设置了 SendCookie=true ，还会在 Cookie 中添加认证相关的信息，例如 Token、Token 的生命周期等，最后执行 LoginResponse 方法返回 Token 和 Token 的过期时间

**先来看下Authenticator函数。Authenticator 函数从 HTTP Authorization Header 中获取用户名和密码，并校验密码是否合法。**

```go
func authenticator() func(c *gin.Context) (interface{}, error) {
  return func(c *gin.Context) (interface{}, error) {
    var login loginInfo
    var err error

    // support header and body both
    if c.Request.Header.Get("Authorization") != "" {
      login, err = parseWithHeader(c)
    } else {
      login, err = parseWithBody(c)
    }
    if err != nil {
      return "", jwt.ErrFailedAuthentication
    }

    // Get the user information by the login username.
    user, err := store.Client().Users().Get(c, login.Username, metav1.GetOptions{})
    if err != nil {
      log.Errorf("get user information failed: %s", err.Error())

      return "", jwt.ErrFailedAuthentication
    }

    // Compare the login password with the user password.
    if err := user.Compare(login.Password); err != nil {
      return "", jwt.ErrFailedAuthentication
    }

    return user, nil
  }
}
```

Authenticator函数需要获取用户名和密码。它首先会判断是否有Authorization请求头，如果有，则调用parseWithHeader函数获取用户名和密码，否则调用parseWithBody从 Body 中获取用户名和密码。如果都获取失败，则返回认证失败错误。

假设我们的请求为：

```shell
$ curl -XPOST -H"Authorization: Basic YWRtaW46QWRtaW5AMjAyMQ==" http://127.0.0.1:8080/login
# 其中，YWRtaW46QWRtaW5AMjAyMQ==值由以下命令生成
$ echo -n 'admin:Admin@2021'|base64
YWRtaW46QWRtaW5AMjAyMQ==

```

parseWithHeader实际上执行的是上述命令的逆向步骤

1. 获取``Authorization``头的值，并调用 ``strings.SplitN ``函数，获取一个切片变量 auth，其值为 ``["Basic","YWRtaW46QWRtaW5AMjAyMQ=="]`` 。
2. 将``YWRtaW46QWRtaW5AMjAyMQ==``进行 base64 解码，得到``admin:Admin@2021``。
3. 调用``strings.SplitN``函数获取`` admin:Admin@2021`` ，得到用户名为admin，密码为``Admin@2021``。

**再来看下PayloadFunc函数：**

```go
func payloadFunc() func(data interface{}) jwt.MapClaims {
    return func(data interface{}) jwt.MapClaims {
        claims := jwt.MapClaims{
            "iss": APIServerIssuer,
            "aud": APIServerAudience,
        }
        if u, ok := data.(*v1.User); ok {
            claims[jwt.IdentityKey] = u.Name
            claims["sub"] = u.Name
        }

        return claims
    }
}
```

PayloadFunc 函数会设置 JWT Token 中 Payload 部分的 iss、aud、sub、identity 字段，供后面使用。

**LoginResponse 函数：**

```go
func loginResponse() func(c *gin.Context, code int, token string, expire time.Time) {
    return func(c *gin.Context, code int, token string, expire time.Time) {
        c.JSON(http.StatusOK, gin.H{
            "token":  token,
            "expire": expire.Format(time.RFC3339),
        })
    }
}
```

```sh
$ curl -XPOST -H"Authorization: Basic YWRtaW46QWRtaW5AMjAyMQ==" http://127.0.0.1:8080/login
{"expire":"2021-09-29T01:38:49+08:00","token":"XX.YY.ZZ"}
```

登陆成功后，iam-apiserver 会返回 Token 和 Token 的过期时间，前端可以将这些信息缓存在 Cookie 中或 LocalStorage 中，之后的请求都可以使用 Token 来进行认证。使用 Token 进行认证，不仅能够提高认证的安全性，还能够避免查询数据库，从而提高认证效率。



**LogoutHandler**

```go
func (mw *GinJWTMiddleware) LogoutHandler(c *gin.Context) {
    // delete auth cookie
    if mw.SendCookie {
        if mw.CookieSameSite != 0 {
            c.SetSameSite(mw.CookieSameSite)
        }

        c.SetCookie(
            mw.CookieName,
            "",
            -1,
            "/",
            mw.CookieDomain,
            mw.SecureCookie,
            mw.CookieHTTPOnly,
        )
    }

    mw.LogoutResponse(c, http.StatusOK)
}
```

### IAM 项目是如何实现 Bearer 认证的？ ###

iam-authz-server 通过在 /v1 路由分组中加载 cache 认证中间件来使用 cache 认证策略：

```go
auth := newCacheAuth()
apiv1 := g.Group("/v1", auth.AuthFunc())
```

newCacheAuth 函数调用auth.NewCacheStrategy创建了一个 cache 认证策略，创建时传入了getSecretFunc函数，该函数会返回密钥的信息。

```go
func newCacheAuth() middleware.AuthStrategy {
    return auth.NewCacheStrategy(getSecretFunc())
}

func getSecretFunc() func(string) (auth.Secret, error) {
    return func(kid string) (auth.Secret, error) {
        cli, err := store.GetStoreInsOr(nil)
        if err != nil {
            return auth.Secret{}, errors.Wrap(err, "get store instance failed")
        }

        secret, err := cli.GetSecret(kid)
        if err != nil {
            return auth.Secret{}, err
        }

        return auth.Secret{
            Username: secret.Username,
            ID:       secret.SecretId,
            Key:      secret.SecretKey,
            Expires:  secret.Expires,
        }, nil
    }
}
```

密钥信息包含了以下字段：

```go
type Secret struct {
    Username string
    ID       string
    Key      string
    Expires  int64
}
```

来看下 cache 认证策略实现的AuthFunc方法

```go
func (cache CacheStrategy) AuthFunc() gin.HandlerFunc {
  return func(c *gin.Context) {
    header := c.Request.Header.Get("Authorization")
    if len(header) == 0 {
      core.WriteResponse(c, errors.WithCode(code.ErrMissingHeader, "Authorization header cannot be empty."), nil)
      c.Abort()

      return
    }

    var rawJWT string
    // Parse the header to get the token part.
    fmt.Sscanf(header, "Bearer %s", &rawJWT)

    // Use own validation logic, see below
    var secret Secret

    claims := &jwt.MapClaims{}
    // Verify the token
    parsedT, err := jwt.ParseWithClaims(rawJWT, claims, func(token *jwt.Token) (interface{}, error) {
      // Validate the alg is HMAC signature
      if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
        return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
      }

      kid, ok := token.Header["kid"].(string)
      if !ok {
        return nil, ErrMissingKID
      }

      var err error
      secret, err = cache.get(kid)
      if err != nil {
        return nil, ErrMissingSecret
      }

      return []byte(secret.Key), nil
    }, jwt.WithAudience(AuthzAudience))
    if err != nil || !parsedT.Valid {
      core.WriteResponse(c, errors.WithCode(code.ErrSignatureInvalid, err.Error()), nil)
      c.Abort()

      return
    }

    if KeyExpired(secret.Expires) {
      tm := time.Unix(secret.Expires, 0).Format("2006-01-02 15:04:05")
      core.WriteResponse(c, errors.WithCode(code.ErrExpired, "expired at: %s", tm), nil)
      c.Abort()

      return
    }

    c.Set(CtxUsername, secret.Username)
    c.Next()
  }
}

// KeyExpired checks if a key has expired, if the value of user.SessionState.Expires is 0, it will be ignored.
func KeyExpired(expires int64) bool {
  if expires >= 1 {
    return time.Now().After(time.Unix(expires, 0))
  }

  return false
}
```

* 第一步，从 Authorization: Bearer XX.YY.ZZ 请求头中获取 XX.YY.ZZ，XX.YY.ZZ 即为 JWT Token。
* 第二步，调用 github.com/dgrijalva/jwt-go 包提供的 ParseWithClaims 函数，该函数会依次执行下面四步操作。
  * 调用 ParseUnverified 函数
  * 从 Token 中获取第一段 XX，base64 解码后得到 JWT Token 的 ``Header{“alg”:“HS256”,“kid”:“a45yPqUnQ8gljH43jAGQdRo0bXzNLjlU0hxa”,“typ”:“JWT”}``。
  * 从 Token 中获取第二段 YY，base64 解码后得到 JWT Token 的 ``Payload{“aud”:“iam.authz.marmotedu.com”,“exp”:1625104314,“iat”:1625097114,“iss”:“iamctl”,“nbf”:1625097114}``。
  * 根据 Token Header 中的 alg 字段，获取 Token 加密函数。
  * 最终 ParseUnverified 函数会返回 Token 类型的变量，Token 类型包含 Method、Header、Claims、Valid 这些重要字段，这些字段会用于后续的认证步骤中。

调用传入的 keyFunc 获取密钥，这里来看下 keyFunc 的实现：

```go
//jwt.ParseWithClaims 的第三个参数
func(token *jwt.Token) (interface{}, error) {
  // Validate the alg is HMAC signature
  if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
    return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
  }

  kid, ok := token.Header["kid"].(string)
  if !ok {
    return nil, ErrMissingKID
  }

  var err error
  secret, err = cache.get(kid)
  if err != nil {
    return nil, ErrMissingSecret
  }

  return []byte(secret.Key), nil
}
```

可以看到，keyFunc 接受 *Token 类型的变量，并获取 Token Header 中的 kid，kid 即为密钥 ID：secretID。接着，调用 cache.get(kid) 获取密钥 secretKey。**cache.get 函数即为 getSecretFunc，getSecretFunc 函数会根据 kid，从内存中查找密钥信息，密钥信息中包含了 secretKey。**

1. 从 Token 中获取 Signature 签名字符串 ZZ，也即 Token 的第三段
2. 获取到 secretKey 之后，token.Method.Verify 验证 Signature 签名字符串 ZZ，也即 Token 的第三段是否合法
3. token.Method.Verify 实际上是使用了相同的加密算法和相同的 secretKey 加密 XX.YY 字符串
4. 假设加密之后的字符串为 WW，接下来会用 WW 和 ZZ base64 解码后的字符串进行比较，如果相等则认证通过，如果不相等则认证失败。

**调用 KeyExpired，验证 secret 是否过期**。secret 信息中包含过期时间，你只需要拿该过期时间和当前时间对比就行。

设置 HTTP Headerusername: colin。

**总结**：iam-authz-server 通过加载 Gin 中间件的方式，在请求/v1/authz接口时进行访问认证。**因为 Bearer 认证具有过期时间，而且可以在认证字符串中携带更多有用信息，还具有不可逆加密等优点，所以 /v1/authz 采用了 Bearer 认证**，Token 格式采用了 JWT 格式，这也是业界在 API 认证中最受欢迎的认证方式

Bearer 认证需要 secretID 和 secretKey，这些信息会通过 **gRPC 接口调用，从 iam-apisaerver 中获取，并缓存在 iam-authz-server 的内存中**供认证时查询使用。

当请求来临时，iam-authz-server Bearer 认证中间件从 JWT Token 中解析出 Header，并从 Header 的 kid 字段中获取到 secretID，根据 secretID 查找到 secretKey

最后使用 secretKey 加密 JWT Token 的 Header 和 Payload，并与 Signature 部分进行对比。如果相等，则认证通过；如果不等，则认证失败。

### IAM 项目认证功能设计技巧 ###

技巧 1：面向接口编程在使用[NewAutoStrategy](https://github.com/marmotedu/iam/blob/v1.0.0/internal/pkg/middleware/auth/auto.go#L30)函数创建 auto 认证策略时，传入了[middleware.AuthStrategy](https://github.com/marmotedu/iam/blob/v1.0.0/internal/pkg/middleware/auth.go#L12)接口类型的参数，这意味着 Basic 认证和 Bearer 认证都可以有不同的实现，这样后期可以根据需要扩展新的认证方式

技巧 2：使用抽象工厂模式[auth.go](https://github.com/marmotedu/iam/blob/v1.0.0/internal/apiserver/auth.go)文件中，通过 newBasicAuth、newJWTAuth、newAutoAuth 创建认证策略时，返回的都是接口。通过返回接口，可以在不公开内部实现的情况下，让调用者使用你提供的各种认证功能。

技巧 3：使用策略模式在 auto 认证策略中，我们会根据 HTTP 请求头Authorization: XXX X.Y.X中的 XXX 来选择并设置认证策略（Basic 或 Bearer）。具体可以查看AutoStrategy的AuthFunc函数：

```go
func (a AutoStrategy) AuthFunc() gin.HandlerFunc {
  return func(c *gin.Context) {
    operator := middleware.AuthOperator{}
    authHeader := strings.SplitN(c.Request.Header.Get("Authorization"), " ", 2)
        ...
    switch authHeader[0] {
    case "Basic":
      operator.SetStrategy(a.basic)
    case "Bearer":
      operator.SetStrategy(a.jwt)
      // a.JWT.MiddlewareFunc()(c)
    default:
      core.WriteResponse(c, errors.WithCode(code.ErrSignatureInvalid, "unrecognized Authorization header."), nil)
      c.Abort()

      return
    }

    operator.AuthFunc()(c)

    c.Next()
  }
}
```





















