---
date: "2026-01-13T15:05:58+08:00"
title: "Gin -- 高性能的 Web 框架"
tags: ["Golang", "Gin"]
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

`Gin` 是一款基于 `Go` 语言开发的 高性能 `HTTP Web` 微框架，是目前 `Go` 生态中最主流、使用率最高的 `Web` 框架。

<!--more-->

## 核心优势

- 🔥 极致高性能
- 🔥 超轻量 & 极简易用
- 🔥 强大的中间件生态
- 🔥 JSON 友好，原生支持
- 🔥 社区生态极其完善，文档齐全

## 快速开始

1. 创建新的 `go` 项目

   ```bash
   mkdir gin-quickstart && cd gin-quickstart
   go mod init gin-quickstart
   ```

2. 引入 `gin` 框架

   ```bash
   go get -u github.com/gin-gonic/gin
   ```

3. 创建 `main.go` 文件，内容如下：

   ```golang
    package main

   import "github.com/gin-gonic/gin"

   func main() {
      router := gin.Default()
      router.GET("/ping", func(c *gin.Context) {
         c.JSON(200, gin.H{
            "message": "pong",
         })
      })
      router.Run(":9999") // 默认监听 8080 端口
   }
   ```

4. 运行 `api` 服务器

   ```bash
   go run main.go
   ```

5. 访问 `http://localhost:9999/ping` ，可以看到以下响应：
   ```json
   { "message": "pong" }
   ```

## 状态码

快速开始中使用 `c.Json` 返回消息，其第一个参数 `200` 是 `http` 状态码，我们可以使用 `net/http` 中定义的常量替代硬编码，如：

> 这里需要引入包 `net/http`

```golang
c.JSON(http.StatusOK, gin.H{
   "message": "pong",
})
```

这样编码可以使代码逻辑更加清晰，配合插件还可以避免错写状态码。

`net/http` 中提供了所有 `http` 状态码，这里列举一些常用的：

> `http` 状态码编码规则  
> 1xx 信息性状态码（Informational）  
> 2xx 成功状态码（Success）  
> 3xx 重定向状态码（Redirection）  
> 4xx 客户端错误状态码（Client Error）  
> 5xx 服务端错误状态码（Server Error）

| 常量                           | 状态码 | 作用                                                                            |
| :----------------------------- | :----- | ------------------------------------------------------------------------------- |
| http.StatusOK                  | 200    | 请求成功 所有正常返回数据的接口（查询、详情等），默认状态码                     |
| http.StatusCreated             | 201    | 创建成功 新增资源（创建用户、订单、上传文件等）                                 |
| http.StatusAccepted            | 202    | 已接收，请求已受理但未处理完成 异步任务提交（如提交报表生成任务、后台数据同步） |
| http.StatusNoContent           | 204    | 无内容，请求成功但无需返回响应体 删除操作、更新操作无返回数据场景               |
| http.StatusMultipleChoices     | 300    | 多种选择，资源有多个地址供客户端选择 资源多版本分发，业务使用较少               |
| http.StatusMovedPermanently    | 301    | 永久重定向 旧域名 / 旧接口迁移到新地址，浏览器缓存新地址                        |
| http.StatusFound               | 302    | 临时重定向 登录后跳转首页、授权后回调页面，浏览器不缓存地址                     |
| http.StatusNotModified         | 304    | 未修改 缓存专用，资源未更新时返回，客户端使用本地缓存（节省带宽）               |
| http.StatusBadRequest          | 400    | 请求错误 参数校验失败、请求体格式错误（如 JSON 解析失败）、参数缺失             |
| http.StatusUnauthorized        | 401    | 未授权 未登录访问需要权限的接口、Token 过期 / 无效、身份认证失败                |
| http.StatusForbidden           | 403    | 禁止访问 已认证但无权限（如普通用户访问管理员接口）、IP 被封禁                  |
| http.StatusNotFound            | 404    | 资源不存在 URL 地址错误、访问的资源 ID 不存在（如查询不存在的用户 / 文章）      |
| http.StatusMethodNotAllowed    | 405    | 方法不允许 请求方法错误（如接口仅支持 POST，客户端用 GET 请求）                 |
| http.StatusConflict            | 409    | 冲突 资源冲突（如创建用户时用户名已存在、重复提交订单）                         |
| http.StatusTooManyRequests     | 429    | 请求过于频繁 接口限流专用，客户端短时间内请求次数超限                           |
| http.StatusInternalServerError | 500    | 服务器内部错误 代码 panic、数据库连接失败、业务逻辑异常等未知错误               |
| http.StatusServiceUnavailable  | 503    | 服务不可用 服务器维护中、负载过高、服务重启，可返回重试时间                     |

## 渲染

`Gin` 中支持多种渲染方式，包括 `String/XML/JSON/YAML/ProtoBuf/Template`。

### String

`Gin` 支持格式化渲染字符串

```golang
router.GET("/someString", func(c *gin.Context) {
   c.String(http.StatusOK, "hello %s", "tom")
})
```

`String` 方法签名 `String(code int, format string, values ...any)` 渲染规则同 `fmt.Printf`

### XML/JSON/YAML/ProtoBuf

通过示例演示如何将返回消息渲染成对应格式数据：

```golang
type Msg struct {
   Name    string `json:"juser" xml:"xuser" yaml:"yuser"`
   Message string
   Number  int
}

msg := &Msg{
   Name:    "tom",
   Message: "hello",
   Number:  3,
}

router.GET("/someXml", func(c *gin.Context) {
   c.XML(http.StatusOK, msg)
})
router.GET("/someJson", func(c *gin.Context) {
   c.JSON(http.StatusOK, msg)
})
router.GET("/someYaml", func(c *gin.Context) {
   c.YAML(http.StatusOK, msg)
})
router.GET("/someProtoBuf", func(c *gin.Context) {
   // 这里需要使用标准的proto生成结构体  这里只说明用法 无法运行
   c.ProtoBuf(http.StatusOK, nil /* some proto struct */)
})
```

结果：  
访问 `/someXml` 返回

```xml
<Msg>
  <xuser>tom</xuser>
  <Message>hello</Message>
  <Number>3</Number>
</Msg>
```

访问 `/someJson` 返回

```json
{ "juser": "tom", "Message": "hello", "Number": 3 }
```

访问 `/someYaml` 产生一个可下载的 yaml 文件 内容如下

```yml
yuser: tom
message: hello
number: 3
```

### PureJSON

通常，`JSON` 使用 `unicode` 替换特殊 `HTML` 字符，例如 `<` 变为 `\u003c`。如果要按字面对这些字符进行编码，则可以使用 `PureJSON` 。

> Go 1.6 及更低版本无法使用此功能。

```golang
msg := gin.H{
   "html": "<b>Hello, world!</b>",
}

router.GET("/json", func(c *gin.Context) {
   c.JSON(http.StatusOK, msg) // {"html":"\u003cb\u003eHello, world!\u003c/b\u003e"}
})

router.GET("/purejson", func(c *gin.Context) {
   c.PureJSON(http.StatusOK, msg) // {"html":"<b>Hello, world!</b>"}
})
```

### SecureJSON

使用 SecureJSON 防止 json 劫持。如果给定的结构是数组值，则默认预置 "while(1)," 到响应体。

```golang
// 你也可以使用自己的 SecureJSON 前缀
	// router.SecureJsonPrefix(")]}',\n")

router.GET("/someJSON", func(c *gin.Context) {
   names := []string{"lena", "austin", "foo"}

   // 将输出：while(1);["lena","austin","foo"]
   c.SecureJSON(http.StatusOK, names)
})
```

### JSONP

使用 `JSONP` 向不同域的服务器请求数据。如果查询参数存在回调，则将回调添加到响应体中。

```golang
router.GET("/JSONP", func(c *gin.Context) {
   data := map[string]interface{}{
   "foo": "bar",
   }

   // /JSONP?callback=x
   // 将输出：x({\"foo\":\"bar\"})
   c.JSONP(http.StatusOK, data)
})
```

### AsciiJSON

使用 `AsciiJSON` 生成具有转义的非 `ASCII` 字符的 `ASCII-only` JSON。

```golang
router.GET("/someJSON", func(c *gin.Context) {
   data := map[string]interface{}{
   "lang": "GO语言",
   "tag":  "<br>",
   }

   // 输出 : {"lang":"GO\u8bed\u8a00","tag":"\u003cbr\u003e"}
   c.AsciiJSON(http.StatusOK, data)
})
```

### Template

`Gin` 框架中可以使用 `Go` 模板语法渲染 `html` 用法如下：

> `Go` 模板用法可以参考 [Golang -- Template 模板渲染](/hugo-blog/note/golang/basic/template/)

1. 准备模板文件
   添加文件 `templates/index.tmpl`

   ```go-html-template
   {{ define "users/index.tmpl" }}
   <html><h1>
      {{ .title }}
      </h1>
      <p>Using users/index.tmpl</p>
   </html>
   {{ end }}
   ```

   `{{ define xxx }}` `{{ end }}` 使用来定义模板名称（渲染时通过模板名称选择模板，默认是文件名），用来解决不同目录下同文件名模板渲染问题

2. 解析（加载模板）  
   `Gin` 框架提供了三个方法加载模板 `LoadHTMLGlob` 、 `LoadHTMLFiles` 以及 `LoadHTMLFS`

   1. `LoadHTMLGlob` 是使用通配符匹配模板文件，底层使用 `filepath.Glob` 匹配文件，匹配规则如下:

      > 需要注意的是 `**` 不是递归匹配任意层级 它和 `*` 的匹配效果一样

      ```plaintext
      pattern:
         { term }
      term:
         '*'         matches any sequence of non-Separator characters
         '?'         matches any single non-Separator character
         '[' [ '^' ] { character-range } ']'
                     character class (must be non-empty)
         c           matches character c (c != '*', '?', '\\', '[')
         '\\' c      matches character c

      character-range:
         c           matches character c (c != '\\', '-', ']')
         '\\' c      matches character c
         lo '-' hi   matches character c for lo <= c <= hi
      ```

      ```golang
      // 匹配 template 下所有文件
      router.LoadHTMLGlob("templates/*")
      // 匹配 template 下所有tmpl文件
      router.LoadHTMLGlob("templates/*.tmpl")
      // 匹配 template 下一级目录中所有tmpl文件
      router.LoadHTMLGlob("templates/**/*.tmpl")
      ```

   2. `LoadHTMLFiles` 加载指定模板文件，精确匹配模式

      ```golang
      router.LoadHTMLFiles("./templates/index.tmpl")
      ```

   3. `LoadHTMLFS` 从 `http.FileSystem` 加载模板

      ```golang
      //go:embed templates
      var tmplFS embed.FS
      sub, _ := fs.Sub(tmplFS, "templates")
      r.LoadHTMLFS(http.FS(sub), "**/*.tmpl")
      ```

3. 渲染模板

   ```golang
   c.HTML(http.StatusOK, "users/index.tmpl", gin.H{
      "title": "Users",
   })
   ```

   其中 `users/index.tmpl` 是我们定义的模板名，`gin.H` 中是用来渲染的数据。

#### 自定义模板方法

```golang
import (
    "fmt"
    "html/template"
    "net/http"
    "time"

    "github.com/gin-gonic/gin"
)

func formatAsDate(t time.Time) string {
    year, month, day := t.Date()
    return fmt.Sprintf("%d/%02d/%02d", year, month, day)
}

func main() {
    router := gin.Default()
    router.Delims("{[{", "}]}")
    router.SetFuncMap(template.FuncMap{
        "formatAsDate": formatAsDate,
    })
    router.LoadHTMLFiles("./testdata/template/raw.tmpl")

    router.GET("/raw", func(c *gin.Context) {
        c.HTML(http.StatusOK, "raw.tmpl", map[string]interface{}{
            "now": time.Date(2017, 07, 01, 0, 0, 0, 0, time.UTC),
        })
    })

    router.Run(":8080")
}
```

```golang-html-template
Date: {[{.now | formatAsDate}]}
```

#### 自定义模板渲染器

```golang
import "html/template"

func main() {
  router := gin.Default()
  html := template.Must(template.ParseFiles("file1", "file2"))
  router.SetHTMLTemplate(html)
  router.Run(":8080")
}
```

#### 自定义分隔符

```golang
router := gin.Default()
router.Delims("{[{", "}]}")
router.LoadHTMLGlob("/path/to/templates")
```

#### 多模板（模板继承）渲染

参考 [多模板渲染](https://github.com/gin-contrib/multitemplate)

### 静态文件服务

`Gin` 通过静态文件服务提供文件资源服务能力。

```golang
func main() {
  router := gin.Default()
  router.Static("/assets", "./assets")
  router.StaticFS("/more_static", http.Dir("my_file_system"))
  router.StaticFile("/favicon.ico", "./resources/favicon.ico")

  router.Run(":9999")
}
```

## 参数解析

### URL 参数

```golang
  router.POST("/post", func(c *gin.Context) {

   id := c.Query("id") // 如果没有 id 参数解析结果为 空 字符串
   page := c.DefaultQuery("page", "0") // 如果没有 page 参数，解析结果为 "0"
   ...
  })
```

### PostForm 参数

```golang
  router.POST("/post", func(c *gin.Context) {

   message := c.PostForm("message") // 如果没有 message 参数解析结果为 空 字符串
   name := c.DefaultPostForm("name", "tom") // 如果没有 name 参数，解析结果为 "tom"
   ...
  })
```

### 路由参数

`Gin` 支持将访问路径作为参数解析：

```golang
func main() {
  router := gin.Default()

  // 此 handler 将匹配 /user/john 但不会匹配 /user/ 或者 /user
  router.GET("/user/:name", func(c *gin.Context) {
    name := c.Param("name")
    c.String(http.StatusOK, "Hello %s", name)
  })

  // 此 handler 将匹配 /user/john/ 和 /user/john/send
  // 如果没有其他路由匹配 /user/john，它将重定向到 /user/john/
  router.GET("/user/:name/*action", func(c *gin.Context) {
    name := c.Param("name")
    action := c.Param("action")
    message := name + " is " + action
    c.String(http.StatusOK, message)
  })

  router.Run(":8080")
}
```

- `:` 匹配单段路径参数，遇到下一个 / 就会立即停止匹配
- `*` 匹配剩余所有路径参数

### 模型绑定与验证

`Gin` 提供了方便的模型绑定功能 和 参数验证功能。  
模型绑定： 可以将访问参数直接与结构体绑定，目前支持 `JSON`、`XML`、`YAML` 和 标准表单值的绑定。  
参数验证： `Gin`使用 [`go-playground/validator/v10`](https://gin-gonic.com/zh-cn/docs/examples/binding-and-validation/) 进行验证, [文档](https://pkg.go.dev/github.com/go-playground/validator/v10#hdr-Baked_In_Validators_and_Tags)。

使用时，需要在要绑定的所有字段上，设置相应的 `tag` 。 例如，使用 `JSON` 绑定时，设置字段标签为 `json:"fieldname"`。

`Gin` 提供了两类绑定方法：

1. Must Bind  
   方法：`Bind`, `BindJSON`, `BindXML`, `BindQuery`, `BindYAML`  
   行为：这些方法属于 `MustBindWith` 的具体调用。 如果发生绑定错误，则请求终止，并触发 `c.AbortWithError(400, err).SetType(ErrorTypeBind)`。响应状态码被设置为 `400` 并且 `Content-Type` 被设置为 `text/plain; charset=utf-8`。 如果您在此之后尝试设置响应状态码，`Gin` 会输出日志 `[GIN-debug] [WARNING] Headers were already written. Wanted to override status code 400 with 422`。 如果您希望更好地控制绑定，考虑使用 `ShouldBind` 等效方法。
2. Should Bind  
   方法：`ShouldBind`, `ShouldBindJSON`, `ShouldBindXML`, `ShouldBindQuery`, `ShouldBindYAML`  
   行为：这些方法属于 `ShouldBindWith` 的具体调用。 如果发生绑定错误，`Gin` 会返回错误并由开发者处理错误和请求。

> 使用 `Bind` 方法时，`Gin` 会尝试根据 `Content-Type` 推断如何绑定。 如果你明确知道要绑定什么，可以使用 `MustBindWith` 或 `ShouldBindWith`。  
> 你也可以指定必须绑定的字段。 如果一个字段的 `tag` 加上了 `binding:"required"`，但绑定时是空值, `Gin` 会报错。
> 如果结构体的某个字段本身也是一个结构体（嵌套结构体），为了正确验证，该结构体的字段也需要添加 `binding:"required"` 标签。

> `Gin` 会按照下面的顺序解析请求中的数据完成绑定：
> 如果是 `GET` 请求，只使用 `Form` 绑定引擎（`query`）。
> 如果是 `POST` 请求，首先检查 `content-type` 是否为包含支持的类型，然后再使用 `Form`（`form-data`）。

示例：

```golang
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

// 绑定 JSON
type Login struct {
	User     string `form:"user" json:"user" xml:"user"  binding:"required"`
	Password string `form:"password" json:"password" xml:"password" binding:"required"`
}

func main() {
	router := gin.Default()

	// 绑定 JSON ({"user": "manu", "password": "123"})
	router.POST("/loginJSON", func(c *gin.Context) {
		var json Login
		if err := c.ShouldBindJSON(&json); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}

		if json.User != "manu" || json.Password != "123" {
			c.JSON(http.StatusUnauthorized, gin.H{"status": "unauthorized"})
			return
		}

		c.JSON(http.StatusOK, gin.H{"status": "you are logged in"})
	})

	// 绑定 XML (
	//  <?xml version="1.0" encoding="UTF-8"?>
	//  <root>
	//    <user>manu</user>
	//    <password>123</password>
	//  </root>)
	router.POST("/loginXML", func(c *gin.Context) {
		var xml Login
		if err := c.ShouldBindXML(&xml); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}

		if xml.User != "manu" || xml.Password != "123" {
			c.JSON(http.StatusUnauthorized, gin.H{"status": "unauthorized"})
			return
		}

		c.JSON(http.StatusOK, gin.H{"status": "you are logged in"})
	})

	// 绑定 HTML 表单 (user=manu&password=123)
	router.POST("/loginForm", func(c *gin.Context) {
		var form Login
		// 根据 Content-Type Header 推断使用哪个绑定器。
		if err := c.ShouldBind(&form); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}

		if form.User != "manu" || form.Password != "123" {
			c.JSON(http.StatusUnauthorized, gin.H{"status": "unauthorized"})
			return
		}

		c.JSON(http.StatusOK, gin.H{"status": "you are logged in"})
	})

	router.Run(":9999") // 默认监听 8080 端口
}

```

测试：

```bash
curl -v -X POST \                                                                                                                                                   beer@debian
  http://localhost:9999/loginJSON \
  -H 'content-type: application/json' \
  -d '{ "user": "manu" }'
Note: Unnecessary use of -X or --request, POST is already inferred.
* Host localhost:9999 was resolved.
* IPv6: ::1
* IPv4: 127.0.0.1
*   Trying [::1]:9999...
* Connected to localhost (::1) port 9999
* using HTTP/1.x
> POST /loginJSON HTTP/1.1
> Host: localhost:9999
> User-Agent: curl/8.14.1
> Accept: */*
> content-type: application/json
> Content-Length: 18
>
* upload completely sent off: 18 bytes
< HTTP/1.1 400 Bad Request
< Content-Type: application/json; charset=utf-8
< Date: Thu, 15 Jan 2026 03:43:19 GMT
< Content-Length: 100
<
* Connection #0 to host localhost left intact
{"error":"Key: 'Login.Password' Error:Field validation for 'Password' failed on the 'required' tag"}%
```

由于 `password` 字段添加了 `Tag` `binding:"required"` 而测试时没有 `Password` 参数，所以会报错。可以使用 `binding:"-"` 或者 去掉 `binding:"required"` 取消验证。

### 文件上传

#### 单文件

```golang
func main() {
  router := gin.Default()
  // 为 multipart forms 设置较低的内存限制 (默认是 32 MiB)
  router.MaxMultipartMemory = 8 << 20  // 8 MiB
  router.POST("/upload", func(c *gin.Context) {
    // 单文件
    file, _ := c.FormFile("file")
    log.Println(file.Filename)

    dst := "./file" + file.Filename
    // 上传文件至指定的完整文件路径
    c.SaveUploadedFile(file, dst)

    c.String(http.StatusOK, fmt.Sprintf("'%s' uploaded!", file.Filename))
  })
  router.Run(":9999")
}
```

测试：

```bash
curl -X POST http://localhost:9999/upload \
  -F "file=@/Users/appleboy/test.zip" \
  -H "Content-Type: multipart/form-data"
```

#### 多文件

```golang
func main() {
  router := gin.Default()
  // 为 multipart forms 设置较低的内存限制 (默认是 32 MiB)
  router.MaxMultipartMemory = 8 << 20  // 8 MiB
  router.POST("/upload", func(c *gin.Context) {
    // Multipart form
    form, _ := c.MultipartForm()
    files := form.File["upload[]"]

    for _, file := range files {
      log.Println(file.Filename)

      // 上传文件至指定目录
      c.SaveUploadedFile(file, "./files/" + file.Filename)
    }
    c.String(http.StatusOK, fmt.Sprintf("%d files uploaded!", len(files)))
  })
  router.Run(":9999")
}
```

测试：

```bash
curl -X POST http://localhost:9999/upload \
  -F "upload[]=@/Users/appleboy/test1.zip" \
  -F "upload[]=@/Users/appleboy/test2.zip" \
  -H "Content-Type: multipart/form-data"
```

## Cookie

设置和使用 `cookie`

```golang
router.GET("/cookie", func(c *gin.Context) {

   cookie, err := c.Cookie("gin_cookie")

   if err != nil {
      cookie = "NotSet"
      c.SetCookie("gin_cookie", "test", 3600, "/", "localhost", false, true)
   }

   fmt.Printf("Cookie value: %s \n", cookie)
})

```

`SetCookie` 参数说明：

| 形参     | 类型   | 说明                         |
| :------- | :----- | :--------------------------- |
| name     | string | Cookie 名称                  |
| value    | string | Cookie 值                    |
| maxAge   | int    | 过期时间(秒)                 |
| path     | string | Cookie 生效路径              |
| domain   | string | Cookie 生效域名              |
| secure   | bool   | 是否仅 HTTPS 下传输          |
| httpOnly | bool   | 是否禁止前端 JS 读取(防 XSS) |

## 路由

### 普通路由

```golang
router.GET("/index", func(c *gin.Context) {...})
router.GET("/login", func(c *gin.Context) {...})
router.POST("/login", func(c *gin.Context) {...})
...
router.Any("/test", func(c *gin.Context) {...}) // 匹配所有方法
router.NoRoute(func(c *gin.Context) {...}) // 未匹配到路由 默认返回 状态码404
```

### 路由组

> 通常我们将路由分组用在 划分业务逻辑 或 划分 API 版本 时。

```golang
func main() {
  router := gin.Default()

  // 简单的路由组: v1 这里添加 {} 是为了让代码结构更清晰
  {
    v1 := router.Group("/v1")
    v1.POST("/login", loginEndpoint)
    v1.POST("/submit", submitEndpoint)
    v1.POST("/read", readEndpoint)
  }

  // 简单的路由组: v2
  {
    v2 := router.Group("/v2")
    v2.POST("/login", loginEndpoint)
    v2.POST("/submit", submitEndpoint)
    v2.POST("/read", readEndpoint)
  }

   // 当然路由组也可以嵌套

   {
      v2User := v2.Group("/user")
      ...
   }

  router.Run(":9999")
}
```

## 重定向

### HTTP 重定向

```golang
router.GET("/test", func(c *gin.Context) {
  c.Redirect(http.StatusMovedPermanently, "http://www.google.com/")
})
```

通过 `POST` 方法进行 HTTP 重定向

```golang
router.POST("/test", func(c *gin.Context) {
  c.Redirect(http.StatusFound, "/foo")
})
```

### 路由重定向

通过路由重定向可以将访问交由其他路由处理

```golang
router.GET("/test", func(c *gin.Context) {
    c.Request.URL.Path = "/test2"
    router.HandleContext(c)
})
router.GET("/test2", func(c *gin.Context) {
    c.JSON(http.StatusOK, gin.H{"hello": "world"})
})
```

## 中间件

`Gin` 框架中通过中间件来进行请求处理中的公共部分，如：登录认证、权限校验、数据分页、记录日志、耗时统计等。  
`Gin` 为我们提供了一些常用的中间件：

- `Logger`：日志中间件默认会输出请求的时间、耗时、方法、状态码、访问路径信息等。
- `Recovery`：`panic` 处理中间件，使用状态码 500 返回错误信息
- `BasicAuth`：基础鉴权中间件，支持账号密码验证，需要传入账号秘密信息

中间件的函数签名为 `type HandlerFunc func(*Context)`，我们可以自行定义：

```golang
func Logger() gin.HandlerFunc {
  return func(c *gin.Context) {
    t := time.Now()

    // 设置 example 变量
    c.Set("example", "12345")

    // 请求前

    c.Next()

    // 请求后
    latency := time.Since(t)
    log.Print(latency)

    // 获取发送的 status
    status := c.Writer.Status()
    log.Println(status)
  }
}
```

我们通过 `Use` 来加载中间件，中间件加载顺序决定了其应用顺序：

> 使用 `gin.Default()` 会默认加载 `Gin` 默认的 日志中间件 和 `panic` 处理中间件，可以使用 `gin.New()` 创建无中间件的引擎

```golang
router := gin.New()
rrouter.Use(Logger()， /* 其他中间件 */...)
```

## 注意事项

1. 当在中间件或 `handler` 中启动新的 `Goroutine` 时，不能使用原始的上下文，必须使用只读副本。

   ```golang
   router.GET("/long_async", func(c *gin.Context) {
      // 创建在 goroutine 中使用的副本
      cCp := c.Copy()
      go func() {
         // 用 time.Sleep() 模拟一个长任务。
         time.Sleep(5 * time.Second)

         // 请注意您使用的是复制的上下文 "cCp"，这一点很重要
         log.Println("Done! in path " + cCp.Request.URL.Path)
      }()
   })
   ```

## 参考资料

[Gin 官方文档](https://gin-gonic.com/)  
[Gin 框架介绍及使用](https://www.liwenzhou.com/posts/Go/gin/)
