### iam-apiserver 使用 ###

用对 secret 资源的 CURD 操作

1. 登录 iam-apiserver，获取 token。
2. 创建一个名为 secret0 的 secret。
3. 获取 secret0 的详细信息。
4. 更新 secret0 的描述。
5. 获取 secret 列表。
6. 删除 secret0。

```go
//登录 iam-apiserver，获取 token：
$ curl -s -XPOST -H"Authorization: Basic `echo -n 'admin:Admin@2021'|base64`" http://127.0.0.1:8080/login | jq -r .token
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2MzUwNTk4NDIsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2MjcyODM4NDIsInN1YiI6ImFkbWluIn0.gTS0n-7njLtpCJ7mvSnct2p3TxNTUQaduNXxqqLwGfI

//便于使用，我们将 token 设置为环境变量：
TOKEN=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2MzUwNTk4NDIsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2MjcyODM4NDIsInN1YiI6ImFkbWluIn0.gTS0n-7njLtpCJ7mvSnct2p3TxNTUQaduNXxqqLwGfI

//创建一个名为 secret0 的 secret：
$ curl -v -XPOST -H "Content-Type: application/json" -H"Authorization: Bearer ${TOKEN}" -d'{"metadata":{"name":"secret0"},"expires":0,"description":"admin secret"}' http://iam.api.marmotedu.com:8080/v1/secrets
* About to connect() to iam.api.marmotedu.com port 8080 (#0)
*   Trying 127.0.0.1...
* Connected to iam.api.marmotedu.com (127.0.0.1) port 8080 (#0)
> POST /v1/secrets HTTP/1.1
> User-Agent: curl/7.29.0
> Host: iam.api.marmotedu.com:8080
> Accept: */*
> Content-Type: application/json
> Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2MzUwNTk4NDIsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2MjcyODM4NDIsInN1YiI6ImFkbWluIn0.gTS0n-7njLtpCJ7mvSnct2p3TxNTUQaduNXxqqLwGfI
> Content-Length: 72
> 
* upload completely sent off: 72 out of 72 bytes
< HTTP/1.1 200 OK
< Content-Type: application/json; charset=utf-8
< X-Request-Id: ff825bea-53de-4020-8e68-4e87574bd1ba
< Date: Mon, 26 Jul 2021 07:20:26 GMT
< Content-Length: 313
< 
* Connection #0 to host iam.api.marmotedu.com left intact
{"metadata":{"id":60,"instanceID":"secret-jedr3e","name":"secret0","createdAt":"2021-07-26T15:20:26.885+08:00","updatedAt":"2021-07-26T15:20:26.907+08:00"},"username":"admin","secretID":"U6CxKs0YVWyOp5GrluychYIRxDmMDFd1mOOD","secretKey":"fubNIn8jLA55ktuuTpXM8Iw5ogdR2mlf","expires":0,"description":"admin secret"}


//请求返回头中返回了X-Request-Id Header，X-Request-Id唯一标识这次请求。如果这次请求失败，就可以将X-Request-Id提供给运维或者开发，通过X-Request-Id定位出失败的请求，进行排障

```

详细的使用方法可以参考 : [test.sh](https://github.com/marmotedu/iam/blob/v1.0.6/scripts/install/test.sh)

顺便介绍下如何测试 IAM 应用中的各个部分。确保 iam-apiserver、iam-authz-server、iam-pump 等服务正常运行后，进入到 IAM 项目的根目录，执行以下命令

```shell
$ ./scripts/install/test.sh iam::test::test # 测试整个IAM应用是否正常运行
$ ./scripts/install/test.sh iam::test::login # 测试登陆接口是否可以正常访问
$ ./scripts/install/test.sh iam::test::user # 测试用户接口是否可以正常访问
$ ./scripts/install/test.sh iam::test::secret # 测试密钥接口是否可以正常访问
$ ./scripts/install/test.sh iam::test::policy # 测试策略接口是否可以正常访问
$ ./scripts/install/test.sh iam::test::apiserver # 测试iam-apiserver服务是否正常运行
$ ./scripts/install/test.sh iam::test::authz # 测试authz接口是否可以正常访问
$ ./scripts/install/test.sh iam::test::authzserver # 测试iam-authz-server服务是否正常运行
$ ./scripts/install/test.sh iam::test::pump # 测试iam-pump是否正常运行
$ ./scripts/install/test.sh iam::test::iamctl # 测试iamctl工具是否可以正常使用
$ ./scripts/install/test.sh iam::test::man # 测试man文件是否正确安装
```

每次发布完 iam-apiserver 后，你可以执行以下命令来完成 iam-apiserver 的冒烟测试

```sh
$ export IAM_APISERVER_HOST=127.0.0.1 # iam-apiserver部署服务器的IP地址
$ export IAM_APISERVER_INSECURE_BIND_PORT=8080 # iam-apiserver HTTP服务的监听端口
$ ./scripts/install/test.sh iam::test::apiserver
```

### iam-apiserver 代码实现 ###

iam-apiserver 服务的 main 函数位于[apiserver.go](https://github.com/marmotedu/iam/blob/v1.0.4/cmd/iam-apiserver/apiserver.go#L18)文件中

Options 配置接管命令行选项，应用配置接管整个应用的配置，HTTP/GRPC 服务配置接管跟 HTTP/GRPC 服务相关的配置。这 3 种配置独立开来，可以解耦命令行选项、应用和应用内的服务，使得这 3 个部分可以独立扩展，又不相互影响。

我们通过```github.com/marmotedu/iam/pkg/app```包的buildCommand方法来构建命令行参数。这里的核心是，通过NewApp函数构建 Application 实例时，传入的Options实现了``Flags() (fss cliflag.NamedFlagSets)``方法，通过 buildCommand 方法中的以下代码，将 option 的 Flag 添加到 cobra 实例的 FlagSet 中：

main函数入口：

```go
func main() {
	rand.Seed(time.Now().UTC().UnixNano())
	if len(os.Getenv("GOMAXPROCS")) == 0 {
		runtime.GOMAXPROCS(runtime.NumCPU())
	}

	apiserver.NewApp("iam-apiserver").Run()
}
```

https://github.com/marmotedu/iam/blob/v1.0.4/internal/apiserver/apiserver.go    apiserver.NewApp

```go

// NewApp creates a App object with default parameters.
func NewApp(basename string) *app.App {
	opts := options.NewOptions()
  //初始化app对象的一些属性，比如options,description,runFunc等
  //其中WithRunFunc配置是将command执行完后的回调配置成 run(opts *options.Options) 方法
  //即cmd.RunE = a.runCommand 中runCommnd方法的配置
	application := app.NewApp("IAM API Server",
		basename,
		app.WithOptions(opts),
		app.WithDescription(commandDesc),
		app.WithDefaultValidArgs(),
		app.WithRunFunc(run(opts)),
	)

	return application
}

func run(opts *options.Options) app.RunFunc {
	return func(basename string) error {
		log.Init(opts.Log)
		defer log.Flush()

		cfg, err := config.CreateConfigFromOptions(opts)
		if err != nil {
			return err
		}

		return Run(cfg)
	}
}


//run.go文件：https://github.com/marmotedu/iam/blob/v1.0.4/internal/apiserver/run.go
// Run runs the specified APIServer. This should never exit.
func Run(cfg *config.Config) error {
	server, err := createAPIServer(cfg)
	if err != nil {
		return err
	}

	return server.PrepareRun().Run()
}
```



https://github.com/marmotedu/iam/blob/v1.0.4/pkg/app/app.go#L157  app.NewApp文件：

```go
//app.go
func NewApp(name string, basename string, opts ...Option) *App {
	a := &App{
		name:     name,
		basename: basename,
	}

	for _, o := range opts {
		o(a)
	}

	a.buildCommand()

	return a
}

//NewAPP 中调用的buildCommand
func (a *App) buildCommand(){
........
	var namedFlagSets cliflag.NamedFlagSets
	if a.options != nil {
		namedFlagSets = a.options.Flags()
		fs := cmd.Flags()
		for _, f := range namedFlagSets.FlagSets {
			fs.AddFlagSet(f)
		}

		usageFmt := "Usage:\n  %s\n"
		cols, _, _ := term.TerminalSize(cmd.OutOrStdout())
		cmd.SetHelpFunc(func(cmd *cobra.Command, args []string) {
			fmt.Fprintf(cmd.OutOrStdout(), "%s\n\n"+usageFmt, cmd.Long, cmd.UseLine())
			cliflag.PrintSections(cmd.OutOrStdout(), namedFlagSets, cols)
		})
		cmd.SetUsageFunc(func(cmd *cobra.Command) error {
			fmt.Fprintf(cmd.OutOrStderr(), usageFmt, cmd.UseLine())
			cliflag.PrintSections(cmd.OutOrStderr(), namedFlagSets, cols)

			return nil
		})
	}
 ........
}

// RunFunc defines the application's startup callback function.
type RunFunc func(basename string) error

// WithRunFunc is used to set the application startup callback function option.
func WithRunFunc(run RunFunc) Option {
	return func(a *App) {
		a.runFunc = run
	}
}
```



```go

// Options runs a iam api server.
type Options struct {
	GenericServerRunOptions *genericoptions.ServerRunOptions       `json:"server"   mapstructure:"server"`
	GRPCOptions             *genericoptions.GRPCOptions            `json:"grpc"     mapstructure:"grpc"`
	InsecureServing         *genericoptions.InsecureServingOptions `json:"insecure" mapstructure:"insecure"`
	SecureServing           *genericoptions.SecureServingOptions   `json:"secure"   mapstructure:"secure"`
	MySQLOptions            *genericoptions.MySQLOptions           `json:"mysql"    mapstructure:"mysql"`
	RedisOptions            *genericoptions.RedisOptions           `json:"redis"    mapstructure:"redis"`
	JwtOptions              *genericoptions.JwtOptions             `json:"jwt"      mapstructure:"jwt"`
	Log                     *log.Options                           `json:"log"      mapstructure:"log"`
	FeatureOptions          *genericoptions.FeatureOptions         `json:"feature"  mapstructure:"feature"`
}

// Flags returns flags for a specific APIServer by section name.
func (o *Options) Flags() (fss cliflag.NamedFlagSets) {
	o.GenericServerRunOptions.AddFlags(fss.FlagSet("generic"))
	o.JwtOptions.AddFlags(fss.FlagSet("jwt"))
	o.GRPCOptions.AddFlags(fss.FlagSet("grpc"))
	o.MySQLOptions.AddFlags(fss.FlagSet("mysql"))
	o.RedisOptions.AddFlags(fss.FlagSet("redis"))
	o.FeatureOptions.AddFlags(fss.FlagSet("features"))
	o.InsecureServing.AddFlags(fss.FlagSet("insecure serving"))
	o.SecureServing.AddFlags(fss.FlagSet("secure serving"))
	o.Log.AddFlags(fss.FlagSet("logs"))

	return fss
}
```

根据应用配置来构建 HTTP/GRPC 服务配置。例如，以下代码根据应用配置，构建了 HTTP 服务器的 Address 参数：

```go
func (s *InsecureServingOptions) ApplyTo(c *server.Config) error {
    c.InsecureServing = &server.InsecureServingInfo{
        Address: net.JoinHostPort(s.BindAddress, strconv.Itoa(s.BindPort)),
    }

    return nil
}
//其中，c *server.Config是 HTTP 服务器的配置，s *InsecureServingOptions是应用配置
```

### iam-apiserver 启动流程设计 ###

<img src="../images/image-20240923201324671.png" alt="image-20240923201324671" style="zoom:50%;" />

* 通过``opts := options.NewOptions()``创建带有默认值的 Options 类型变量 opts。opts 变量作为github.com/marmotedu/iam/pkg/app包的NewApp函数的输入参数，最终在 App 框架中，被来自于命令行参数或配置文件的配置（也可能是二者 Merge 后的配置）所填充，opts 变量中各个字段的值会用来创建应用配置。

* 会注册run函数到 App 框架中。run 函数是 iam-apiserver 的启动函数，里面封装了我们自定义的启动逻辑。run 函数中，首先会初始化日志包，这样我们就可以根据需要，在后面的代码中随时记录日志了。

  ```go
  func run(opts *options.Options) app.RunFunc {
  	return func(basename string) error {
  		log.Init(opts.Log)
  		defer log.Flush()
  
  		cfg, err := config.CreateConfigFromOptions(opts)
  		if err != nil {
  			return err
  		}
  
  		return Run(cfg)
  	}
  }
  ```

* 之后，根据应用配置，创建 HTTP/GRPC 服务器所使用的配置。在创建配置后，会先分别进行配置补全，再使用补全后的配置创建 Web 服务实例，例如：

  ```go
  genericServer, err := genericConfig.Complete().New()
  if err != nil {
      return nil, err
  }
  extraServer, err := extraConfig.complete().New()
  if err != nil {
      return nil, err
  }
  ...
  func (c *ExtraConfig) complete() *completedExtraConfig {
      if c.Addr == "" {
          c.Addr = "127.0.0.1:8081"
      }
  
      return &completedExtraConfig{c}
  }
  ```

  上面的代码中，首先调用Complete/complete函数补全配置，再基于补全后的配置，New 一个 HTTP/GRPC 服务实例。

  这里有个设计技巧：complete函数返回的是一个``*completedExtraConfig类型的实例``，在创建 GRPC 实例时，是调用``completedExtraConfig``结构体提供的New方法，这种设计方法可以确保我们创建的 GRPC 实例一定是基于 complete 之后的配置（completed）。

* 最后，调用PrepareRun方法，进行 HTTP/GRPC 服务器启动前的准备。在准备函数中，我们可以做各种初始化操作，例如初始化数据库，安装业务相关的 Gin 中间件、RESTful API 路由等。

完成 HTTP/GRPC 服务器启动前的准备之后，调用Run方法启动 HTTP/GRPC 服务。在Run方法中，分别启动了 GRPC 和 HTTP 服务。

### iam-apiserver 的 REST API 请求处理流程 ###

* 首先，我们通过 API 调用``HTTP Method HTTP Request Path``请求 iam-apiserver 提供的 RESTful API 接口。
* Gin Web 框架接收到 HTTP 请求之后，会通过认证中间件完成请求的认证，iam-apiserver 提供了 Basic 认证和 Bearer 认证两种认证方式。
* 认证通过后，请求会被我们加载的一系列中间件所处理，例如跨域、RequestID、Dump 等中间件。
* 根据 ``HTTP Method HTTP Request Path``进行路由匹配

### iam-apiserver 代码架构 ###

iam-apiserver 代码架构分为 4 层：模型层（Models）、控制层（Controller）、业务层 （Service）、仓库层（Repository）

<img src="../images/image-20240923202026824.png" alt="image-20240923202026824" style="zoom:50%;" />

**模型层（Models）**

```go
type Secret struct {
  // May add TypeMeta in the future.
  // metav1.TypeMeta `json:",inline"`

  // Standard object's metadata.
  metav1.ObjectMeta `       json:"metadata,omitempty"`
  Username          string `json:"username"           gorm:"column:username"  validate:"omitempty"`
  SecretID          string `json:"secretID"           gorm:"column:secretID"  validate:"omitempty"`
  SecretKey         string `json:"secretKey"          gorm:"column:secretKey" validate:"omitempty"`

  // Required: true
  Expires     int64  `json:"expires"     gorm:"column:expires"     validate:"omitempty"`
  Description string `json:"description" gorm:"column:description" validate:"description"`
}
```

另外，这里的模型既可以作为数据库模型，又可以作为 API 接口的请求模型（入参、出参）。如果我们能够确保创建资源时的属性、资源保存在数据库中的属性、返回资源的属性三者一致，就可以使用同一个模型。通过使用同一个模型，可以使我们的代码更加简洁、易维护，并能提高开发效率。如果这三个属性有差异，你可以另外新建模型来适配。

**仓库层（Repository)**

仓库层用来跟数据库 / 第三方服务进行 CURD 交互，作为应用程序的数据引擎进行应用数据的输入和输出。这里需要注意，仓库层仅对数据库 / 第三方服务执行 CRUD 操作，不封装任何业务逻辑。

将从数据库 / 微服务中获取的数据转换为控制层、业务层能识别的数据结构，将控制层、业务层的数据格式转换为数据库或微服务能识别的数据格式。

```go
type secrets struct {
	db *gorm.DB
}

func newSecrets(ds *datastore) *secrets {
	return &secrets{ds.db}
}

// Get return an secret by the secret identifier.
func (s *secrets) Get(ctx context.Context, username, name string, opts metav1.GetOptions) (*v1.Secret, error) {
	secret := &v1.Secret{}
	err := s.db.Where("username = ? and name= ?", username, name).First(&secret).Error
	if err != nil {
		if errors.Is(err, gorm.ErrRecordNotFound) {
			return nil, errors.WithCode(code.ErrSecretNotFound, err.Error())
		}

		return nil, errors.WithCode(code.ErrDatabase, err.Error())
	}

	return secret, nil
}
```

**业务层 (Service)**

业务层主要用来完成业务逻辑处理，我们可以把所有的业务逻辑处理代码放在业务层。业务层会处理来自控制层的请求，并根据需要请求仓库层完成数据的 CURD 操作

iam-apiserver 的业务层位于internal/apiserver/service目录下。下面是 iam-apiserver 业务层中，用来创建密钥的函数：

```go
// SecretSrv defines functions used to handle secret request.
type SecretSrv interface {
	Create(ctx context.Context, secret *v1.Secret, opts metav1.CreateOptions) error
	Update(ctx context.Context, secret *v1.Secret, opts metav1.UpdateOptions) error
	Delete(ctx context.Context, username, secretID string, opts metav1.DeleteOptions) error
	DeleteCollection(ctx context.Context, username string, secretIDs []string, opts metav1.DeleteOptions) error
	Get(ctx context.Context, username, secretID string, opts metav1.GetOptions) (*v1.Secret, error)
	List(ctx context.Context, username string, opts metav1.ListOptions) (*v1.SecretList, error)
}

type secretService struct {
	store store.Factory
}

var _ SecretSrv = (*secretService)(nil)

func newSecrets(srv *service) *secretService {
	return &secretService{store: srv.store}
}


func (s *secretService) Get(
	ctx context.Context,
	username, secretID string,
	opts metav1.GetOptions,
) (*v1.Secret, error) {
	secret, err := s.store.Secrets().Get(ctx, username, secretID, opts)
	if err != nil {
		return nil, err
	}

	return secret, nil
}
```

service.go

```go
// Service defines functions used to return resource interface.
type Service interface {
	Users() UserSrv
	Secrets() SecretSrv
	Policies() PolicySrv
}

type service struct {
	store store.Factory
}

// NewService returns Service interface.
func NewService(store store.Factory) Service {
	return &service{
		store: store,
	}
}

func (s *service) Users() UserSrv {
	return newUsers(s)
}

func (s *service) Secrets() SecretSrv {
	return newSecrets(s)
}

func (s *service) Policies() PolicySrv {
	return newPolicies(s)
}
```

可以看到，业务层最终请求仓库层的s.store的Get方法

**控制层（Controller）**

iam-apiserver 的控制层位于internal/apiserver/controller目录下。下面是 iam-apiserver 控制层中创建密钥的代码：

```go
func (s *SecretHandler) Create(c *gin.Context) {
  log.L(c).Info("create secret function called.")

  var r v1.Secret

  if err := c.ShouldBindJSON(&r); err != nil {
    core.WriteResponse(c, errors.WithCode(code.ErrBind, err.Error()), nil)

    return
  }

  if errs := r.Validate(); len(errs) != 0 {
    core.WriteResponse(c, errors.WithCode(code.ErrValidation, errs.ToAggregate().Error()), nil)

    return
  }

  username := c.GetString(middleware.UsernameKey)

  secrets, err := s.srv.Secrets().List(c, username, metav1.ListOptions{
    Offset: pointer.ToInt64(0),
    Limit:  pointer.ToInt64(-1),
  })
  if err != nil {
    core.WriteResponse(c, errors.WithCode(code.ErrDatabase, err.Error()), nil)

    return
  }

  if secrets.TotalCount >= maxSecretCount {
    core.WriteResponse(c, errors.WithCode(code.ErrReachMaxCount, "secret count: %d", secrets.TotalCount), nil)

    return
  }

  // must reassign username
  r.Username = username

  if err := s.srv.Secrets().Create(c, &r, metav1.CreateOptions{}); err != nil {
    core.WriteResponse(c, err, nil)

    return
  }

  core.WriteResponse(c, nil, r)
}


//https://github.com/marmotedu/iam/blob/v1.0.3/internal/apiserver/controller/v1/secret/secret.go 文件
// SecretHandler create a secret handler used to handle request for secret resource.
type SecretHandler struct {
	srv   srvv1.Service
	store store.Factory
}

// NewSecretHandler creates a secret handler.
func NewSecretHandler(store store.Factory) *SecretHandler {
	return &SecretHandler{
		srv:   srvv1.NewService(store),
		store: store,
	}
}
```

1. 解析 HTTP 请求参数。
2. 进行参数验证，这里可以添加一些业务性质的参数校验，例如：secrets.TotalCount >= maxSecretCount。
3. 调用业务层``s.srv``的Create方法，完成密钥的创建。
4. 返回 HTTP 请求参数。























