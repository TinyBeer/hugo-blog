---
date: "2026-01-02T17:43:53+08:00"
title: "Hugo -- giscus 评论系统"
tags: ["Hugo", "PaperModX", "giscus"]
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

本文将为 `PaperModX` 主题的博客添加 `giscus` 博客系统。

<!--more-->

在 [Hugo -- PaperModX 主题 ](/hugo-blog/note/hugo/hugo-papermodx/) 中，我们将博客主题切换到了 `PaperModX`，并进行了一些个性化配置。本文接续上次的工作，继续为博客添加博客系统。

## 代码管理

> 注意!!!  
> 改完代码后需要主仓库和子模块的代码改动都需要提交

由于 `PaperModX` 主题，原生没有对 `giscus` 评论系统进行支持，所以我们需要对主题进行魔改。这种情况主要有两种处理方式：

1. 将原仓库内容纳入到主仓库中一起进行管理，而非 `子模块`
2. Fork 一份代码到自己的账号下

基于对后续更新、维护等的考虑，本文采用第二种方式。如果觉得麻烦使用第一种也可以。

具体操作步骤如下：

1. Fork 主题仓库到自己的账号下
   进入[PaperModX 主题仓库](https://github.com/lguenth/hugo-PaperModX)，点击 `Fork` 按钮，为 `Fork` 的仓库设置仓库名（可选），确认 `Fork` 。
2. 将 `Fork` 好的仓库作为子模块添加到主仓库

   ```bash
   # 添加子模块
   git submodule add <Fork的仓库地址> themes/<子模块文件夹名>
   # 或
   git submodule add --depth=1 <Fork的仓库地址> themes/<子模块文件夹名>
   ```

   > 注意：如果需要在主仓库编辑子仓库内容不要添加 `--depth=1`

   考虑到后续可能需要移除一些不想要的主题（子模块），这里介绍一下如何移除子模块：

   ```bash
    # 列出所有子模块，格式：<提交哈希> <子模块路径> <子模块分支/标签>
    git submodule status

    # 移除路径为 "vendor/my-submodule" 的子模块（替换为你的子模块路径）
    git submodule deinit -f vendor/my-submodule

    # 示例：删除子模块路径（与步骤1中的路径保持一致）
    git rm -f vendor/my-submodule
   ```

3. 应用主题
   修改 `Hugo` 配置文件中的主题配置 `theme: <子模块文件夹名>`

## 添加评论区

### 获取 giscus 评论系统代码

进入 [giscus 官网](https://giscus.app) 下滑到 配置 标题处，根据提示填写你的 `github` 仓库信息，以及希望的评论区主题样式等信息可以得到一段 `script` 脚本：

```js
<script
  src="https://giscus.app/client.js"
  data-repo="[在此输入仓库]"
  data-repo-id="[在此输入仓库 ID]"
  data-category="[在此输入分类名]"
  data-category-id="[在此输入分类 ID]"
  data-mapping="pathname"
  data-strict="0"
  data-reactions-enabled="1"
  data-emit-metadata="0"
  data-input-position="bottom"
  data-theme="preferred_color_scheme"
  data-lang="zh-CN"
  crossorigin="anonymous"
  async
></script>
```

### 将评论区代码添加到主题模板中

找到主题仓库目录中 `layouts/partials/comments.html` 文件，这里是规划好的填写评论区模板的位置。  
可以看到这里已经有 `remark42` 、`telegram_widget` 、`disqus` 三个评论系统的代码了，我们可以直接在这里添加上 `giscus` 的代码，如：

```go-html-template
{{- /* You can add your own layouts/comments.html to override this file */ -}}

<script
  src="https://giscus.app/client.js"
  data-repo="[在此输入仓库]"
  data-repo-id="[在此输入仓库 ID]"
  data-category="[在此输入分类名]"
  data-category-id="[在此输入分类 ID]"
  data-mapping="pathname"
  data-strict="0"
  data-reactions-enabled="1"
  data-emit-metadata="0"
  data-input-position="bottom"
  data-theme="preferred_color_scheme"
  data-lang="zh-CN"
  crossorigin="anonymous"
  async
></script>


{{- $pageCommentSystems := .Param "pageCommentSystems"}}
{{- if not $pageCommentSystems }}
  {{- $pageCommentSystems = site.Params.defaultCommentSystems }}
{{- end }}

{{- $page := . -}}
{{- with site.Params.commentSystems -}}
  {{- if $pageCommentSystems.remark42 -}}
  {{- with .remark42 -}}
    {{- partial "remark42.html" (dict "page" $page "ctx" .) }}
  {{- end -}}
  {{- end -}}

  {{- if $pageCommentSystems.telegramWidget -}}
  {{- with .telegramWidget -}}
    {{- partial "telegram_widget.html" . }}
  {{- end -}}
  {{- end -}}

  {{- if $pageCommentSystems.disqus -}}
  {{- with .disqus -}}
    {{- partial "disqus.html" (dict "page" $page "ctx" .) }}
  {{- end -}}
  {{- end -}}
{{- end -}}

```

但处于保持原有结构的想法，我们在这里添加一个 `giscus` 的分支：

```go-html-template
{{- /* You can add your own layouts/comments.html to override this file */ -}}

{{- $pageCommentSystems := .Param "pageCommentSystems"}}
{{- if not $pageCommentSystems }}
  {{- $pageCommentSystems = site.Params.defaultCommentSystems }}
{{- end }}

{{- $page := . -}}
{{- with site.Params.commentSystems -}}
  {{- /* ！！！！分支添加到这里！！！！ */}}
  {{- if $pageCommentSystems.giscus -}}
  {{- with .giscus -}}
    {{- partial "giscus.html" (dict "page" $page "ctx" .) }}
  {{- end -}}
  {{- end -}}

  {{- if $pageCommentSystems.remark42 -}}
  {{- with .remark42 -}}
    {{- partial "remark42.html" (dict "page" $page "ctx" .) }}
  {{- end -}}
  {{- end -}}

  {{- if $pageCommentSystems.telegramWidget -}}
  {{- with .telegramWidget -}}
    {{- partial "telegram_widget.html" . }}
  {{- end -}}
  {{- end -}}

  {{- if $pageCommentSystems.disqus -}}
  {{- with .disqus -}}
    {{- partial "disqus.html" (dict "page" $page "ctx" .) }}
  {{- end -}}
  {{- end -}}
{{- end -}}
```

然后在 `layouts/partials` 目录下创建一个 `giscus.html` 文件夹，将评论区脚本拷贝进去。这里为了方便修改配置进行了参数提炼：

```go-html-template
<script src="https://giscus.app/client.js"
        data-repo="{{- .ctx.repo -}}"
        data-repo-id="{{- .ctx.repo_id -}}"
        data-category="{{- .ctx.category -}}"
        data-category-id="{{- .ctx.category_id -}}"
        data-mapping="{{- .ctx.mapping -}}"
        data-strict="{{- .ctx.strict -}}"
        data-reactions-enabled="{{- .ctx.reactions_enabled -}}"
        data-emit-metadata="{{- .ctx.emit_metadata -}}"
        data-input-position="{{- .ctx.position -}}"
        data-theme="{{- .ctx.theme -}}"
        data-lang="{{- .ctx.lang -}}"
        data-loading="{{- .ctx.loading -}}"
        crossorigin="{{- .ctx.crossorigin -}}"
        async>
</script>
```

最后在站点配置中的 `Params` 中添加评论系统配置即可，这里的参数可根据需求填充和修改：

```yml
params:
  defaultCommentSystems:
    giscus: true
  commentSystems:
    giscus: 
      repo: "[在此输入仓库]"
      repo_id: "[在此输入仓库 ID]"
      category: "[在此输入分类名]"
      category_id: "[在此输入分类 ID]"
      mapping: "pathname"
      position: "bottom"
      theme: "dark_dimmed"
      lang: "en"
      loading: "lazy"
      crossorigin: "anonymous"
      strict: "0"
      reactions_enabled: "1"
      emit_metadata: "0"
```

## 页面优化

### 移除页脚文章链接

为 `layouts/partials/post_nav_links.html` 加上判断行为：

```go-html-template
{{/* get pages from the current page's section */}}
{{- $sections := site.Params.mainSections }}
{{- if .Section }}
  {{- $sections = slice .Section }}
{{- end }}
{{- $pages := where (where site.RegularPages "Section" "in" $sections) "Params.hidden" "!=" true }}

{{- /* 判断是否需要 页脚文章链接 */}}
{{- if not site.Params.disablePostNavLink -}}
  {{- if and (gt (len $pages) 1) (in $pages . ) }}
  <nav class="paginav">
    {{- with $pages.Next . }}
    <a class="prev" href="{{ .Permalink }}">
      <span class="title">
        <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="feather feather-arrow-left" style="user-select: text;"><line x1="19" y1="12" x2="5" y2="12" style="user-select: text;"></line><polyline points="12 19 5 12 12 5" style="user-select: text;"></polyline></svg>&nbsp;
        {{- i18n "prev_page" }}</span>
      <br>
      <span>{{- .Name -}}</span>
    </a>
    {{- end }}
    {{- with $pages.Prev . }}
    <a class="next" href="{{ .Permalink }}">
      <span class="title">
        {{- i18n "next_page" -}}
        &nbsp;<svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="feather feather-arrow-right" style="user-select: text;"><line x1="5" y1="12" x2="19" y2="12" style="user-select: text;"></line><polyline points="12 5 19 12 12 19" style="user-select: text;"></polyline></svg>
      </span>
      <br>
      <span>{{- .Name -}}</span>
    </a>
    {{- end }}
  </nav>
  {{- end }}
{{- end }}

```

在站点配置中的 `Params` 中添加开关配置：

```yml
params:
  disablePostNavLink: true
```

## 参考资料

[PaperModX 文档](https://reorx.github.io/hugo-PaperModX/)  
[giscus 官网](https://giscus.app)
