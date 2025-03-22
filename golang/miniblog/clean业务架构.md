### MariaDB 安装和配置

因为 miniblog 项目用到了 MariaDB 数据库来存储数据，而 miniblog 服务在启动时会先尝试连接 MariaDB 数据库

1. 配置安装 MariaDB 需要的 Yum 源。配置命令如下：

```shell
$ sudo tee -a /etc/yum.repos.d/mariadb-10.5.repo <<'EOF'
# MariaDB 10.5 CentOS repository list - created 2020-10-23 01:54 UTC
# http://downloads.mariadb.org/mariadb/repositories/
[mariadb]
name = MariaDB
baseurl = https://mirrors.aliyun.com/mariadb/yum/10.5/centos8-amd64/
module_hotfixes=1
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=0
EOF

```

mac安装：
```shell
brew install mariadb

# 启动MariaDB服务器:
brew services stop mariadb

# 检查MariaDB服务的运行状态:
brew services info mariadb

#启动 MariaDB，并设置开机启动。启动命令如下：
$ sudo systemctl enable mariadb
$ sudo systemctl start mariadb

# 设置 root 初始密码。初始化命令如下：
sudo mysqladmin -uroot password 'miniblog1234'

# 登录数据库并创建 miniblog 用户。创建命令如下：
$ mysql -h127.0.0.1 -P3306 -uroot -p'miniblog1234' # 连接 MariaDB，-h 指定主机，-P 指定监听端口，-u 指定登录用户，-p 指定登录密码
MariaDB [(none)]> grant all on miniblog.* TO miniblog@127.0.0.1 identified by  'miniblog1234';

```

用 `miniblog` 用户登录 MariaDB，创建 `miniblog` 数据库。

```sh
$ cd $WORKSPACE/golang/src/github.com/marmotedu/miniblog
$ mysql -h127.0.0.1 -P3306 -u miniblog -p'miniblog1234'
MariaDB [(none)]> source configs/miniblog.sql; 
MariaDB [miniblog]> use miniblog ; 
Database changed
MariaDB [miniblog]> show tables; 
+--------------------+
| Tables_in_miniblog |
+--------------------+
| post               |
| user               |
+--------------------+
2 rows in set (0.000 sec)

```

根据数据库表生成 Model 文件。

```shell
$ mkdir -p internal/pkg/model
$ cd internal/pkg/model
$ db2struct --gorm --no-json -H 127.0.0.1 -d miniblog -t user --package model --struct UserM -u miniblog -p 'miniblog1234' --target=user.go
$ db2struct --gorm --no-json -H 127.0.0.1 -d miniblog -t post --package model --struct PostM -u miniblog -p 'miniblog1234' --target=post.go

```

为了以后的部署，我们使用 `mysqldump` 将创建数据库和表的 SQL 语句保存在 `configs` 目录下供以后部署使用：

```shell
$ mysqldump -h127.0.0.1 -uminiblog --databases miniblog -p'miniblog1234' --add-drop-database --add-drop-table --add-drop-trigger --add-locks --no-data > configs/miniblog.sql

```

### Store 层代码开发

`internal/miniblog/store/store.go` 文件内容如下

```go
package store

import (
    "sync"

    "gorm.io/gorm"
)

var (
    once sync.Once
    // 全局变量，方便其它包直接调用已初始化好的 S 实例.
    S *datastore
)

// IStore 定义了 Store 层需要实现的方法.
type IStore interface {
    Users() UserStore
}

// datastore 是 IStore 的一个具体实现.
type datastore struct {
    db *gorm.DB
}

// 确保 datastore 实现了 IStore 接口.
var _ IStore = (*datastore)(nil)

// NewStore 创建一个 IStore 类型的实例.
func NewStore(db *gorm.DB) *datastore {
    // 确保 S 只被初始化一次
    once.Do(func() {
        S = &datastore{db}
    })

    return S
}

// Users 返回一个实现了 UserStore 接口的实例.
func (ds *datastore) Users() UserStore {
    return newUsers(ds.db)
}

```

`internal/miniblog/store/user.go` 文件内容如下：

```go
package store

import (
    "context"

    "gorm.io/gorm"

    "github.com/marmotedu/miniblog/internal/pkg/model"
)

// UserStore 定义了 user 模块在 store 层所实现的方法.
type UserStore interface {
    Create(ctx context.Context, user *model.UserM) error
}

// UserStore 接口的实现.
type users struct {
    db *gorm.DB
}

// 确保 users 实现了 UserStore 接口.
var _ UserStore = (*users)(nil)

func newUsers(db *gorm.DB) *users {
    return &users{db}
}

// Create 插入一条 user 记录.
func (u *users) Create(ctx context.Context, user *model.UserM) error {
    return u.db.Create(&user).Error
}

```

在开发阶段，来确保结构体实现了期望的接口：

```go
var _ IStore = (*datastore)(nil)
var _ UserStore = (*users)(nil)

```

使用了工厂方法设计模式，实现了以下风格的代码：

```go
type IStore interface {
    Users() UserStore
}

type UserStore interface {
    Create(ctx context.Context, user *model.UserM) error
}

```

Store 层依赖 `*gorm.DB` 实例，所以接下来，我们还需要创建 `*gorm.DB` 实例。因为创建 `*gorm.DB` 是一个项目无关的，可供第三方项目引用的动作，所以，我们选择将创建方式以包的形式存放在项目根目录下的 `pkg/` 目录下

```go
package db

import (
    "fmt"
    "time"

    "gorm.io/driver/mysql"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"
)

// MySQLOptions 定义 MySQL 数据库的选项.
type MySQLOptions struct {
    Host                  string
    Username              string
    Password              string
    Database              string
    MaxIdleConnections    int
    MaxOpenConnections    int
    MaxConnectionLifeTime time.Duration
    LogLevel              int
}

// DSN 从 MySQLOptions 返回 DSN.
func (o *MySQLOptions) DSN() string {
    return fmt.Sprintf(`%s:%s@tcp(%s)/%s?charset=utf8&parseTime=%t&loc=%s`,
        o.Username,
        o.Password,
        o.Host,
        o.Database,
        true,
        "Local")
}

// NewMySQL 使用给定的选项创建一个新的 gorm 数据库实例.
func NewMySQL(opts *MySQLOptions) (*gorm.DB, error) {
    logLevel := logger.Silent
    if opts.LogLevel != 0 {
        logLevel = logger.LogLevel(opts.LogLevel)
    }
    db, err := gorm.Open(mysql.Open(opts.DSN()), &gorm.Config{
        Logger: logger.Default.LogMode(logLevel),
    })
    if err != nil {
        return nil, err
    }

    sqlDB, err := db.DB()
    if err != nil {
        return nil, err
    }

    // SetMaxOpenConns 设置到数据库的最大打开连接数
    sqlDB.SetMaxOpenConns(opts.MaxOpenConnections)

    // SetConnMaxLifetime 设置连接可重用的最长时间
    sqlDB.SetConnMaxLifetime(opts.MaxConnectionLifeTime)

    // SetMaxIdleConns 设置空闲连接池的最大连接数
    sqlDB.SetMaxIdleConns(opts.MaxIdleConnections)

    return db, nil
}

```

最后，我们需要在 main 流程中初始化 Store 层

```go
    if err := initStore(); err != nil {
        return err
    }

```

### Biz 层代码开发

 [internal/miniblog/biz/biz.go](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fmarmotedu%2Fminiblog%2Fblob%2Ffeature%2Fs16%2Finternal%2Fminiblog%2Fbiz%2Fbiz.go)）

```go
// IBiz 定义了 Biz 层需要实现的方法.
type IBiz interface {
    Users() user.UserBiz
}

// biz 是 IBiz 的一个具体实现.
type biz struct {
    ds store.IStore
}

// 确保 biz 实现了 IBiz 接口.
var _ IBiz = (*biz)(nil)

// NewBiz 创建一个 IBiz 类型的实例.
func NewBiz(ds store.IStore) *biz {
    return &biz{ds: ds}
}

// Users 返回一个实现了 UserBiz 接口的实例.
func (b *biz) Users() user.UserBiz {
    return user.New(b.ds)
}

```

```go
// UserBiz 定义了 user 模块在 biz 层所实现的方法.
type UserBiz interface {
    Create(ctx context.Context, r *v1.CreateUserRequest) error
}

// UserBiz 接口的实现.
type userBiz struct {
    ds store.IStore
}

// 确保 userBiz 实现了 UserBiz 接口.
var _ UserBiz = (*userBiz)(nil)

// New 创建一个实现了 UserBiz 接口的实例.
func New(ds store.IStore) *userBiz {
    return &userBiz{ds: ds}
}

// Create 是 UserBiz 接口中 `Create` 方法的实现.
func (b *userBiz) Create(ctx context.Context, r *v1.CreateUserRequest) error {
    var userM model.UserM
    _ = copier.Copy(&userM, r)

    if err := b.ds.Users().Create(ctx, &userM); err != nil {
        if match, _ := regexp.MatchString("Duplicate entry '.*' for key 'username'", err.Error()); match {
            return errno.ErrUserAlreadyExist
        }

        return err
    }

    return nil
}

```

### Controller 层代码开发

有了 Biz 层的代码，接下来我们就可以开发 Controller 层的代码。Controller 层的开发思路和 Store 层、Biz 层的开发思路保持一致。

`user.go` 文件用来创建 `UserController`：

```go
// UserController 是 user 模块在 Controller 层的实现，用来处理用户模块的请求.
type UserController struct {
    b biz.IBiz
}

// New 创建一个 user controller.
func New(ds store.IStore) *UserController {
    return &UserController{b: biz.NewBiz(ds)}
}

```

`UserController` 具有 `Create` 方法，用来实现 `POST /v1/users` 接口

```go
func (ctrl *UserController) Create(c *gin.Context) {
    log.C(c).Infow("Create user function called")

    var r v1.CreateUserRequest
    if err := c.ShouldBindJSON(&r); err != nil {
        core.WriteResponse(c, errno.ErrBind, nil)

        return
    }

    if _, err := govalidator.ValidateStruct(r); err != nil {
        core.WriteResponse(c, errno.ErrInvalidParameter.SetMessage(err.Error()), nil)

        return
    }

    if err := ctrl.b.Users().Create(c, &r); err != nil {
        core.WriteResponse(c, err, nil)

        return
    }

    core.WriteResponse(c, nil, nil)
}

```

有了 Controller 层的方法之后，我们就可以将该方法绑定到指定的路由上（API 路径上）

```go
// installRouters 安装 miniblog 接口路由.
func installRouters(g *gin.Engine) error {
    // 注册 404 Handler.
    g.NoRoute(func(c *gin.Context) {
        core.WriteResponse(c, errno.ErrPageNotFound, nil)
    })

    // 注册 /healthz handler.
    g.GET("/healthz", func(c *gin.Context) {
        log.C(c).Infow("Healthz function called")

        core.WriteResponse(c, nil, map[string]string{"status": "ok"})
    })

    uc := user.New(store.S)

    // 创建 v1 路由分组
    v1 := g.Group("/v1")
    {
        // 创建 users 路由分组
        userv1 := v1.Group("/users")
        {
            userv1.POST("", uc.Create)
        }
    }

    return nil
}

```

### 编译、启动、测试

就先来创建一个 `root` 用户，密码为 `miniblog1234`。编译并启动服务：

```shell
$ make
$ $ _output/miniblog -c configs/miniblog.yaml


$ curl -XPOST -H"Content-Type: application/json" -d'{"username":"root","password":"miniblog1234","nickname":"root","email":"nosbelm@qq.com","phone":"1818888xxxx"}' http://127.0.0.1:8080/v1/users

```













