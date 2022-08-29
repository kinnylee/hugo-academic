---
title: Golang http请求源码分析
date: 2020-09-16
toc: true
categories:
  - k8s
  - go第三方库
tags:
  - k8s
thumbnail: 'images/go.png'
---

# Golang http请求源码分析

go提供的标准库net/http，实现一个简单的http server非常容易，只需要短短几行代码。本篇文章将会对go标准库net/http实现http服务的原理进行较为深入的探究

## 快速搭建http server服务

搭建http server的大概步骤包括：

- 编写handler处理函数
- 注册路由
- 创建服务并开启监听

```Go
package main

import (
  "io"
  "log"
  "net/http"
)

// 请求处理函数
func indexHandler(w http.ResponseWriter, r *http.Request) {
  _, _ = io.WriteString(w, "hello, world!\n")
}

func main() {
  // 注册路由
  http.HandleFunc("/", indexHandler)
  // 创建服务并开启监听
  err := http.ListenAndServe(":8001", nil)
  if err != nil {
    log.Fatal("ListenAndServe: ", err)
  }
}
```

## http服务处理流程

- 请求会先进入路由
- 路由为请求找到合适的handler
- handler对request进行处理，并构建response

![http服务处理流程](http://assets.processon.com/chart_image/5f63606e7d9c0833ecf363cc.png)

## Golang的http包处理流程

- 路由处理的核心对象是ServeMux
- ServeMux内部维护一个map属性，保存了路由路径和路由处理函数的映射关系
- 注册路由时，往map中写入数据
- 匹配路由时，从map中找到合适的handler处理

![go-http处理流程](http://assets.processon.com/chart_image/5f637df37d9c0833ecf38b15.png)

## 关键源码逻辑

下图展示的源码中的关键逻辑：

[高清地址](https://www.processon.com/view/link/5f6384b35653bb28eb44fbcc)

![golang http源码关键逻辑](http://assets.processon.com/chart_image/5f6384b37d9c0833ecf3918c.png)

## 路由注册接口

共有两个函数可以用于路由注册，底层都调用的是DefaultServeMux

源码位置：src/net/http/server.go

```Go
type Handler interface {
  ServeHTTP(ResponseWriter, *Request)
}

func Handle(pattern string, handler Handler) {
  DefaultServeMux.Handle(pattern, handler)
}

func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
  DefaultServeMux.HandleFunc(pattern, handler)
}
```

## 路由实现

go中的路由基于ServeMux结构实现

```Go
type ServeMux struct {
  mu    sync.RWMutex
  // 存储路由和handler的对应关系
  m     map[string]muxEntry
  // 将muxEntry排序存放，排序按照路由表达式由长到短排序
  es    []muxEntry
  // 路由表达式是否包含主机名
  hosts bool
}

type muxEntry struct {
  // 路由处理函数
  h       Handler
  // 路由表达式
  pattern string
}

```

## 路由注册逻辑

- go提供了默认的路由实例DefaultServeMux，如果用户没有自定义路由，就用这个默认的路由
- 添加路由函数的核心逻辑：将表达式作为key，路由处理函数和表达式组成的muxEntry作为value保存到map中

```Go
// 服务启动后的默认路由实例
var DefaultServeMux = &defaultServeMux

// 前面demo中调用handle的内部逻辑
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
  DefaultServeMux.HandleFunc(pattern, handler)
}

// HandleFunc
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
  ...
  mux.Handle(pattern, HandlerFunc(handler))
}

// Handle
func (mux *ServeMux) Handle(pattern string, handler Handler) {
  ...
  // 创建ServeMux的m实例
  if mux.m == nil {
    mux.m = make(map[string]muxEntry)
  }
  // 根据路由表达式和路由处理函数，构造muxEntry对象
  e := muxEntry{h: handler, pattern: pattern}
  // muxEntry保存到map中
  mux.m[pattern] = e

  // 如果表达式以 '/' 结尾，加入到排序列表中
  if pattern[len(pattern)-1] == '/' {
    mux.es = appendSorted(mux.es, e)
  }

  if pattern[0] != '/' {
    mux.hosts = true
  }
}
```

## 开启服务

核心逻辑包括：监听端口、等待连接、创建连接、处理请求

```Go
// 开启服务的入口
func ListenAndServe(addr string, handler Handler) error {
  // 创建一个Server，传入handler
  // 我们的例子中handler为空
  server := &Server{Addr: addr, Handler: handler}
  // 调用ListenAndServe真正监听
  return server.ListenAndServe()
}

// ListenAndServe
func (srv *Server) ListenAndServe() error {
  ...
  ln, err := net.Listen("tcp", addr)
  return srv.Serve(ln)
}

func (srv *Server) Serve(l net.Listener) error {
  ...
  // for循环
  for {
    // 创建上下文对象
    ctx := context.WithValue(baseCtx, ServerContextKey, srv)
    // 等待新的连接建立
    rw, err := l.Accept()
    ...
    // 连接建立时，创建连接对象
    c := srv.newConn(rw)
    c.setState(c.rwc, StateNew) // before Serve can return
    // 创建协程处理请求
    go c.serve(connCtx)
  }
}
```

## 处理请求

处理请求的逻辑主要是：根据路由请求去和ServeMux的m做匹配，找到合适的handler

```Go
func (c *conn) serve(ctx context.Context) {
  ...
  for {
    // 读取下一个请求进行处理（所有的请求都在该协程中进行）
    w, err := c.readRequest(ctx)
    ...

    // 内部转调ServeHTTP函数
    serverHandler{c.server}.ServeHTTP(w, w.req)
    ...
  }
}

// ServeHTTP
func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
  // sh.srv.Handler是前面的http.ListenAndServe(":8001", nil)传入的handler
  handler := sh.srv.Handler
  // 如果handler为空，就用默认的DefaultServeMux
  if handler == nil {
    handler = DefaultServeMux
  }
  if req.RequestURI == "*" && req.Method == "OPTIONS" {
    handler = globalOptionsHandler{}
  }
  // 这里就是调用ServeMux的ServeHTTP
  handler.ServeHTTP(rw, req)
}

// ServeHTTP
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
  ...
  h, _ := mux.Handler(r)
  h.ServeHTTP(w, r)
}

// Handler
func (mux *ServeMux) Handler(r *Request) (h Handler, pattern string) {
  ...
  return mux.handler(host, r.URL.Path)
}

// handler
func (mux *ServeMux) handler(host, path string) (h Handler, pattern string) {
  ...
  if mux.hosts {
    h, pattern = mux.match(host + path)
  }
  if h == nil {
    h, pattern = mux.match(path)
  }
  if h == nil {
    h, pattern = NotFoundHandler(), ""
  }
  return
}

// 匹配路由函数
func (mux *ServeMux) match(path string) (h Handler, pattern string) {
  // 先从前面介绍的ServeMux的m中精确查找路由表达式
  v, ok := mux.m[path]
  // 如果找到，直接返回handler
  if ok {
    return v.h, v.pattern
  }

  // 如果不能精确匹配，就去列表中找到最接近的路由
  // mux.es中的路由是按照从长到短排序的
  for _, e := range mux.es {
    if strings.HasPrefix(path, e.pattern) {
      return e.h, e.pattern
    }
  }
  return nil, ""
}
```