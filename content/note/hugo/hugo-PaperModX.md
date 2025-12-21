---
date: "2025-12-21T16:10:27+08:00"
title: "Hugo博客 -- PaperModX主题"
tags: ["Hugo", "PaperModX"]
categories: "笔记"
description: ""
draft: false
searchHidden: false

showToc: true
TocOpen: true
hidemeta: false
comments: false
ShowReadingTime: true
ShowBreadCrumbs: false
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: false
UseHugoToc: true
---

由于 `PaperMod` 无法将 目录 固定到右侧（刚需），所以决定切换到 `PapaerModX` 主题。

<!--more-->

## 简介

`PaperModX` 是大佬 fork PaperMod 主题后魔改的版本，添加了一些方便的特性：

1. 悬浮在右侧的目录
2. `InstantClick` 预加载
3. [Simple Icons](https://simpleicons.org/)社交图标
4. 方便的可选择的 UI 优化

## 切换主题

1. 添加主题
   ```bash
   git submodule add --depth=1 https://github.com/reorx/hugo-PaperModX.git themes/PaperModX
   ```
2. 切换主题
   将站点配置中的的配置修改为 `PaperModX`

```yaml
theme: "PaperModX"
```

## 主题配置

### 右侧悬浮目录

在站点配置的 `params` 中:

```yaml
params:
  TocSide: "left" # or 'right'
```

### InstantClick

`InstantClick` 是一个轻量的 JS 库，它的核心逻辑是提前加载用户可能会点击的链接对应的页面资源，当用户实际点击的时候，就可以直接展示已经加载好的内容，以此来降低页面跳转的延迟感，让博客的浏览体验更流畅。
在站点配置的 `params` 中开启:

```yaml
params:
  EnableInstantClick: true
```

### 修改链接颜色
添加 `assets/css/extended/custom.css` 文件，如：
```css
:root {
  --link-color: var(--primary);
  --link-hover-color: #573eaa;
  --link-underline-shadow: 0 1px 0 var(--link-color);
  --link-hover-underline-color: #573eaa;
  --link-hover-underline-shadow: 0 2px 0 var(--link-hover-underline-color);
}
```

### 外部链接标记
在 `menu` 中添加 参数 `params.external` :
```yaml
menu:
  main:
    - name: "@Author"
      url: "https://reorx.com"
      params:
        external: true
```

### Chroma 代码高亮
默认情况下，（代码高亮的）主题浅色模式为 `github`，深色模式为 `dracula`；你可以通过在网站的 `assets/css/lib` 目录下添加 `chroma-light.css` 和 `chroma-dark.css` 文件来修改这些主题。

### 社交图标
进入 [Simple Icons](https://simpleicons.org/) 查看对应图标

## 参考资料
[PaperModX 文档](https://reorx.github.io/hugo-PaperModX/)