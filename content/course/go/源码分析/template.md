## 基本使用



## 源码分析

```go
type Template struct {
	name string // template的名称
	*parse.Tree // 解析树
	*common 
  leftDelim  string // 左边分隔符，默认为{{}}
	rightDelim string // 左边分隔符，默认为{{}} 
}

type common struct {
  // 所有的template共享同一个common
	tmpl   map[string]*Template // Map from name to defined templates.
	option option
	// We use two maps, one for parsing and one for execution.
	// This separation makes the API cleaner since it doesn't
	// expose reflection to the client.
	muFuncs    sync.RWMutex // protects parseFuncs and execFuncs
	parseFuncs FuncMap
	execFuncs  map[string]reflect.Value
}
```

