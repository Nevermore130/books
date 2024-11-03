### 如何构建应用框架 ###

* 命令行参数解析：主要用来解析命令行参数，这些命令行参数可以影响命令的运行效果。
* 配置文件解析：一个大型应用，通常具有很多参数，为了便于管理和配置这些参数，通常会将这些参数放在一个配置文件中，供程序读取并解析。

### 命令行参数解析工具：Pflag 使用介绍 ###

Pflag 可以对命令行参数进行处理，一个命令行参数在 Pflag 包中会解析为一个 Flag 类型的变量。Flag 是一个结构体

```go
type Flag struct {
    Name                string // flag长选项的名称
    Shorthand           string // flag短选项的名称，一个缩写的字符
    Usage               string // flag的使用文本
    Value               Value  // flag的值
    DefValue            string // flag的默认值
    Changed             bool // 记录flag的值是否有被设置过
    NoOptDefVal         string // 当flag出现在命令行，但是没有指定选项值时的默认值
    Deprecated          string // 记录该flag是否被放弃
    Hidden              bool // 如果值为true，则从help/usage输出信息中隐藏该flag
    ShorthandDeprecated string // 如果flag的短选项被废弃，当使用flag的短选项时打印该信息
    Annotations         map[string][]string // 给flag设置注解
}
```

Flag 的值是一个 Value 类型的接口，Value 定义如下：

```go
type Value interface {
    String() string // 将flag类型的值转换为string类型的值，并返回string的内容
    Set(string) error // 将string类型的值转换为flag类型的值，转换失败报错
    Type() string // 返回flag的类型，例如：string、int、ip等
}
```

FlagSet 是一些预先定义好的 Flag 的集合

#### Pflag 使用方法 ####

```go
//支持长选项、默认值和使用文本，并将标志的值存储在指针中。
var name = pflag.String("name", "colin", "Input Your Name")

//支持长选项、默认值和使用文本，并将标志的值绑定到变量。
var name string
pflag.StringVar(&name, "name", "colin", "Input Your Name")
```

使用Get<Type>获取参数的值。

```go
i, err := flagset.GetInt("flagname")
```

获取非选项参数:

```go
package main

import (
    "fmt"

    "github.com/spf13/pflag"
)

var (
    flagvar = pflag.Int("flagname", 1234, "help message for flagname")
)

func main() {
    pflag.Parse()

    fmt.Printf("argument number is: %v\n", pflag.NArg())
    fmt.Printf("argument list is: %v\n", pflag.Args())
    fmt.Printf("the first argument is: %v\n", pflag.Arg(0))
}

//$ go run example1.go arg1 arg2
//argument number is: 2
//argument list is: [arg1 arg2]
//the first argument is: arg1
```

* 可以调用pflag.Parse()来解析定义的标志。解析后，可通过pflag.Args()返回所有的非选项参数
* 通过pflag.Arg(i)返回第 i 个非选项参数

### 配置解析神器：Viper 使用介绍 ###

Viper 可以从不同的位置读取配置，不同位置的配置具有不同的优先级，高优先级的配置会覆盖低优先级相同的配置

```go

//设置默认值。
viper.SetDefault("ContentDir", "content")
viper.SetDefault("LayoutDir", "layouts")
viper.SetDefault("Taxonomies", map[string]string{"tag": "tags", "category": "categories"})


```

读取配置文件：

```go
package main

import (
  "fmt"

  "github.com/spf13/pflag"
  "github.com/spf13/viper"
)

// xxx -c config.yaml
var (
  cfg  = pflag.StringP("config", "c", "", "Configuration file.")
  help = pflag.BoolP("help", "h", false, "Show this help message.")
)

func main() {
  pflag.Parse()
  if *help {
    pflag.Usage()
    return
  }

  // 从配置文件中读取配置
  if *cfg != "" {
    viper.SetConfigFile(*cfg)   // 指定配置文件名
    viper.SetConfigType("yaml") // 如果配置文件名中没有文件扩展名，则需要指定配置文件的格式，告诉viper以何种格式解析文件
  } else {
    viper.AddConfigPath(".")          // 把当前目录加入到配置文件的搜索路径中
    viper.AddConfigPath("$HOME/.iam") // 配置文件搜索路径，可以设置多个配置文件搜索路径
    viper.SetConfigName("config")     // 配置文件名称（没有文件扩展名）
  }

  if err := viper.ReadInConfig(); err != nil { // 读取配置文件。如果指定了配置文件名，则使用指定的配置文件，否则在注册的搜索路径中搜索
    panic(fmt.Errorf("Fatal error config file: %s \n", err))
  }

  fmt.Printf("Used configuration file is: %s\n", viper.ConfigFileUsed())
}
```

监听和重新读取配置文件。

Viper 支持在运行时让应用程序实时读取配置文件，也就是热加载配置。可以通过 WatchConfig 函数热加载配置。在调用 WatchConfig 函数之前，需要确保已经添加了配置文件的搜索路径。另外，还可以为 Viper 提供一个回调函数，以便在每次发生更改时运行。

```go
viper.WatchConfig()
viper.OnConfigChange(func(e fsnotify.Event) {
   // 配置文件发生变更之后会调用的回调函数
  fmt.Println("Config file changed:", e.Name)
})
```

使用环境变量:

* Viper 读取环境变量是区分大小写的
* 使用 SetEnvPrefix，可以告诉 Viper 在读取环境变量时使用前缀
* 比如，我们设置了 viper.SetEnvPrefix(“VIPER”)，当使用 viper.Get(“apiversion”) 时，实际读取的环境变量是VIPER_APIVERSION

**使用标志**

Viper 支持 Pflag 包，能够绑定 key 到 Flag。与 BindEnv 类似，在调用绑定方法时，不会设置该值，但在访问它时会设置

```go
viper.BindPFlag("token", pflag.Lookup("token")) // 绑定单个标志
```

#### 读取配置 ####

访问嵌套的键
```go
{
    "host": {
        "address": "localhost",
        "port": 5799
    },
    "datastore": {
        "metric": {
            "host": "127.0.0.1",
            "port": 3099
        },
        "warehouse": {
            "host": "198.0.0.1",
            "port": 2112
        }
    }
}

//Viper 可以通过传入.分隔的路径来访问嵌套字段：
viper.GetString("datastore.metric.host") // (返回 "127.0.0.1")

//Viper 可以支持将所有或特定的值解析到结构体、map 等
type config struct {
  Port int
  Name string
  PathMap string `mapstructure:"path_map"`
}

var C config

err := viper.Unmarshal(&C)
if err != nil {
  t.Fatalf("unable to decode into struct, %v", err)
}
```



### 现代化的命令行框架：Cobra  ###

例如：

```shell
git clone URL --bare # clone 是一个命令，URL是一个非选项参数，bare是一个选项参数
```

使用 Cobra 库创建命令

通常情况下，我们会将 rootCmd 放在文件 cmd/root.go 中

```go
var rootCmd = &cobra.Command{
  Use:   "hugo",
  Short: "Hugo is a very fast static site generator",
  Long: `A Fast and Flexible Static Site Generator built with
                love by spf13 and friends in Go.
                Complete documentation is available at http://hugo.spf13.com`,
  Run: func(cmd *cobra.Command, args []string) {
    // Do Stuff Here
  },
}

func Execute() {
  if err := rootCmd.Execute(); err != nil {
    fmt.Println(err)
    os.Exit(1)
  }
}
```

还可以在 init() 函数中定义标志和处理配置，例如 cmd/root.go。

```go
import (
  "fmt"
  "os"

  homedir "github.com/mitchellh/go-homedir"
  "github.com/spf13/cobra"
  "github.com/spf13/viper"
)

var (
    cfgFile     string
    projectBase string
    userLicense string
)

func init() {
  cobra.OnInitialize(initConfig)
  rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "", "config file (default is $HOME/.cobra.yaml)")
  rootCmd.PersistentFlags().StringVarP(&projectBase, "projectbase", "b", "", "base project directory eg. github.com/spf13/")
  rootCmd.PersistentFlags().StringP("author", "a", "YOUR NAME", "Author name for copyright attribution")
  rootCmd.PersistentFlags().StringVarP(&userLicense, "license", "l", "", "Name of license for the project (can provide `licensetext` in config)")
  rootCmd.PersistentFlags().Bool("viper", true, "Use Viper for configuration")
  viper.BindPFlag("author", rootCmd.PersistentFlags().Lookup("author"))
  viper.BindPFlag("projectbase", rootCmd.PersistentFlags().Lookup("projectbase"))
  viper.BindPFlag("useViper", rootCmd.PersistentFlags().Lookup("viper"))
  viper.SetDefault("author", "NAME HERE <EMAIL ADDRESS>")
  viper.SetDefault("license", "apache")
}

func initConfig() {
  // Don't forget to read config either from cfgFile or from home directory!
  if cfgFile != "" {
    // Use config file from the flag.
    viper.SetConfigFile(cfgFile)
  } else {
    // Find home directory.
    home, err := homedir.Dir()
    if err != nil {
      fmt.Println(err)
      os.Exit(1)
    }

    // Search config in home directory with name ".cobra" (without extension).
    viper.AddConfigPath(home)
    viper.SetConfigName(".cobra")
  }

  if err := viper.ReadInConfig(); err != nil {
    fmt.Println("Can't read config:", err)
    os.Exit(1)
  }
}
```

main.go 中调用 rootCmd.Execute() 来执行命令：

```go
package main

import (
  "{pathToYourApp}/cmd"
)

func main() {
  cmd.Execute()
}
```

编译并运行。

将 main.go 中{pathToYourApp}替换为对应的路径

```go
$ go mod init github.com/marmotedu/gopractise-demo/cobra/newApp2
$ go build -v .
$ ./newApp2 -h
A Fast and Flexible Static Site Generator built with
love by spf13 and friends in Go.
Complete documentation is available at http://hugo.spf13.com
 
Usage:
hugo [flags]
hugo [command]
 
Available Commands:
help Help about any command
version Print the version number of Hugo
 
Flags:
-a, --author string Author name for copyright attribution (default "YOUR NAME")
--config string config file (default is $HOME/.cobra.yaml)
-h, --help help for hugo
-l, --license licensetext Name of license for the project (can provide licensetext in config)
-b, --projectbase string base project directory eg. github.com/spf13/
--viper Use Viper for configuration (default true)
 
Use "hugo [command] --help" for more information about a command.
```

将标志绑定到 Viper:

我们可以将标志绑定到 Viper，这样就可以使用 viper.Get() 获取标志的值

```go
var author string

func init() {
  rootCmd.PersistentFlags().StringVar(&author, "author", "YOUR NAME", "Author name for copyright attribution")
  viper.BindPFlag("author", rootCmd.PersistentFlags().Lookup("author"))
}
```

##### 非选项参数验证 #####

使用预定义验证函数

```go
var cmd = &cobra.Command{
  Short: "hello",
  Args: cobra.MinimumNArgs(1), // 使用内置的验证函数
  Run: func(cmd *cobra.Command, args []string) {
    fmt.Println("Hello, World!")
  },
}
```

自定义验证函数

```go
var cmd = &cobra.Command{
  Short: "hello",
  // Args: cobra.MinimumNArgs(10), // 使用内置的验证函数
  Args: func(cmd *cobra.Command, args []string) error { // 自定义验证函数
    if len(args) < 1 {
      return errors.New("requires at least one arg")
    }
    if myapp.IsValidColor(args[0]) {
      return nil
    }
    return fmt.Errorf("invalid color specified: %s", args[0])
  },
  Run: func(cmd *cobra.Command, args []string) {
    fmt.Println("Hello, World!")
  },
}
```

















