---
title: Go第三库之命令行框架：go-cli
date: 2020-08-17
author: kinnylee
toc: true
categories:
  - Go
tags:
  - Go第三方库
---

## Go-cli

### 简介

- A simple, fast, and fun package for building command line apps in Go
- 一个简单的、快速的、并且有趣的用于构建go应用程序的包
- [github地址](https://github.com/urfave/cli)
- 目标是帮助开发人员快速构建易于表达的命令行程序
- 目前存在两个版本v1、v2（新版本）

### 快速上手

[v2官方使用说明](https://github.com/urfave/cli/blob/master/docs/v2/manual.md)

- 1. 添加模块依赖

```bash
GO111MODULE=on go get github.com/urfave/cli/v2
```

- 2. 使用

```go
package main

import (
  "os"
  "github.com/urfave/cli/v2"
)

func main() {
  (&cli.App{}).Run(os.Args)
}
```

- 3. 编译：生成可执行文件

```go
go build
```

- 4. 使用：会打印出帮助文档

```bash
./cse
NAME:
   cse - A new cli application

USAGE:
   cse [global options] command [command options] [arguments...]

COMMANDS:
   help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --help, -h  show help (default: false)
```

### 使用说明

cli.App提供了很多参数，比较重要的几个有：

- Name: 应用程序名称
- Flag：参数信息
- Command：要执行的命令

#### Arguments

- App可以指定Action函数，action的回调函数参数为cli.Context
- 调用cli.Context的Args方法，可以查看参数

```go
app := &cli.App{
    Action: func(c *cli.Context) error {
      fmt.Printf("Hello %q", c.Args().Get(0))
      return nil
    },
  }
```

#### Flag

- App的Flags参数，可以设置相关参数

##### 常用参数

- Name: flag名称
- Aliases：别名，可以指定多个
- Usage：使用说明
- Value：默认值
- Required: 是否是必填值

```Go
app := &cli.App{
  ...
  Flags: []cli.Flag{
    &cli.IntFlag{
      Name:        "c",
      Aliases: []string{"concurrency"},
      Usage:       "concurrency thread",
      Value:       5,
      Destination: &ConcurrencyThread,
    },
  },
}
```

##### 接收Flag

- Destination参数，用于接收传入的flag值

```Go
app := &cli.App{
  ...
  Flags: []cli.Flag{
    &cli.IntFlag{
      Name:        "c",
      Aliases: []string{"concurrency"},
      Usage:       "concurrency thread",
      Value:       5,
      Destination: &ConcurrencyThread,
    },
  },
}
```

##### 排序

- Flag的默认排序是代码定义的顺序
- 可以设置为按照名称排序

```Go
func main() {
  ...
  sort.Sort(cli.FlagsByName(app.Flags))

  err := app.Run(os.Args)
  if err != nil {
    log.Fatal(err)
  }
}
```

##### 读取环境变量值

- 可以为Flag的默认值为环境变量的值
- 环境变量可以设置多个，读取到的第一个有值的环境变量作为默认值

```Go
app := &cli.App{
  Flags: []cli.Flag{
    &cli.StringFlag{
      Name: "repo_url",
      Usage: "resource repository url",
      Value: "https://obs.cn-north-4.myhuaweicloud.com",
      EnvVars: []string{"CSE_REPO_URL"},
    },
  },
}
```

##### 读取文件值

- 可以设置Flag的默认值为读取的文件中的数据

```Go
app := &cli.App{
  Flags: []cli.Flag{
    &cli.StringFlag{
      Name: "repo_url",
      Usage: "resource repository url",
      Value: "https://obs.cn-north-4.myhuaweicloud.com",
      FilePath: "/etc/cse/repo_url",
    },
  },
}
```

#### 读取配置文件

有一个单独的包altsrc提供配置文件和flag值的关联，目前支持的文件格式包括：

- Yaml
- Json
- Toml

```Go
// 读取load.yaml文件中的test值，作为flag的值
func main() {
  flags := []cli.Flag{
    altsrc.NewIntFlag(&cli.IntFlag{Name: "test"}),
    &cli.StringFlag{Name: "load"},
  }

  app := &cli.App{
    Action: func(c *cli.Context) error {
      fmt.Println("yaml ist rad")
      return nil
    },
    Before: altsrc.InitInputSourceWithContext(flags, altsrc.NewYamlSourceFromFlagFunc("load")),
    Flags: flags,
  }

  app.Run(os.Args)
}
```

##### 参数读取的优先级（由高到低）

- 0：用户通过命令行传入的值
- 1：环境变量的值（如果有）
- 2：配置文件的值（如果有）
- 3：Default参数设置的值

#### Command

- Command参数指定要执行操作的相关信息
- Command的Action指定真正的操作业务逻辑

```Go
app := &cli.App{
  ...
  Commands: []*cli.Command{
    {
      Name:    "download",
      Aliases: nil,
      Usage:   "download resource by description file(project.json)",
      Action: func(context *cli.Context) error {
        return nil
      },
    },
  },
}
```

##### Command排序

- Command的默认排序是代码定义的顺序
- 可以设置为按照名称排序

```Go
func main() {
  ...
  sort.Sort(cli.CommandsByName(app.Commands))

  err := app.Run(os.Args)
  if err != nil {
    log.Fatal(err)
  }
}
```

##### 子命令 SubCommand

效果：

```bash
./cse download
NAME:
   cse download - download resource by description file(project.json)

USAGE:
   cse download command [command options] [arguments...]

COMMANDS:
   Installer  download installer
   Yum        download yum
   help, h    Shows a list of commands or help for one command

./cse download Installer
download installer

```

```Go
app := &cli.App{
  ...
  Commands: []*cli.Command{
    {
      Name:    "download",
      Aliases: nil,
      Usage:   "download resource by description file(project.json)",
      Subcommands: []*cli.Command{
        {
          Name: "Installer",
          Usage: "download installer",
          Action: func(context *cli.Context) error {
            fmt.Println("download installer")
            return nil
          },
        },
        {
          Name: "Yum",
          Usage: "download yum",
          Action: func(context *cli.Context) error {
            fmt.Println("download yum")
            return nil
          },
        },
      },
    },
  },
}
```

##### 命令分类

- 当有很多command时，可以将相关的command分为一个组，在帮助文档中按组展示
- 只需要添加Category字段即可

```Go
app := &cli.App{
  Commands: []*cli.Command{
    {
      Name: "noop",
    },
    {
      Name:     "add",
      Category: "template",
    },
    {
      Name:     "remove",
      Category: "template",
    },
  },
}
```

#### 所有的参数

```Go
type App struct {
	// The name of the program. Defaults to path.Base(os.Args[0])
	Name string
	// Full name of command for help, defaults to Name
	HelpName string
	// Description of the program.
	Usage string
	// Text to override the USAGE section of help
	UsageText string
	// Description of the program argument format.
	ArgsUsage string
	// Version of the program
	Version string
	// Description of the program
	Description string
	// List of commands to execute
	Commands []*Command
	// List of flags to parse
	Flags []Flag
	// Boolean to enable bash completion commands
	EnableBashCompletion bool
	// Boolean to hide built-in help command and help flag
	HideHelp bool
	// Boolean to hide built-in help command but keep help flag.
	// Ignored if HideHelp is true.
	HideHelpCommand bool
	// Boolean to hide built-in version flag and the VERSION section of help
	HideVersion bool
	// categories contains the categorized commands and is populated on app startup
	categories CommandCategories
	// An action to execute when the shell completion flag is set
	BashComplete BashCompleteFunc
	// An action to execute before any subcommands are run, but after the context is ready
	// If a non-nil error is returned, no subcommands are run
	Before BeforeFunc
	// An action to execute after any subcommands are run, but after the subcommand has finished
	// It is run even if Action() panics
	After AfterFunc
	// The action to execute when no subcommands are specified
	Action ActionFunc
	// Execute this function if the proper command cannot be found
	CommandNotFound CommandNotFoundFunc
	// Execute this function if an usage error occurs
	OnUsageError OnUsageErrorFunc
	// Compilation date
	Compiled time.Time
	// List of all authors who contributed
	Authors []*Author
	// Copyright of the binary if any
	Copyright string
	// Writer writer to write output to
	Writer io.Writer
	// ErrWriter writes error output
	ErrWriter io.Writer
	// Execute this function to handle ExitErrors. If not provided, HandleExitCoder is provided to
	// function as a default, so this is optional.
	ExitErrHandler ExitErrHandlerFunc
	// Other custom info
	Metadata map[string]interface{}
	// Carries a function which returns app specific info.
	ExtraInfo func() map[string]string
	// CustomAppHelpTemplate the text template for app help topic.
	// cli.go uses text/template to render templates. You can
	// render custom help text by setting this variable.
	CustomAppHelpTemplate string
	// Boolean to enable short-option handling so user can combine several
	// single-character bool arguments into one
	// i.e. foobar -o -v -> foobar -ov
	UseShortOptionHandling bool

	didSetup bool
}
```

#### 多个参数合并

- 如果有多个参数，传统的方法是分别指定每个参数，比如：-s -o -m
- 通过相关设置，可以将多个参数合并，比如：-som

```Go
app := &cli.App{}
app.UseShortOptionHandling = true
```

#### bash自动补全

- 可以通过EnableBashCompletion设置参数自动补全
- 默认只支持subcommand，可以实现补全方法`BashComplete`支持所有命令自动补全

```Go
app := cli.NewApp()
app.EnableBashCompletion = true
```

#### 退出状态

- 调用App.Run不会自动调用os.Exit，这意味着默认的退出码将不会生效，自动变成0
- 如果出错，需要显示的在return之前调用状态码

```Go
app := &cli.App{
  Flags: []cli.Flag{
    &cli.BoolFlag{
      Name:  "ginger-crouton",
      Usage: "is it in the soup?",
    },
  },
  Action: func(ctx *cli.Context) error {
    if !ctx.Bool("ginger-crouton") {
      return cli.Exit("Ginger croutons are not in the soup", 86)
    }
    return nil
  },
}
```