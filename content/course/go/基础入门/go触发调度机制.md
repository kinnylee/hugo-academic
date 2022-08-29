---
title: go触发调度机制
author: kinnylee
date: 2020-08-28
toc: true
featured: true
categories:
  - go
tags:
  - go
  - go基础入门
  - 源码分析
thumbnail: "images/go.png"
---

# go调度机制

gopark 和上面的 goready 对应，互为逆操作。gopark 和 goready 在 runtime 的源码中会经常遇到，涉及了 goroutine 的调度过程，