---
title: go第三方库 - Cobra
date: 2020-09-01
toc: true
categories:
  - k8s
  - go第三方库
tags:
  - k8s
thumbnail: 'images/cobra.png'
---

# k8s的命令行工具-cobra框架

## 简介

[Cobra](https://github.com/spf13/cobra) 是一个创建强大的现代化Cli命令行应用程序的go语言库。很多知名的开源软件都使用Cobra实现Cli。比如：

- k8s：kubectl等多个组件都用到
- Istio: istioctl命令
- Docker：docker命令
- Etcd：etcd命令
- Helm
- RClone

> spf13这个github项目的大佬开发了好几个知名的项目。比如：
> hugo：世界上最快的构建静态网站的项目（该文档的博客系统就是用hugo搭建的）
> cobra：今天的主角，命令行框架
> viper：配置信息处理框架

### 特性

- 支持子命令行模式
- 命令智能建议
- 轻松生成应用程序和命令
- 自动生成命令和参数的帮助信息
- 自动生成详细的命令行帮助
- 自动识别 -h --help
- bash环境下自动补全
- 自动生成帮助手册

## 基本概念

Cobra基于三个基本结构构建应用程序：

- Commands：表示执行动作。是应用程序交互的中心，支持子命令 SubCommand
- Args：执行参数
- Flags：动作的标识符，可以修改command的行为

展现形式为：
APPNAME VERB NOUN --ADJECTIVE. 或者 APPNAME COMMAND ARG --FLAG

## 安装

1. 通过`go get`安装最新版本库

```bash
go get github.io/spf13/cobra
```

2. 引入包名

```Go
import "github.com/spf13/cobra"
```

## 快速上手

基本操作包括三步：

- 创建command主命令，并定义Run执行函数（只定义，还没有执行），AddCommand可以添加命令或者子命令
- 为命令添加命令行参数
- 执行Execute命令（会在内部回调Run函数）

```Go
func main() {
  var Version bool

  var rootCmd = &cobra.Command{
    Use:                        "root [sub]",
    Short:                      "root command",
    Run: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside rootCmd Run with args: %v\n", args)
      if Version {
        fmt.Println("Version: 1.0")
      }
    },
  }

  flags := rootCmd.Flags()
  // 设置flags
  // 参数含义依次是：接收参数的变量，命令行参数名称，名称简写，默认值，参数提示信息
  flags.BoolVarP(&Version, "version", "v", false, "print version information")

  // 开始执行
  _ = rootCmd.Execute()
}
```

### 添加额外的command

```go
rootCmd.AddCommand(&cobra.Command{
    Use: "hello",
    Short: "hello",
    Run: func(cmd *cobra.Command, args []string) {
      fmt.Printf("hello")
    },
  })
```

### Flags

Flags分为全局flag和局部flag：

- 全局flag：对所有的command生效。rootCmd.PersistentFlags()
- 局部flag：只对特定的command生效。localCmd.Flags()

## 模板脚手架

cobra提供命令行工具，支持快速搭建cli应用程序，代替手工编写繁琐代码

```bash
# 引入包依赖
go get -u github.com/spf13/cobra/cobra
# 使用模板脚手架
cobra init --pkg-name cobraDemo
cd cobraDemo
# 查看目录结构
tree .
.
├── LICENSE
├── cmd
│   └── root.go
└── main.go
```

## 生命周期

Command中的Run字段，是执行的核心方法。Cobra支持应用生命周期的配置，运行在执行Run方法的前后定义业务逻辑。具体的执行顺序为：

- PersistentPreRun
- PreRun
- Run
- PostRun
- PersistentPostRun

```Go
var rootCmd = &cobra.Command{
    Use:   "root [sub]",
    Short: "My root command",
    PersistentPreRun: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside rootCmd PersistentPreRun with args: %v\n", args)
    },
    PreRun: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside rootCmd PreRun with args: %v\n", args)
    },
    Run: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside rootCmd Run with args: %v\n", args)
    },
    PostRun: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside rootCmd PostRun with args: %v\n", args)
    },
    PersistentPostRun: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside rootCmd PersistentPostRun with args: %v\n", args)
    },
  }
```

## 拼写错误智能提示

默认是开启拼写错误智能提示的，如果想关闭，可以如下设置

> 底层使用了字符串相似度算法（编辑距离算法 Levenshtein Distance），由俄罗斯科学家Vladimir Levenshtein在1965年提出

```Go
command.DisableSuggestions = true
```
