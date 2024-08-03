举个例子，低质量的 Makefile 文件是什么样的

```makefile
build: clean vet
  @mkdir -p ./Role
  @export GOOS=linux && go build -v .

vet:
  go vet ./...

fmt:
  go fmt ./...

clean:
  rm -rf dashboard
```

上面这个 Makefile 存在不少问题。例如：功能简单，只能完成最基本的编译、格式化等操作，像构建镜像、自动生成代码等一些高阶的功能都没有；扩展性差，没法编译出可在 Mac 下运行的二进制文件；没有 Help 功能，使用难度高；单 Makefile 文件，结构单一，不适合添加一些复杂的管理功能。



那么如何编写一个高质量的 Makefile 呢

下面是 IAM 项目的 Makefile 所集成的功能

```makefile
$ make help

Usage: make <TARGETS> <OPTIONS> ...

Targets:
  # 代码生成类命令
  gen                Generate all necessary files, such as error code files.

  # 格式化类命令
  format             Gofmt (reformat) package sources (exclude vendor dir if existed).

  # 静态代码检查
  lint               Check syntax and styling of go sources.

  # 测试类命令
  test               Run unit test.
  cover              Run unit test and get test coverage.

  # 构建类命令
  build              Build source code for host platform.
  build.multiarch    Build source code for multiple platforms. See option PLATFORMS.

  # Docker镜像打包类命令
  image              Build docker images for host arch.
  image.multiarch    Build docker images for multiple platforms. See option PLATFORMS.
  push               Build docker images for host arch and push images to registry.
  push.multiarch     Build docker images for multiple platforms and push images to registry.

  # 部署类命令
  deploy             Deploy updated components to development env.

  # 清理类命令
  clean              Remove all files that are created by building.

  # 其他命令，不同项目会有区别
  release            Release iam
  verify-copyright   Verify the boilerplate headers for all files.
  ca                 Generate CA files for all iam components.
  install            Install iam system with all its components.
  swagger            Generate swagger document.
  tools              install dependent tools.

  # 帮助命令
  help               Show this help info.

# 选项
Options:
  DEBUG        Whether to generate debug symbols. Default is 0.
  BINS         The binaries to build. Default is all of cmd.
               This option is available when using: make build/build.multiarch
               Example: make build BINS="iam-apiserver iam-authz-server"
  ...
```

通常而言，Go 项目的 Makefile 应该实现以下功能：格式化代码、静态代码检查、单元测试、代码构建、文件清理、帮助等等。如果通过 docker 部署，还需要有 docker 镜像打包功能。因为 Go 是跨平台的语言，所以构建和 docker 打包命令，还要能够支持不同的 CPU 架构和平台。为了能够更好地控制 Makefile 命令的行为，还需要支持 Options。



help 命令最好通过解析 Makefile 文件来输出集成的功能，例如：

```makefile
## help: Show this help info.
.PHONY: help
help: Makefile
  @echo -e "\nUsage: make <TARGETS> <OPTIONS> ...\n\nTargets:"
  @sed -n 's/^##//p' $< | column -t -s ':' | sed -e 's/^/ /'
  @echo "$$USAGE_OPTIONS"
```

上面的 help 命令，通过解析 Makefile 文件中的##注释，获取支持的命令。通过这种方式，我们以后新加命令时，就不用再对 help 命令进行修改了。

#### 设计合理的 Makefile 结构 ####

根目录下的 Makefile 聚合所有的 Makefile 命令，具体实现则按功能分类，放在另外的 Makefile 中。

<img src="https://static001.geekbang.org/resource/image/5c/f7/5c524e0297b6d6e4e151643d2e1bbbf7.png?wh=2575x1017" style="zoom:33%;" />

以下是 IAM 项目根目录下，Makefile 的内容摘录

```makefile
include scripts/make-rules/golang.mk
include scripts/make-rules/image.mk
include scripts/make-rules/gen.mk
include scripts/make-rules/...

## build: Build source code for host platform.
.PHONY: build
build:
  @$(MAKE) go.build

## build.multiarch: Build source code for multiple platforms. See option PLATFORMS.
.PHONY: build.multiarch
build.multiarch:
  @$(MAKE) go.build.multiarch

## image: Build docker images for host arch.
.PHONY: image
image:
  @$(MAKE) image.build

## push: Build docker images for host arch and push images to registry.
.PHONY: push
push:
  @$(MAKE) image.push

## ca: Generate CA files for all iam components.
.PHONY: ca
ca:
  @$(MAKE) gen.ca
```

**善用通配符和自动变量**

```makefile
tools.verify.%:
  @if ! which $* &>/dev/null; then $(MAKE) tools.install.$*; fi
```

这里也能看出tools.verify.%这种命名方式的好处：tools 说明依赖的定义位于scripts/make-rules/tools.mk Makefile 中；verify说明tools.verify.%伪目标属于 verify 分类，主要用来验证工具是否安装