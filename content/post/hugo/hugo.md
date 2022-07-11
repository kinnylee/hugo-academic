## hugo 使用到的框架

- Chroma：代码高亮插件
- Goldmark: Markdown渲染工具

### CHroma配置说明

```YAML:
markup:
  defaultMarkdownHandler: goldmark
  goldmark:
    renderer:
      unsafe: true
    parser:
      attribute:
        block: true
        title: true
  highlight:
    codeFences: true # 代码围栏
    noHl: false
    lineNumbersInTable: true # 使用表来格式化行号的代码，而不是标签
    noClasses: false # 使用 class 标签，而不是内勤的内联样式
    guessSyntax: true # 代码联想
    lineNos: true # 是否显示行号
  tableOfContents:
    startLevel: 2
    endLevel: 3
```