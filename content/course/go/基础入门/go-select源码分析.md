---
title: go-select源码分析
author: kinnylee
date: 2020-08-27
toc: true
categories:
  - go
tags:
  - go
  - go基础入门
  - 源码分析
thumbnail: "images/go.png"
---

## select简介

### select使用

下面是select的最简单的用法：

```Go

```

### select源码入口

select在源码中也没有对应的实现，而是通过编译器将相关符号翻译为底层实现。
使用以下命令将go源码翻译为汇编

```bash
go tool compile -N -l -S main.go>hello.s
```

查看部分带有CALL指令的核心内容如下：

```armasm
0x0102 00258 (main.go:64) CALL runtime.selectgo(SB)
```

可以猜测对应关系：

- select语句对应：runtime.selectgo函数

相关源码只需要到runtime包下，全局搜索就可以找到在文件runtime/chan.go下

```Go
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {}
```

## 源码分析

### case数据结构

```Go
type scase struct {
  c           *hchan         // chan
  elem        unsafe.Pointer // data element
  kind        uint16
  pc          uintptr // race pc (for race detector / msan)
  releasetime int64
}
```

### 编译器

源码位置：src/cmd/compile/internal/gc/select.go

```Go
func walkselect(sel *Node) {
  lno := setlineno(sel)
  if sel.Nbody.Len() != 0 {
    Fatalf("double walkselect")
  }

  init := sel.Ninit.Slice()
  sel.Ninit.Set(nil)

  init = append(init, walkselectcases(&sel.List)...)
  sel.List.Set(nil)

  sel.Nbody.Set(init)
  walkstmtlist(sel.Nbody.Slice())

  lineno = lno
}

func walkselectcases(cases *Nodes) []*Node {
  n := cases.Len()
  sellineno := lineno

  // optimization: zero-case select
  // 如果没有case，直接调用block函数，内部是调用的gopark阻塞
  if n == 0 {
    return []*Node{mkcall("block", nil, nil)}
  }

  // optimization: one-case select: single op.
  // TODO(rsc): Reenable optimization once order.go can handle it.
  // golang.org/issue/7672.
  // 只有一个case的情况
  if n == 1 {
    cas := cases.First()
    setlineno(cas)
    l := cas.Ninit.Slice()
    // 没有default的处理逻辑
    if cas.Left != nil { // not default:
      n := cas.Left
      l = append(l, n.Ninit.Slice()...)
      n.Ninit.Set(nil)
      var ch *Node
      switch n.Op {
      default:
        Fatalf("select %v", n.Op)

        // ok already
      case OSEND:
        ch = n.Left

      case OSELRECV, OSELRECV2:
        ch = n.Right.Left
        if n.Op == OSELRECV || n.List.Len() == 0 {
          if n.Left == nil {
            n = n.Right
          } else {
            n.Op = OAS
          }
          break
        }

        if n.Left == nil {
          nblank = typecheck(nblank, ctxExpr|ctxAssign)
          n.Left = nblank
        }

        n.Op = OAS2
        n.List.Prepend(n.Left)
        n.Rlist.Set1(n.Right)
        n.Right = nil
        n.Left = nil
        n.SetTypecheck(0)
        n = typecheck(n, ctxStmt)
      }

      // if ch == nil { block() }; n;
      a := nod(OIF, nil, nil)

      a.Left = nod(OEQ, ch, nodnil())
      var ln Nodes
      ln.Set(l)
      a.Nbody.Set1(mkcall("block", nil, &ln))
      l = ln.Slice()
      a = typecheck(a, ctxStmt)
      l = append(l, a, n)
    }

    l = append(l, cas.Nbody.Slice()...)
    l = append(l, nod(OBREAK, nil, nil))
    return l
  }

  // convert case value arguments to addresses.
  // this rewrite is used by both the general code and the next optimization.
  // 多个case的情况，把每个分支转换为case结构体
  for _, cas := range cases.Slice() {
    setlineno(cas)
    n := cas.Left
    if n == nil {
      continue
    }
    switch n.Op {
    case OSEND:
      n.Right = nod(OADDR, n.Right, nil)
      n.Right = typecheck(n.Right, ctxExpr)

    case OSELRECV, OSELRECV2:
      if n.Op == OSELRECV2 && n.List.Len() == 0 {
        n.Op = OSELRECV
      }

      if n.Left != nil {
        n.Left = nod(OADDR, n.Left, nil)
        n.Left = typecheck(n.Left, ctxExpr)
      }
    }
  }

  // optimization: two-case select but one is default: single non-blocking op.
  if n == 2 && (cases.First().Left == nil || cases.Second().Left == nil) {
    var cas *Node
    var dflt *Node
    if cases.First().Left == nil {
      cas = cases.Second()
      dflt = cases.First()
    } else {
      dflt = cases.Second()
      cas = cases.First()
    }

    n := cas.Left
    setlineno(n)
    r := nod(OIF, nil, nil)
    r.Ninit.Set(cas.Ninit.Slice())
    switch n.Op {
    default:
      Fatalf("select %v", n.Op)

    case OSEND:
      // if selectnbsend(c, v) { body } else { default body }
      ch := n.Left
      r.Left = mkcall1(chanfn("selectnbsend", 2, ch.Type), types.Types[TBOOL], &r.Ninit, ch, n.Right)

    case OSELRECV:
      // if selectnbrecv(&v, c) { body } else { default body }
      r = nod(OIF, nil, nil)
      r.Ninit.Set(cas.Ninit.Slice())
      ch := n.Right.Left
      elem := n.Left
      if elem == nil {
        elem = nodnil()
      }
      r.Left = mkcall1(chanfn("selectnbrecv", 2, ch.Type), types.Types[TBOOL], &r.Ninit, elem, ch)

    case OSELRECV2:
      // if selectnbrecv2(&v, &received, c) { body } else { default body }
      r = nod(OIF, nil, nil)
      r.Ninit.Set(cas.Ninit.Slice())
      ch := n.Right.Left
      elem := n.Left
      if elem == nil {
        elem = nodnil()
      }
      receivedp := nod(OADDR, n.List.First(), nil)
      receivedp = typecheck(receivedp, ctxExpr)
      r.Left = mkcall1(chanfn("selectnbrecv2", 2, ch.Type), types.Types[TBOOL], &r.Ninit, elem, receivedp, ch)
    }

    r.Left = typecheck(r.Left, ctxExpr)
    r.Nbody.Set(cas.Nbody.Slice())
    r.Rlist.Set(append(dflt.Ninit.Slice(), dflt.Nbody.Slice()...))
    return []*Node{r, nod(OBREAK, nil, nil)}
  }

  var init []*Node

  // generate sel-struct
  lineno = sellineno
  selv := temp(types.NewArray(scasetype(), int64(n)))
  r := nod(OAS, selv, nil)
  r = typecheck(r, ctxStmt)
  init = append(init, r)

  order := temp(types.NewArray(types.Types[TUINT16], 2*int64(n)))
  r = nod(OAS, order, nil)
  r = typecheck(r, ctxStmt)
  init = append(init, r)

  // register cases
  for i, cas := range cases.Slice() {
    setlineno(cas)

    init = append(init, cas.Ninit.Slice()...)
    cas.Ninit.Set(nil)

    // Keep in sync with runtime/select.go.
    const (
      caseNil = iota
      caseRecv
      caseSend
      caseDefault
    )

    var c, elem *Node
    var kind int64 = caseDefault

    if n := cas.Left; n != nil {
      init = append(init, n.Ninit.Slice()...)

      switch n.Op {
      default:
        Fatalf("select %v", n.Op)
      case OSEND:
        kind = caseSend
        c = n.Left
        elem = n.Right
      case OSELRECV, OSELRECV2:
        kind = caseRecv
        c = n.Right.Left
        elem = n.Left
      }
    }

    setField := func(f string, val *Node) {
      r := nod(OAS, nodSym(ODOT, nod(OINDEX, selv, nodintconst(int64(i))), lookup(f)), val)
      r = typecheck(r, ctxStmt)
      init = append(init, r)
    }

    setField("kind", nodintconst(kind))
    if c != nil {
      c = convnop(c, types.Types[TUNSAFEPTR])
      setField("c", c)
    }
    if elem != nil {
      elem = convnop(elem, types.Types[TUNSAFEPTR])
      setField("elem", elem)
    }

    // TODO(mdempsky): There should be a cleaner way to
    // handle this.
    if instrumenting {
      r = mkcall("selectsetpc", nil, nil, bytePtrToIndex(selv, int64(i)))
      init = append(init, r)
    }
  }

  // run the select
  lineno = sellineno
  chosen := temp(types.Types[TINT])
  recvOK := temp(types.Types[TBOOL])
  r = nod(OAS2, nil, nil)
  r.List.Set2(chosen, recvOK)

  // 调用selectgo，这个函数是决定了如何选择分支
  fn := syslook("selectgo")
  r.Rlist.Set1(mkcall1(fn, fn.Type.Results(), nil, bytePtrToIndex(selv, 0), bytePtrToIndex(order, 0), nodintconst(int64(n))))
  r = typecheck(r, ctxStmt)
  init = append(init, r)

  // selv and order are no longer alive after selectgo.
  init = append(init, nod(OVARKILL, selv, nil))
  init = append(init, nod(OVARKILL, order, nil))

  // dispatch cases
  for i, cas := range cases.Slice() {
    setlineno(cas)

    cond := nod(OEQ, chosen, nodintconst(int64(i)))
    cond = typecheck(cond, ctxExpr)
    cond = defaultlit(cond, nil)

    r = nod(OIF, cond, nil)

    if n := cas.Left; n != nil && n.Op == OSELRECV2 {
      x := nod(OAS, n.List.First(), recvOK)
      x = typecheck(x, ctxStmt)
      r.Nbody.Append(x)
    }

    r.Nbody.AppendNodes(&cas.Nbody)
    r.Nbody.Append(nod(OBREAK, nil, nil))
    init = append(init, r)
  }

  return init
}
```

### select.go

```go
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
  if debugSelect {
    print("select: cas0=", cas0, "\n")
  }

  cas1 := (*[1 << 16]scase)(unsafe.Pointer(cas0))
  order1 := (*[1 << 17]uint16)(unsafe.Pointer(order0))

  scases := cas1[:ncases:ncases]
  pollorder := order1[:ncases:ncases]
  lockorder := order1[ncases:][:ncases:ncases]

  // 生成随机顺序
  // generate permuted order
  for i := 1; i < ncases; i++ {
    j := fastrandn(uint32(i + 1))
    pollorder[i] = pollorder[j]
    pollorder[j] = uint16(i)
  }
  }
```

