---
title: go-map原理解析
author: kinnylee
date: 2020-09-08
toc: true
categories:
  - go
tags:
  - go
  - go基础入门
  - 源码分析
thumbnail: "images/go.png"
---

# map原理

- map数据结构
- map初始化
- 插入
- 删除
- 扩容
- 遍历
- NAN

## 结构体

源码位置：src/runtime/map.go

```Go
type hmap struct {
  // 元素数量
  count     int
  // 标识目前map的状态：写、扩容
  flags     uint8
  // log_2 桶数量
  B         uint8
  // 元素超出该数量认为溢出
  noverflow uint16
  hash0     uint32 // hash seed
  // 桶数组，指向bmap，长度为 2^B
  buckets    unsafe.Pointer
  // 扩容用的桶
  oldbuckets unsafe.Pointer
  // 扩容进度
  nevacuate  uintptr
  // 可选字段，额外属性
  extra *mapextra // optional fields
}

// 桶
type bmap struct {
  // 不仅仅只有这个字段，其他字段是在编译时加进去的，因为map中存放的数据类型不确定
  // 参考 src/cmd/compile/internal/gc/reflect.go的bmap函数

  // bucketCnt=8，也就是bmap中的桶固定只有8个格子，每个格子内存放一对key-value，
  // 表示key-value的字段并不在这里表示，而是编译时动态添加的
  tophash [bucketCnt]uint8

  // 其他编译生成的字段包括
  // overflow 用于连接下一个bmap（溢出桶）
}
```

## map初始化

```Go
func makemap(t *maptype, hint int, h *hmap) *hmap {
  mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
  if overflow || mem > maxAlloc {
    hint = 0
  }

  // initialize Hmap
  if h == nil {
    h = new(hmap)
  }
  h.hash0 = fastrand()

  // Find the size parameter B which will hold the requested # of elements.
  // For hint < 0 overLoadFactor returns false since hint < bucketCnt.
  B := uint8(0)
  for overLoadFactor(hint, B) {
    B++
  }
  h.B = B

  // allocate initial hash table
  // if B == 0, the buckets field is allocated lazily later (in mapassign)
  // If hint is large zeroing this memory could take a while.
  if h.B != 0 {
    var nextOverflow *bmap
    h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
    if nextOverflow != nil {
      h.extra = new(mapextra)
      h.extra.nextOverflow = nextOverflow
    }
  }

  return h
}
```