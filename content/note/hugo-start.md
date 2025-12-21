---
date: "2025-12-20T16:59:04+08:00"
title: "使用Hugo搭建一个静态博客"
tags: ["Hugo", "PaperMod"]
categories: "笔记"
# description: "Hugo静态博客搭建笔记"
draft: true

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

记录一下使用 `Hugo` 搭建静态博客。本次搭建使用 `PaperMod` 主题。

<!--more-->

## 安装 Hugo

> 本次安装使用 Golang 编译，安装前请确保 `Git` 和 `Golang (1.24.0或更新版本)` 已安装并配置完成。

Hugo 提供了三个版本供用户选择，`standard`，`extended`以及`extended/deploy`。标准版本提供核心功能，另外两个版本则添加了一些其他特性。

| 特性                                                                           | extended 版本 | extended/deploy 版本 |
| :----------------------------------------------------------------------------- | :------------ | :------------------- |
| 编码 WebP 格式图片                                                             | ✔️            | ✔️                   |
| Sass 转 Css                                                                    | ✔️            | ✔️                   |
| 直接部署上云[详情](https://gohugo.io/host-and-deploy/deploy-with-hugo-deploy/) | ❌            | ✔️                   |

1. 根据需求确定好要安装的版本，执行对应安装脚本进行安装。

   ```bash
   ## standard 版本
   go install github.com/gohugoio/hugo@latest
   ## extended 版本
   CGO_ENABLED=1 go install -tags extended github.com/gohugoio/hugo@latest
   ## extended/deploy 版本
   CGO_ENABLED=1 go install -tags extended,withdeploy github.com/gohugoio/hugo@latest
   ```

2. 验证安装是否成功
   ```bash
   hugo version
   ## 如果安装成，会输出hugo当前版本信息
   ## 例如： hugo v0.153.0+extended linux/amd64 BuildDate=unknown
   ```

## 快速开始

```bash
## 将 MyFreshWebsite 替换为你喜欢的名字
hugo new site MyFreshWebsite --format yaml
cd MyFreshWebsite

git init
## 以子模块方式添加 PaperMod
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
## 配置使用 PaperMod 主题
ehco "theme: \"PaperMod\"" >> hugo.yaml
## 在本地启动 Hugo 服务
## 这是可以根据提示 访问 http://localhost:1313/ 查看主题是否成功配置
hugo server
```

- 主题(子模块)的维护

```bash
## 手动克隆子模块 克隆项目时没有克隆子模块时使用
git submodule update --init --recursive

## 更新子模块
git submodule update --remote --merge
```

## 创建文章

```bash
## 命令格式 hugo new content [path] [flags]
hugo new content/post/my-first-post.md
## 或者
hugo new content post/my-first-post.md
```

Hugo 会根据所填写的文章路径猜测文章的类型，比如上方给出的两个命令中 `post` 会被认定为创建文章的类型，此外如果使用的主题中指定了文章类型，也会被 Hugo 使用。
Hugo 使用模板配置在你所指的路径下生成文章。模板配置在根目录下的 `archetypes` 文件夹中，如果 Hugo 没有找到对应的模板配置，则会使用默认配置 `default.md`。

### 配置文章模板

如果我们需要创建新的文章模板，只需要在 `archetypes` 文件夹中添加一个 Markdown 文件即可，文件名为 `模板类型名.md`。  
模板配置的内容主是 `Front Matter(元信息)`， 下面给出一个 `note.md` 的模板样例：

```md
---
<!-- 当前日期 -->
date: '{{ .Date }}'
<!-- 将文件名中的空格替换为- 作为文章的标题 -->
title: '{{ replace .File.ContentBaseName "-" " " | title }}'
<!-- 空标签 -->
tags: []
<!-- 分类指定为笔记 -->
categories: "笔记"
<!-- 描述信息 -->
description: "Desc Text."
<!-- 草稿 -->
draft: true


showToc: true
TocOpen: false
hidemeta: false
comments: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
---
```

## 个性化配置

本次部署的配置项目较多，大家可以使用官方提供的配置模板，根据实际需求修改后使用。文末提供了本次部署使用的全部配置[传送门](#参考配置)

### 菜单栏配置

菜单栏配置路径为 `menu.main` ， 配置项目：

- identifier：区分不同的菜单
- name：菜单项实际显示内容
- weight：控制菜单项显示顺序
- url：导航到哪个路径

```yaml
menu:
  main:
    - identifier: archives
      name: 归档
      url: /archives/
      weight: 1
    - identifier: categories
      name: 分类
      url: /categories/
      weight: 2
    - identifier: tags
      name: 标签
      url: /tags/
      weight: 3
    - identifier: search
      name: 搜索
      url: /search/
      weight: 4
```

### 搜索页

在 `content` 目录下添加 `search.md` 文件， 内容如下：

> 记得在菜单栏为搜索页面添加入口

```markdown
---
title: "搜索" # 标题
layout: "search" # 指定页面为搜索页 别动
url: "/search" # 关联链接路径
# description: "Description for Search"
summary: "search"
placeholder: "请输入需要搜索的内容" # 占位提示信息
---
```

### 归档页面

和搜索页类似的，在 `content` 目录下添加 `archive,md` 文件，内容如下：

```markdown
---
title: "Archive" # 标题
layout: "archives" # 排版配置  别动
url: "/archives/" # 关联链接
summary: archives
---
```

### 汉化

1. 修改语言配置
   在 hugo.yaml 配置中完成对应修改
   ```yaml
   # 博客语言
   languageCode: zh-cn
   defaultContentLanguage: zh-cn
   defaultContentLanguageInSubdir: false
   ```
2. PaperMod 并没有附带中文支持，所以设置语言码为 `zh-cn` 后还需要添加一些配置
   在 `i18n` 文件夹下添加 `zh-cn.toml` 文件，内容如下：

```toml
# i18n/zh-cn.toml
[toc]
other = "目录"
```

此处列举而的并不详尽，如果遇到没有翻译的，大家可以自行手动添加。

## 部署
### github
1. 关联远程仓库
  ```bash
  git remote add origin <远程仓库地址>
  git branch -M main
  git add .
  git commit -m "first commit"
  git push -u origin main
  ```
2. (可选)在本地仓库添加 `.gitignore` 文件，忽略掉一个不必要的文件，文件内容如下：
  ```text
  .hugo_build.lock
  public/
  ```
3. 修改 git 仓库构建部署源
  将 `Setting > Pages > Build and deployment > Source` 配置由 `Deploy from a branch` 修改为 `Github Action`。
4. 修改图片缓存路径
  在站点配置文件 `hugo.yaml` 中添加以下配置:
  ```yaml
  caches:
    images:
      dir: :cacheDir/images
  ```
5. 添加 git 工作流配置
  添加配置文件 `.github/workflows/hugo.yaml`，可以执行一下命令：
  ```bash
  mkdir -p .github/workflows
  touch .github/workflows/hugo.yaml#
  ```
  具体配置放在文末[传送门](#github工作流配置)
6. 提交配置
  ```bash
  git add .
  git commit -m "github workflow"
  git push
  ```
7. 查看效果
  进入 github 仓库，选择 `Action` 选项，即可看到部署状态。
  
## 参考配置

### hugo.yaml
```yaml
baseURL: https://example.org/
# 博客语言
languageCode: zh-cn
defaultContentLanguage: zh-cn
defaultContentLanguageInSubdir: false
# 博客名称
title: "🍺Let's Beer!!"
# 主题 别动
theme: "PaperMod"
# 每页多少文章
# 官方文档中使用的 paginate 已经弃用
paginate.pagerSize: 5

# 方便搜索引擎爬取数据
enableRobotsTXT: true

buildDrafts: false
buildFuture: false
buildExpired: false

# Todo 不生效  需要研究一下 目前先注释掉
# googleAnalytics: UA-123-45

# 压缩部署文件大小
minify:
  disableXML: true
  minifyOutput: true

params:
  # 指定当前环境
  env: production # to enable google analytics, opengraph, twitter-cards and schema.
  # 站点标题
  title: "🍺Let's Beer!!"
  # 站点描述
  description: "TinyBeer"
  # 站点关键词
  keywords: [Blog, PaperMod]
  # 作者
  author: TinyBeer
  # author: ["Me", "You"] # multiple authors
  images: ["<link or path of image for opengraph, twitter-cards>"]
  # 日期格式
  DateFormat: "2006-01-02"
  # 默认主题色
  defaultTheme: auto # dark, light
  # 禁用主题选择
  disableThemeToggle: false
  # 显示阅读时间
  ShowReadingTime: true
  # 显示分享按钮
  ShowShareButtons: false
  # 显示文章导航链接
  ShowPostNavLinks: false
  # 显示面包屑导航
  ShowBreadCrumbs: false
  # 显示代码拷贝按钮
  ShowCodeCopyButtons: true
  # 显示字数统计
  ShowWordCount: true
  # 显示RSS订阅按钮
  ShowRssButtonInSectionTermList: flase
  disableSpecial1stPost: false
  # 关闭回到顶部按钮
  disableScrollToTop: false
  hideFooter: false

  languageAltTitle: “English”
  displayFullLangName: true

  assets:
    # disableHLJS: true # to disable highlight.js
    # disableFingerprinting: true
    favicon: "<link / abs url>"
    favicon16x16: "<link / abs url>"
    favicon32x32: "<link / abs url>"
    apple_touch_icon: "<link / abs url>"
    safari_pinned_tab: "<link / abs url>"

  label:
    text: "Beer"
    # icon: /apple-touch-icon.png
    iconHeight: 35

  # # profile-mode
  # profileMode:
  #   enabled: false # needs to be explicitly set
  #   title: ExampleSite
  #   subtitle: "This is subtitle"
  #   imageUrl: "<img location>"
  #   imageWidth: 120
  #   imageHeight: 120
  #   imageTitle: my image
  #   buttons:
  #     - name: Posts
  #       url: posts
  #     - name: Tags
  #       url: tags

  # # home-info mode
  # homeInfoParams:
  #   Title: "Hi there \U0001F44B"
  #   Content: Welcome to my blog

  # socialIcons:
  #   - name: x
  #     url: "https://x.com/"
  #   - name: stackoverflow
  #     url: "https://stackoverflow.com"
  #   - name: github
  #     url: "https://github.com/"

  analytics:
    google:
      SiteVerificationTag: "XYZabc"
    bing:
      SiteVerificationTag: "XYZabc"
    yandex:
      SiteVerificationTag: "XYZabc"

  cover:
    hidden: true # hide everywhere but not in structured data
    hiddenInList: true # hide on list pages and home
    hiddenInSingle: true # hide on single page

  # editPost:
  #   URL: "https://github.com/<path_to_repo>/content"
  #   Text: "Suggest Changes" # edit text
  #   appendFilePath: true # to append file path to Edit link

  # for search
  # https://fusejs.io/api/options.html
  fuseOpts:
    isCaseSensitive: false
    shouldSort: true
    location: 0
    distance: 1000
    threshold: 0.4
    minMatchCharLength: 0
    limit: 10 # refer: https://www.fusejs.io/api/methods.html#search
    keys: ["title", "permalink", "summary", "content"]

menu:
  main:
    - identifier: archives
      name: 归档
      url: /archives/
      weight: 1
    - identifier: categories
      name: 分类
      url: /categories/
      weight: 2
    - identifier: tags
      name: 标签
      url: /tags/
      weight: 3
    - identifier: search
      name: 搜索
      url: /search/
      weight: 4

# Read: https://github.com/adityatelange/hugo-PaperMod/wiki/FAQs#using-hugos-syntax-highlighter-chroma
pygmentsUseClasses: true
markup:
  highlight:
    noClasses: false
    # anchorLineNos: true
    # codeFences: true
    # guessSyntax: true
    # lineNos: true
    # style: monokai
outputs:
  home:
    - HTML
    - RSS
    - JSON # necessary for search
```

### note.md

```markdown
---
date: "{{ .Date }}"
title: '{{ replace .File.ContentBaseName "-" " " | title }}'
tags: []
categories: "笔记"
description: "Desc Text."
draft: true
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
```

### zh-cn.toml

```toml

```

### Github工作流配置
```yaml
# .github/workflows/hugo.yaml
name: Build and deploy
on:
  push:
    branches:
      - main
  workflow_dispatch:
permissions:
  contents: read
  pages: write
  id-token: write
concurrency:
  group: pages
  cancel-in-progress: false
defaults:
  run:
    shell: bash
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      DART_SASS_VERSION: 1.97.1
      GO_VERSION: 1.25.5
      HUGO_VERSION: 0.153.1
      NODE_VERSION: 24.12.0
      TZ: Europe/Oslo
    steps:
      - name: Checkout
        uses: actions/checkout@v5
        with:
          submodules: recursive
          fetch-depth: 0
      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: false
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5
      - name: Create directory for user-specific executable files
        run: |
          mkdir -p "${HOME}/.local"
      - name: Install Dart Sass
        run: |
          curl -sLJO "https://github.com/sass/dart-sass/releases/download/${DART_SASS_VERSION}/dart-sass-${DART_SASS_VERSION}-linux-x64.tar.gz"
          tar -C "${HOME}/.local" -xf "dart-sass-${DART_SASS_VERSION}-linux-x64.tar.gz"
          rm "dart-sass-${DART_SASS_VERSION}-linux-x64.tar.gz"
          echo "${HOME}/.local/dart-sass" >> "${GITHUB_PATH}"
      - name: Install Hugo
        run: |
          curl -sLJO "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
          mkdir "${HOME}/.local/hugo"
          tar -C "${HOME}/.local/hugo" -xf "hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
          rm "hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
          echo "${HOME}/.local/hugo" >> "${GITHUB_PATH}"
      - name: Verify installations
        run: |
          echo "Dart Sass: $(sass --version)"
          echo "Go: $(go version)"
          echo "Hugo: $(hugo version)"
          echo "Node.js: $(node --version)"
      - name: Install Node.js dependencies
        run: |
          [[ -f package-lock.json || -f npm-shrinkwrap.json ]] && npm ci || true
      - name: Configure Git
        run: |
          git config core.quotepath false
      - name: Cache restore
        id: cache-restore
        uses: actions/cache/restore@v4
        with:
          path: ${{ runner.temp }}/hugo_cache
          key: hugo-${{ github.run_id }}
          restore-keys:
            hugo-
      - name: Build the site
        run: |
          hugo \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/" \
            --cacheDir "${{ runner.temp }}/hugo_cache"
      - name: Cache save
        id: cache-save
        uses: actions/cache/save@v4
        with:
          path: ${{ runner.temp }}/hugo_cache
          key: ${{ steps.cache-restore.outputs.cache-primary-key }}
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

## 参考资料

[Hugo 官方文档](https://gohugo.io/)  
[PaperMod 官方文档](https://adityatelange.github.io/hugo-PaperMod/)
