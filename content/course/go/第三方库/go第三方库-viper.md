# go第三方库-viper

## 使用

```bash
go get github.com/spf13/viper
```



## 概述

viper是一个完整的go语言配置框架，能处理多种配置类型和文件格式，支持以下功能：

- 设置默认值
- 支持json、yaml、toml、hcl、env、property文件格式
- 支持读取环境变量
- 支持远程配置（etcd、consul）等，并监听数据变化
- 支持命令行配置
- 支持设置模糊值

## 使用说明

### 设置值存入viper

#### 设置默认值

```go
viper.SetDefault("key", "value")
```

#### 读取配置文件

```go
// 设置配置文件名（不带后缀）
viper.SetConfigName("config") // name of config file (without extension)
// 设置文件后缀
viper.SetConfigType("yaml") // REQUIRED if the config file does not have the extension in the name
// 查找配置文件的路径
viper.AddConfigPath("/etc/appname/")   // path to look for the config file in
// 可以配置多个路径
viper.AddConfigPath("$HOME/.appname")  // call multiple times to add many search paths
viper.AddConfigPath(".")               // optionally look for config in the working directory
// 读取配置文件
err := viper.ReadInConfig() // Find and read the config file
if err != nil { // Handle errors reading the config file
	panic(fmt.Errorf("Fatal error config file: %s \n", err))
}
```

