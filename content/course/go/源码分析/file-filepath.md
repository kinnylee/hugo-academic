
---
title: go协程入门
author: kinnylee
date: 2020-09-14
toc: true
categories:
  - go
tags:
  - go
  - go基础入门
thumbnail: "images/go.png"
---

# Go文件路径操作

windows、linux下的文件路径差异比较大，go语言作为跨平台的语言，底层做了这种差异的封装。跟文件路径操作相关的源码主要是path、path/filepath这两个包。这两个包有重复的部分，大部分情况下建议使用path/filepath这个包

## 函数列表

- Clean
- ToSlash
- FromSlash
- SplitList
- Split
- Join
- Ext
- EvalSymlinks
- Abs
- Rel
- Walk
- Base
- Dir
- VolumeName
