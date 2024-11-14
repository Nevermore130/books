iam-authz-server 目前的唯一功能，是通过提供 /v1/authz RESTful API 接口完成资源授权。 /v1/authz 接口是通过[github.com/ory/ladongithub.com/ory/ladon](https://github.com/ory/ladon/blob/master/manager.go)来完成资源授权的。

### github.com/ory/ladon 包介绍 ###

Ladon 是用 Go 语言编写的用于实现访问控制策略的库，类似于 RBAC（基于角色的访问控制系统，Role Based Access Control）和 ACL（访问控制列表，Access Control Lists）

Ladon 解决了这个问题：在特定的条件下，谁能够 / 不能够对哪些资源做哪些操作。为了解决这个问题，Ladon 引入了授权策略。授权策略是一个有语法规范的文档，这个文档描述了谁在什么条件下能够对哪些资源做哪些操作

```json
{
  "description": "One policy to rule them all.",
  "subjects": ["users:<peter|ken>", "users:maria", "groups:admins"],
  "actions" : ["delete", "<create|update>"],
  "effect": "allow",
  "resources": [
    "resources:articles:<.*>",
    "resources:printer"
  ],
  "conditions": {
    "remoteIP": {
        "type": "CIDRCondition",
        "options": {
            "cidr": "192.168.0.1/16"
        }
    }
  }
}
```

核心元素包括主题（Subject）、操作（Action）、效力（Effect）、资源（Resource）以及生效条件（Condition）

有了授权策略，我们就可以传入请求上下文，由 Ladon 来决定请求是否能通过授权。下面是一个请求示例：

```json
{
  "subject": "users:peter",
  "action" : "delete",
  "resource": "resources:articles:ladon-introduction",
  "context": {
    "remoteIP": "192.168.0.5"
  }
}
```



另外，Ladon 还支持授权审计，用来记录授权历史。我们可以通过在 ladon.Ladon 中附加一个 ladon.AuditLogger 来实现：

```go
import "github.com/ory/ladon"
import manager "github.com/ory/ladon/manager/memory"

func main() {

    warden := ladon.Ladon{
        Manager: manager.NewMemoryManager(),
        AuditLogger: &ladon.AuditLoggerInfo{}
    }

    // ...
}
```

在上面的示例中，我们提供了`` ladon.AuditLoggerInfo``，该 AuditLogger 会在授权时打印调用的策略到标准错误。AuditLogger 是一个 interface：

```go
//https://github.com/ory/ladon/blob/master/audit_logger.go

// AuditLogger tracks denied and granted authorizations.
type AuditLogger interface {
    LogRejectedAccessRequest(request *Request, pool Policies, deciders Policies)
    LogGrantedAccessRequest(request *Request, pool Policies, deciders Policies)
}
```

要实现一个新的 AuditLogger，你只需要实现 AuditLogger 接口就可以了。比如，我们可以实现一个 AuditLogger，将授权日志保存到 Redis 或者 MySQL 中。

Ladon 支持跟踪一些授权指标，比如 deny、allow、not match、error。你可以通过实现 ladon.Metric 接口，来对这些指标进行处理。ladon.Metric 接口定义如下：

```go
// Metric is used to expose metrics about authz
type Metric interface {
    // RequestDeniedBy is called when we get explicit deny by policy
    RequestDeniedBy(Request, Policy)
    // RequestAllowedBy is called when a matching policy has been found.
    RequestAllowedBy(Request, Policies)
    // RequestNoMatch is called when no policy has matched our request
    RequestNoMatch(Request)
    // RequestProcessingError is called when unexpected error occured
    RequestProcessingError(Request, Policy, error)
}
```

例如，你可以通过下面的示例，将这些指标暴露给 prometheus：

```go
type prometheusMetrics struct{}

func (mtr *prometheusMetrics) RequestDeniedBy(r ladon.Request, p ladon.Policy) {}
func (mtr *prometheusMetrics) RequestAllowedBy(r ladon.Request, policies ladon.Policies) {}
func (mtr *prometheusMetrics) RequestNoMatch(r ladon.Request) {}
func (mtr *prometheusMetrics) RequestProcessingError(r ladon.Request, err error) {}

func main() {

    warden := ladon.Ladon{
        Manager: manager.NewMemoryManager(),
        Metric:  &prometheusMetrics{},
    }

    // ...
}
```

### iam-authz-server 使用方法介绍 ###

1. 登陆 iam-apiserver 系统，获取访问令牌：

```shell
$ token=`curl -s -XPOST -H'Content-Type: application/json' -d'{"username":"admin","password":"Admin@2021"}' http://127.0.0.1:8080/login | jq -r .token`
```

2. 创建授权策略：

```shell
$ curl -s -XPOST -H"Content-Type: application/json" -H"Authorization: Bearer $token" -d'{"metadata":{"name":"authztest"},"policy":{"description":"One policy to rule them all.","subjects":["users:<peter|ken>","users:maria","groups:admins"],"actions":["delete","<create|update>"],"effect":"allow","resources":["resources:articles:<.*>","resources:printer"],"conditions":{"remoteIP":{"type":"CIDRCondition","options":{"cidr":"192.168.0.1/16"}}}}}' http://127.0.0.1:8080/v1/policies
```

3. 创建密钥，并从请求结果中提取 secretID 和 secretKey：

```shell
$ curl -s -XPOST -H"Content-Type: application/json" -H"Authorization: Bearer $token" -d'{"metadata":{"name":"authztest"},"expires":0,"description":"admin secret"}' http://127.0.0.1:8080/v1/secrets
{"metadata":{"id":23,"name":"authztest","createdAt":"2021-04-08T07:24:50.071671422+08:00","updatedAt":"2021-04-08T07:24:50.071671422+08:00"},"username":"admin","secretID":"ZuxvXNfG08BdEMqkTaP41L2DLArlE6Jpqoox","secretKey":"7Sfa5EfAPIwcTLGCfSvqLf0zZGCjF3l8","expires":0,"description":"admin secret"}
```

iamctl 提供了 jwt sigin 子命令，可以根据 secretID 和 secretKey 签发 Token，方便使用。

```shell
$ iamctl jwt sign ZuxvXNfG08BdEMqkTaP41L2DLArlE6Jpqoox 7Sfa5EfAPIwcTLGCfSvqLf0zZGCjF3l8 # iamctl jwt sign $secretID $secretKey
eyJhbGciOiJIUzI1NiIsImtpZCI6Ilp1eHZYTmZHMDhCZEVNcWtUYVA0MUwyRExBcmxFNkpwcW9veCIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXV0aHoubWFybW90ZWR1LmNvbSIsImV4cCI6MTYxNzg0NTE5NSwiaWF0IjoxNjE3ODM3OTk1LCJpc3MiOiJpYW1jdGwiLCJuYmYiOjE2MTc4Mzc5OTV9.za9yLM7lHVabPAlVQLCqXEaf8sTU6sodAsMXnmpXjMQ
```

可以通过 iamctl jwt show 来查看 Token 的内容：

```go
$ iamctl jwt show eyJhbGciOiJIUzI1NiIsImtpZCI6Ilp1eHZYTmZHMDhCZEVNcWtUYVA0MUwyRExBcmxFNkpwcW9veCIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXV0aHoubWFybW90ZWR1LmNvbSIsImV4cCI6MTYxNzg0NTE5NSwiaWF0IjoxNjE3ODM3OTk1LCJpc3MiOiJpYW1jdGwiLCJuYmYiOjE2MTc4Mzc5OTV9.za9yLM7lHVabPAlVQLCqXEaf8sTU6sodAsMXnmpXjMQ
Header:
{
    "alg": "HS256",
    "kid": "ZuxvXNfG08BdEMqkTaP41L2DLArlE6Jpqoox",
    "typ": "JWT"
}
Claims:
{
    "aud": "iam.authz.marmotedu.com",
    "exp": 1617845195,
    "iat": 1617837995,
    "iss": "iamctl",
    "nbf": 1617837995
}
```

4. 调用/v1/authz接口，完成资源授权请求(调用iam-authz-server)。

```shell
$ curl -s -XPOST -H'Content-Type: application/json' -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsImtpZCI6Ilp1eHZYTmZHMDhCZEVNcWtUYVA0MUwyRExBcmxFNkpwcW9veCIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXV0aHoubWFybW90ZWR1LmNvbSIsImV4cCI6MTYxNzg0NTE5NSwiaWF0IjoxNjE3ODM3OTk1LCJpc3MiOiJpYW1jdGwiLCJuYmYiOjE2MTc4Mzc5OTV9.za9yLM7lHVabPAlVQLCqXEaf8sTU6sodAsMXnmpXjMQ' -d'{"subject":"users:maria","action":"delete","resource":"resources:articles:ladon-introduction","context":{"remoteIP":"192.168.0.5"}}' http://127.0.0.1:9090/v1/authz
{"allowed":true}
```

### iam-authz-server 关键代码分析 ###

调用 iam-authz-server 的 /v1/authz API 接口，实现资源的访问授权。 /v1/authz 对应的 controller 方法是[Authorize](https://github.com/marmotedu/iam/blob/v1.0.4/internal/authzserver/controller/v1/authorize/authorize.go#L33)

```go
func (a *AuthzController) Authorize(c *gin.Context) {
  var r ladon.Request
  if err := c.ShouldBind(&r); err != nil {
    core.WriteResponse(c, errors.WithCode(code.ErrBind, err.Error()), nil)

    return
  }

  //获取Authorizer 对象 包含ladon.Warden类型的对象
  auth := authorization.NewAuthorizer(authorizer.NewAuthorization(a.store))
  if r.Context == nil {
    r.Context = ladon.Context{}
  }

  r.Context["username"] = c.GetString("username")
  
  //调用auth对象的Authorize方法：
  //1. 获取Authorizer中的warden对象 调用warden.IsAllowed(request)
  //2. IsAllowed会调用FindRequestCandidates
  rsp := auth.Authorize(&r)

  core.WriteResponse(c, nil, rsp)
}
```

* 在 Authorize 方法中调用 c.ShouldBind(&r) ，将 API 请求参数解析到 ladon.Request 类型的结构体变量中。
* authorizer.go  中 的NewAuthorization(a.store)  返回AuthorizationInterface对象，**a.store 实现了PolicyGetter 接口**
* 调用authorization.NewAuthorizer函数，该函数会创建并返回包含 Manager 和 AuditLogger 字段的Authorizer类型的变量。**并将AuthorizationInterface对象放到Manager中**

```go
//https://github.com/marmotedu/iam/blob/v1.0.4/internal/authzserver/authorization/authorizer/authorizer.go 文件

// PolicyGetter defines function to get policy for a given user.
type PolicyGetter interface {
	GetPolicy(key string) ([]*ladon.DefaultPolicy, error)
}

// Authorization implements authorization.AuthorizationInterface interface.
type Authorization struct {
	getter PolicyGetter
}

// NewAuthorization create a new Authorization instance.
func NewAuthorization(getter PolicyGetter) authorization.AuthorizationInterface {
	return &Authorization{getter}
}


// Get returns the policy detail by the given identifier.
// Return nil because we use mysql storage to store the policy.
func (auth *Authorization) Get(id string) (*ladon.DefaultPolicy, error) {
	return nil, nil
}

// List returns all the policies under the username.
func (auth *Authorization) List(username string) ([]*ladon.DefaultPolicy, error) {
	return auth.getter.GetPolicy(username)
}
```

```package authorization```   中 对于Controller 中 NewAuthorizer(authorizationClient AuthorizationInterface) 的实现

```go
type Authorizer struct {
	warden ladon.Warden
}

// NewAuthorizer creates a local repository authorizer and returns it.
func NewAuthorizer(authorizationClient AuthorizationInterface) *Authorizer {
	return &Authorizer{
		warden: &ladon.Ladon{
			Manager:     NewPolicyManager(authorizationClient),
			AuditLogger: NewAuditLogger(authorizationClient),
		},
	}
}

// AuthorizationInterface defiens the CURD method for lady policy.
type AuthorizationInterface interface {
	Create(*ladon.DefaultPolicy) error
	Update(*ladon.DefaultPolicy) error
	Delete(id string) error
	DeleteCollection(idList []string) error
	Get(id string) (*ladon.DefaultPolicy, error)
	List(username string) ([]*ladon.DefaultPolicy, error)

	// The following two functions tracks denied and granted authorizations.
	LogRejectedAccessRequest(request *ladon.Request, pool ladon.Policies, deciders ladon.Policies)
	LogGrantedAccessRequest(request *ladon.Request, pool ladon.Policies, deciders ladon.Policies)
}

//Manager 包含一些函数，比如 Create、Update 和 FindRequestCandidates 等，用来对授权策略进行增删改查。AuditLogger 包含 LogRejectedAccessRequest 和 LogGrantedAccessRequest 函数，分别用来记录被拒绝的授权请求和被允许的授权请求，将其作为审计数据使用

//Controller 调用auth.Authorize函数，对请求进行访问授权
func (a *Authorizer) Authorize(request *ladon.Request) *authzv1.Response {
  log.Debug("authorize request", log.Any("request", request))

  if err := a.warden.IsAllowed(request); err != nil {
    return &authzv1.Response{
      Denied: true,
      Reason: err.Error(),
    }
  }

  return &authzv1.Response{
    Allowed: true,
  }
}
```

* Controller 调用auth.Authorize函数，对请求进行访问授权。

该函数会调用 ``a.warden.IsAllowed(request) ``完成资源访问授权。IsAllowed 函数会调用`` FindRequestCandidates(r) ``查询所有的策略列表，这里要注意，我们只需要查询请求用户的 policy 列表。在 Authorize 函数中，我们将 username 存入 ladon Request 的 context 中：

[iam](https://github.com/marmotedu/iam/tree/v1.0.4)/[internal](https://github.com/marmotedu/iam/tree/v1.0.4/internal)/[authzserver](https://github.com/marmotedu/iam/tree/v1.0.4/internal/authzserver)/[authorization](https://github.com/marmotedu/iam/tree/v1.0.4/internal/authzserver/authorization)  /manager.go

```go
import (
	"github.com/marmotedu/errors"
	"github.com/ory/ladon"
)

// PolicyManager is a mysql implementation for Manager to store
// policies persistently.
type PolicyManager struct {
	client AuthorizationInterface
}

// NewPolicyManager initializes a new PolicyManager for given apimachinery api
// client.
func NewPolicyManager(client AuthorizationInterface) ladon.Manager {
	return &PolicyManager{
		client: client,
	}
}
```

在FindRequestCandidates函数中，我们可以从 Request 中取出 username，并根据 username 查询缓存中的 policy 列表，FindRequestCandidates 实现如下：

```go
func (m *PolicyManager) FindRequestCandidates(r *ladon.Request) (ladon.Policies, error) {
    username := ""
  
    if user, ok := r.Context["username"].(string); ok {
      username = user
    }
  
  	//PolicyManager 是我们自己实现的 基于ladon 库的Manager对象，其中实现了FindRequestCandidates
    // FindRequestCandidates 中会获取我们自己的client并调用实现了client接口的List方法
  // 这个client实际上是之前controller中 authorizer.NewAuthorization(a.store) 返回的AuthorizationInterface
    policies, err := m.client.List(username)
    if err != nil {
      return nil, errors.Wrap(err, "list policies failed")
    }
  
    ret := make([]ladon.Policy, 0, len(policies))
    for _, policy := range policies {
      ret = append(ret, policy)
    }
  
    return ret, nil
  }
```

* PolicyManager 是我们自己实现的 基于ladon 库的Manager对象，其中实现了FindRequestCandidates
* FindRequestCandidates 中会获取我们自己的client并调用实现了client接口的List方法
* ``m.client.List(username)``这个client实际上是之前controller中 authorizer.NewAuthorization(a.store) 返回的AuthorizationInterface
* ``client.list`` 调用了 Authorization 实现的 List方法：

```go
func (auth *Authorization) List(username string) ([]*ladon.DefaultPolicy, error) {
	return auth.getter.GetPolicy(username)
}
```

* ``auth.getter ``是controller中``a.store``对象 调用 GetPolicy

再来看下controller的初始化

internal/authzserver/router.go

```go
func installController(g *gin.Engine) *gin.Engine {
	auth := newCacheAuth()
	g.NoRoute(auth.AuthFunc(), func(c *gin.Context) {
		core.WriteResponse(c, errors.WithCode(code.ErrPageNotFound, "page not found."), nil)
	})

	storeIns, _ := store.GetStoreInsOr(nil)
	apiv1 := g.Group("/v1", auth.AuthFunc())
	{
		authzController := authorize.NewAuthzController(storeIns)

		// Router for authorization
		apiv1.POST("/authz", authzController.Authorize)
	}

	return g
}
```

storeIns 的初始化 ```internal/authzserver/server.go``

```go
func (s *authzServer) initialize() error {
	ctx, cancel := context.WithCancel(context.Background())

	// keep redis connected
	go storage.ConnectToRedis(ctx, s.buildStorageConfig())

	storeIns, err := store.GetStoreInsOr(store.GetGRPCClientOrDie(s.rpcServer, s.clientCA))
	if err != nil {
		return errors.Wrap(err, "get store instance failed")
	}
	// cron to reload all secrets and policies from iam-apiserver
	load.NewLoader(ctx, storeIns).Start()
	......

	return nil
}
```

真正的store实现的GetPolicy   ``internal/authzserver/store/store.go``

```go
// GetPolicy return user's ladon policies for the given user.
func (s *Store) GetPolicy(key string) ([]*ladon.DefaultPolicy, error) {
	s.lock.Lock()
	defer s.lock.Unlock()

	value, ok := s.policies.Get(key)
	if !ok {
		return nil, ErrPolicyNotFound
	}

	return value.([]*ladon.DefaultPolicy), nil
}

// Store is used to store secrets and policies.
type Store struct {
	lock     *sync.RWMutex
	cli      StoreClient
	secrets  *ristretto.Cache
	policies *ristretto.Cache
}

// StoreClient defines functions used to get all secrets and policies.
type StoreClient interface {
	GetSecrets() (map[string]*pb.SecretInfo, error)
	GetPolicies() (map[string][]*ladon.DefaultPolicy, error)
}
```

```internal/authzserver/server.go```

*  在服务初始化的时候调用NewLoader初始化一个loader对象
* loader对象Start()    会从iam-apiserver 获取 secrets and policies 数据 放到缓存中，以便Store对象去GetPolicy 的时候可以从缓存获取数据

```go
func (s *authzServer) initialize() error {
	ctx, cancel := context.WithCancel(context.Background())

	// keep redis connected
	go storage.ConnectToRedis(ctx, s.buildStorageConfig())

	storeIns, err := store.GetStoreInsOr(store.GetGRPCClientOrDie(s.rpcServer, s.clientCA))
	if err != nil {
		return errors.Wrap(err, "get store instance failed")
	}
	// cron to reload all secrets and policies from iam-apiserver
	load.NewLoader(ctx, storeIns).Start()

	// start analytics service
	if s.analyticsOptions.Enable {
		analyticsStore := storage.RedisCluster{KeyPrefix: RedisKeyPrefix}
		analyticsIns := analytics.NewAnalytics(s.analyticsOptions, &analyticsStore)
		analyticsIns.Start()
		s.gs.AddShutdownCallback(shutdown.ShutdownFunc(func(string) error {
			analyticsIns.Stop()

			return nil
		}))
	}

	s.gs.AddShutdownCallback(shutdown.ShutdownFunc(func(string) error {
		cancel()

		return nil
	}))

	return nil
}
```

```internal/authzserver/store/load/load.go```

```go
// Store defines function to reload storage.
type Store interface {
	Reload() error
}

// Load is used to reload given storage.
type Load struct {
	ctx   context.Context
	lock  *sync.RWMutex
	store Store
}

// Start start a loop service.
func (l *Load) Start() {
	go startPubSubLoop()
	// 1s is the minimum amount of time between hot reloads. The
	// interval counts from the start of one reload to the next.
	go l.reloadLoop()
	go l.reloadQueueLoop()
	l.DoReload()
}


// DoReload reload secrets and policies.
func (l *Load) DoReload() {
	l.lock.Lock()
	defer l.lock.Unlock()

	if err := l.store.Reload(); err != nil {
		log.Errorf("faild to refresh target storage: %s", err.Error())
	}

	log.Debug("refresh target storage succ")
}
```

回到``internal/authzserver/store/store.go``

```go
// Reload reload secrets and policies.
func (s *Store) Reload() error {
	s.lock.Lock()
	defer s.lock.Unlock()

	// reload secrets
	secrets, err := s.cli.GetSecrets()
	if err != nil {
		return errors.Wrap(err, "list secrets failed")
	}

	s.secrets.Clear()
	for key, val := range secrets {
		s.secrets.Set(key, val, 1)
	}

	// reload policies
	policies, err := s.cli.GetPolicies()
	if err != nil {
		return errors.Wrap(err, "list policies failed")
	}

	s.policies.Clear()
	for key, val := range policies {
		s.policies.Set(key, val, 1)
	}

	return nil
}
```

```internal/authzserver/server.go``` 中initialize     方法会传入Store里的Client对象

```go
func (s *authzServer) initialize() error {
		......
  storeIns, err := store.GetStoreInsOr(store.GetGRPCClientOrDie(s.rpcServer, s.clientCA))
  .....
}
```

```internal/authzserver/store/client.go ```

获取StoreClient的实现，GRPCClient

```go
// GetGRPCClientOrDie return cache instance and panics on any error.
func GetGRPCClientOrDie(address string, clientCA string) StoreClient {
	if address != "" && clientCA != "" {
		once.Do(func() {
			var (
				err   error
				conn  *grpc.ClientConn
				creds credentials.TransportCredentials
			)

			creds, err = credentials.NewClientTLSFromFile(clientCA, "")
			if err != nil {
				log.Panicf("credentials.NewClientTLSFromFile err: %v", err)
			}

			conn, err = grpc.Dial(address, grpc.WithBlock(), grpc.WithTransportCredentials(creds))
			if err != nil {
				log.Panicf("Connect to grpc server failed, error: %s", err.Error())
			}

			client = &GRPCClient{pb.NewCacheClient(conn)}
			log.Infof("Connected to grpc server, address: %s", address)
		})
	}

	return client
}
```

```go
//https://github.com/marmotedu/iam/blob/v1.0.4/internal/authzserver/store/client.go
// GetPolicies returns all the authorization policies.
func (c *GRPCClient) GetPolicies() (map[string][]*ladon.DefaultPolicy, error) {
	pols := make(map[string][]*ladon.DefaultPolicy)

	log.Info("Loading policies")

	req := &pb.ListPoliciesRequest{
		Offset: pointer.ToInt64(0),
		Limit:  pointer.ToInt64(-1),
	}

	resp, err := c.cli.ListPolicies(context.Background(), req)
	if err != nil {
		return nil, errors.Wrap(err, "list policies failed")
	}

	log.Infof("Policies found (%d total)[username:name]:", len(resp.Items))

	for _, v := range resp.Items {
		log.Infof(" - %s:%s", v.Username, v.Name)

		var policy ladon.DefaultPolicy

		if err := json.Unmarshal([]byte(v.PolicyShadow), &policy); err != nil {
			log.Warnf("failed to load policy for %s, error: %s", v.Name, err.Error())

			continue
		}

		pols[v.Username] = append(pols[v.Username], &policy)
	}

	return pols, nil
}
```





