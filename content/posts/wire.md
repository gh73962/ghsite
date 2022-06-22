---
title: "译: Compile-time Dependency Injection With Go Cloud's Wire"
date: 2022-06-22
tags:
- dependency injection
- wire
by:
- Robert van Gent
summary: 如何使用wire, Go依赖注入工具.
---

## 概述

Go团队最近[宣布](https://golang.google.cn/go-cloud)了一个开源项目[Go Cloud](https://github.com/google/go-cloud),
该项目通过稳定的云平台API和工具帮助开放式云平台的开发.
这篇博文带你详细了解如何通过Wire来实现依赖注入

## Wire解决了什么问题?

[依赖注入](https://en.wikipedia.org/wiki/Dependency_injection)
是一个用来生成高内聚耦合代码的标准工具，
它显式地为组件提供它们工作所需的所有依赖关系。
在 Go 中，这通常采用将依赖项传递给构造函数的形式：

```go
	// NewUserStore returns a UserStore that uses cfg and db as dependencies.
	func NewUserStore(cfg *Config, db *mysql.DB) (*UserStore, error) {...}
```

这种方式在小规模的场景上效果很好，
但是大型应用可能会有更复杂的依赖关系图，
有着一大块初始化代码的结果就是依赖于顺序,这种情况可不太好。
通常很难干净地分解这种代码，
尤其是某些依赖项被多次使用。
用一个服务的实现替换另一个的可能会非常痛苦，
因为它涉及通过添加一组全新的依赖项（及其依赖项......）来修改现有依赖关系图，
并删除未使用的旧的。
在实践中，修改应用程序中初始化代码的大型依赖图既乏味又缓慢。

像 Wire 这样的依赖注入工具旨在简化初始化代码的管理。
你描述了你的服务及其依赖项，
不论是代码还是配置，Wire 都会通过结果图计算出顺序以及如何传递每个服务所需的内容。
通过更改函数签名或添加或删除初始化程序来修改应用程序的依赖项，然后让 Wire 完成生成整个依赖图初始化代码的繁琐工作。

## 为什么是Go Cloud的一部分？

Go Cloud 的目标是通过为有用的云服务提供可移植的 Go API 来更轻松地编写可移植的云应用程序。
举个例子，[blob.Bucket](https://pkg.go.dev/github.com/google/go-cloud/blob)
提供了一个存储API，这个API带有亚马逊S3和谷歌云存储（GCS）的实现；
使用 `blob.Bucket` 编写的应用程序可以在不更改其应用程序逻辑的情况下交换实现。
但是，初始化代码本质上是特定于提供程序的，每个提供者都有一组不同的依赖项。

举个例子，[构造一个 GCS `blob.Bucket`](https://pkg.go.dev/github.com/google/go-cloud/blob/gcsblob#OpenBucket)
需要一个`gcp.HTTPClient`，它需要`google.Credentials`，
当[构造一个S3](https://pkg.go.dev/github.com/google/go-cloud/blob/s3blob)
需要一个`aws.Config`，它需要AWS credentials。
因此，使用不同的`blob.Bucket`实现来更新应用程序涉及到我们上面描述的那种对依赖的乏味更新。
Wire 的驱动用例是使交换 Go Cloud 可移植 API 的实现变得容易，
但它也是依赖注入的通用工具。

## Hasn't this been done already?

目前已经有很多的依赖注入框架，
对于Go,[Uber's dig](https://github.com/uber-go/dig) 和 [Facebook's inject](https://github.com/facebookgo/inject) 
都是使用通过反射来实现运行时依赖注入。
Wire是从Java的 [Dagger 2] (https://google.github.io/dagger/)获得启发，
它使用代码生成来实现的而不是反射或服务定位器模式。

我们认为这种方法有几个优点：

  - 当依赖路径变的复杂的时候,运行时依赖注入会难以追踪和debug，
    使用代码生成意味着运行时执行的初始化代码是规律的
    ~~惯用~~的Go代码是很容易理解和debug的。
    任何东西都不会被中间框架的“魔法”误导。
    特别是忘记依赖项之类的问题会变成编译时错误，而不是运行时错误。
  - 不同于 [服务器定位模式](https://en.wikipedia.org/wiki/Service_locator_pattern)，
    无需编造任何名称或密钥来注册服务。
    Wire 使用 Go 类型来连接组件及其依赖项。
  - 更容易避免依赖膨胀，Wire生成的代码只需你导入你需要依赖即可，
    所以你的二进制文件不会有任何未使用的导入。
    运行时依赖注入在运行之前并不能识别未使用的依赖。
  - Wire 的依赖图是静态可知的，这为工具和可视化提供了机会。

## 它是如何工作的？

Wire有俩个基本的设计理念: 提供者(providers)和注入者(injectors)。

_提供者_ 是常规的 Go 函数, 它为它们的依赖"提供"值，
这些值被简单地描述为函数的入参。
下面这些简单示例有个三个提供者 :

```go
	// NewUserStore is the same function we saw above; it is a provider for UserStore,
	// with dependencies on *Config and *mysql.DB.
	func NewUserStore(cfg *Config, db *mysql.DB) (*UserStore, error) {...}

	// NewDefaultConfig is a provider for *Config, with no dependencies.
	func NewDefaultConfig() *Config {...}

	// NewDB is a provider for *mysql.DB based on some connection info.
	func NewDB(info *ConnectionInfo) (*mysql.DB, error) {...}
```

那些常用的提供者可以成组归入`ProviderSets`。
举个例子，在创建一个`*UserStore`时通常使用一个默认的`*Config`，
所以我们我可以把`NewUserStore`和`NewDefaultConfig`合成一组,归入到`ProviderSet`:

```go
	var UserStoreSet = wire.ProviderSet(NewUserStore, NewDefaultConfig)
```

_注入者_ 是被生成出来的函数，是在依赖顺序中用来调用提供者。
你要编写注入者的函数签名，包括所有你需要的入参，
构建最终结果需要使用提供者集合或提供者列表插入对`wire.Build`的调用：

```go
	func initUserStore() (*UserStore, error) {
		// We're going to get an error, because NewDB requires a *ConnectionInfo
		// and we didn't provide one.
		wire.Build(UserStoreSet, NewDB)
		return nil, nil  // These return values are ignored.
	}
```

现在我们运行go generate来执行Wire:

	$ go generate
	wire.go:2:10: inject initUserStore: no provider found for ConnectionInfo (required by provider of *mysql.DB)
	wire: generate failed

Oops! 我们没有包含`ConnectionInfo`或告诉Wire如何构建一个。
Wire 会帮助性的告诉我们所涉及的行号和类型。
我们可以为它添加一个提供者到`wire.Build`，
或将其添加为参数：

```go
	func initUserStore(info ConnectionInfo) (*UserStore, error) {
		wire.Build(UserStoreSet, NewDB)
		return nil, nil  // These return values are ignored.
	}
```

现在 `go generate` 将使用生成的代码创建一个新文件：

	// File: wire_gen.go
	// Code generated by Wire. DO NOT EDIT.
	//go:generate wire
	//+build !wireinject

```go
	func initUserStore(info ConnectionInfo) (*UserStore, error) {
		defaultConfig := NewDefaultConfig()
		db, err := NewDB(info)
		if err != nil {
			return nil, err
		}
		userStore, err := NewUserStore(defaultConfig, db)
		if err != nil {
			return nil, err
		}
		return userStore, nil
	}
```

任何非注入器声明都会被复制到生成的文件中。
运行时并不会依赖 Wire：
所有编写的代码只是普通的Go代码而已。

如你所见，这个输出非常接近开发者自己要编写的代码。
这是一个只有三个组件的简单示例，
所以手工编写初始化器不会太痛苦，
但是 Wire 为具有更复杂依赖图的组件和应用程序节省了大量的人力。

## 我如何参与并了解更多信息？

这个 [Wire README](https://github.com/google/wire/blob/master/README.md) 
更详细地介绍了如何使用 Wire 及其更高级的功能。
还有一个[教程](https://github.com/google/wire/tree/master/_tutorial)
介绍了如何在一个简单的应用程序中使用 Wire。

我们感谢您对 Wire 体验的任何意见！
[Wire](https://github.com/google/wire) 的开发是在 GitHub 上进行的，
因此您可以[提交问题](https://github.com/google/wire/issues/new/choose)
来告诉我们有什么更好的地方。
有关该项目的更新和讨论，
加入 [Go Cloud 邮件列表](https://groups.google.com/forum/#!forum/go-cloud)。

感谢您花时间了解 Go Cloud 的 Wire。
我们很高兴与您合作，让 Go 成为开发人员构建可移植云应用程序的首选语言。