# 主题配置说明

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
```
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