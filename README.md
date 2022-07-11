# 主题配置说明

## 升级版本

一定要先升级版本，否则很多特性用不了，代码格式显示混乱！！！

[升级说明](https://wowchemy.com/docs/hugo-tutorials/update/)

1. 修改 go.mod中的依赖版本

```bash
require (
  github.com/wowchemy/wowchemy-hugo-themes/modules/wowchemy-plugin-netlify-cms main
  github.com/wowchemy/wowchemy-hugo-themes/modules/wowchemy-plugin-netlify main
  github.com/wowchemy/wowchemy-hugo-themes/modules/wowchemy/v5 main
)
```

2. 修改 config/_defualt/config.yaml 中的依赖版本

```yaml
module:
  imports:
    - path: github.com/wowchemy/wowchemy-hugo-themes/modules/wowchemy-plugin-netlify-cms
    - path: github.com/wowchemy/wowchemy-hugo-themes/modules/wowchemy-plugin-netlify
    - path: github.com/wowchemy/wowchemy-hugo-themes/modules/wowchemy/v5
```
## 修改图标

将图标保存为 512 x 512px, 保存路径为 assets/icon.png

## config/_default

- menus.yaml: 菜单栏配置
- config.yaml：基本信息配置
- param.yaml：参数配置

### param.yaml

[说明](https://wowchemy.com/docs/getting-started/customization/)

## 自定义主题

## 修改代码风格

[参考](https://wowchemy.com/docs/getting-started/customization/#code-syntax-highlighting)

```bash
mkdir -p assets/css/libs/chroma/
hugo gen chromastyles --style=xcode > assets/css/libs/chroma/xcode-light.css
hugo gen chromastyles --style=xcode-dark > assets/css/libs/chroma/xcode-dark.css
hugo gen chromastyles --style=solarized-dark > assets/css/libs/chroma/solarized-dark.css
hugo gen chromastyles --style=olarized-dark > assets/css/libs/chroma/olarized-dark.css
hugo gen chromastyles --style=base16-snazzy > assets/css/libs/chroma/base16-snazzy.css
hugo gen chromastyles --style=dracula > assets/css/libs/chroma/dracula.css
hugo gen chromastyles --style=emacs > assets/css/libs/chroma/emacs.css
```

同时修改主题
```yaml
features:
  syntax_highlighter:
    theme_light: xcode-dark
    theme_dark: xcode-dark
```

## 修改文章宽度

[参考](https://github.com/wowchemy/wowchemy-hugo-themes/discussions/2005)

新增 assets/scss/custom.scss

```css
.article-container {
  max-width: 1060px;
  padding: 0 20px 0 20px;
  margin: 0 auto 0 auto;
  }
```
## 新增 markdown 目录

[参考](https://github.com/wowchemy/wowchemy-hugo-themes/discussions/2706)
[参考2](https://wowchemy.com/docs/content/writing-markdown-latex/#table-of-contents)
正文新增 `{{< toc hide_on="xl">}}`

## 支持代码添加行号

config.yaml文件中新增：
```YAML
markup:
  _merge: deep
  highlight:
    lineNos: true
```
