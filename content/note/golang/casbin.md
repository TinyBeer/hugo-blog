---
date: "2026-08-10T11:23:52+08:00"
title: "Casbin -- 跨语言、多模型开源权限校验库"
tags: ["Casbin", "Access Control", "AC"]
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

Casbin 是一套跨语言、支持多种访问控制模型（RBAC/ABAC 等）的开源权限校验库，通过独立配置策略文件解耦业务代码与权限逻辑。<!--more-->

> ⚠️ 本文主要介绍 casbin 在 go 语言中的使用，关于 casbin 中的概念仅简单介绍，如需了解更多可以参考文末的 [参考资料](#参考资料)

## PERM 模型

在开始使用前，我们有必要先了解一下 casbin 中的 PERM 模型：

Policy（策略）+ Effect（效果）+ Request（请求）+ Matcher（匹配器），是 Casbin 权限框架的通用访问控制元模型，不是某一种权限模型，而是一套描述所有权限体系的通用语法。

### 核心组件

| 组件       | 作用                         | 标准定义                                                                                          |
| :--------- | :--------------------------- | :------------------------------------------------------------------------------------------------ |
| Request(r) | 定义一次访问请求的结构       | 三元组 r = sub, obj, actsub：访问者（用户 / 角色）obj：资源（接口 / 文件）act：操作（GET/DELETE） |
| Policy(p)  | 存储具体权限规则             | p = sub, obj, act, efteft：allow/deny（允许 / 拒绝）                                              |
| Matcher(m) | 匹配规则：请求如何和策略比对 | 表达式 m = r.sub == p.sub && r.obj == p.obj && r.act == p.act                                     |
| Effect(e)  | 多条策略冲突时的合并规则     | 如 e = some(where (p.eft == allow))：任意一条允许则放行                                           |

### 核心优势

- 统一抽象：一套语法兼容 RBAC、ABAC、ACL、RBAC+ABAC 混合权限；
- 配置驱动：切换权限逻辑只改配置文件，不用修改业务代码；
- 解耦权限与业务：权限规则外置，支持数据库 / 文件持久化。

### 模型示例

使用 PERM 模型定义 RBAC 模型：

```ini
# Request定义：谁、资源、操作
[request_definition]
r = sub, obj, act

# 权限策略格式
[policy_definition]
p = sub, obj, act

# 匹配规则
[matchers]
m = r.sub == p.sub && r.obj == p.obj && r.act == p.act

# 冲突效果：存在允许策略则放行
[policy_effect]
e = some(where (p.eft == allow))
```

1. `[request_definition]` 请求定义

   定义一次用户访问请求由哪几个参数组成，变量名 r 固定代表 request。
   - sub：subject 访问主体，发起请求的人 / 客户端，如 user_alice、admin
   - obj：object 资源对象，要访问的资源，如 /api/user、doc123
   - act：action 操作行为，对资源做什么，如 read、write、delete

   例如用户 alice 请求读取用户接口：`r = alice, /api/user, read`

2. `[policy_definition]` 权限策略定义

   p 代表 policy，定义权限规则的存储格式，每一条 p 就是一条权限。字段顺序必须和 matcher、实际策略数据对齐。

   > 补充：默认隐式带 eft=allow，所以配置里没写；如果需要拒绝策略会写成 p = sub, obj, act, eft。

   ```ini
    # admin 拥有 /api/user 的写权限
    # admin 拥有 /api/order 的读权限
    # guest 拥有 /api/public 的读权限

    p, admin, /api/user, write
    p, admin, /api/order, read
    p, guest, /api/public, read
   ```

3. `[matcher]` 匹配器（核心判断逻辑）

   m 是匹配表达式，用来判断当前请求是否匹配某一条权限策略，返回 true /false。
   1. `g(r.sub, p.sub)`  
      从角色关系 g 中查找：请求的用户 r.sub 是否属于策略对应的角色 p.sub  
      比如请求用户 alice，策略角色 admin，因为 g,alice,admin 存在，这一段为 true
   2. `r.obj == p.obj`  
      请求访问的资源 和 权限策略绑定的资源完全一致
   3. `r.act == p.act`  
      请求操作 和 权限策略允许的操作完全一致

4. `[policy_effect]` 策略效果（冲突规则）

   多条策略同时命中时，决定最终放行还是拒绝，是权限最终裁决规则。
   - some(xxx)：任意一条满足条件则整体结果为 true
   - p.eft == allow：策略效果为允许

5. `[role_definition]` 角色关联定义

   g = group，用来做用户 ↔ 角色 的绑定，实现 RBAC 角色体系。`_, _` 代表两个参数：用户, 角色。支持多级角色继承。如果不使用 RBAC，这一部分可以忽略。

   ```ini
   # 用户 alice 归属 admin 角色
   # 用户 bob 归属 guest 角色
   g, alice, admin
   g, bob, guest

   # alice 会自动继承 senior_admin + super_admin 所有权限。
   g, senior_admin, super_admin
   g, alice, senior_admin
   ```

6. 补充

- 例中精准匹配 RBAC，资源、操作必须完全相等；如果需要模糊匹配（通配符 \*），matcher 需要修改，例如 r.obj =~ p.obj
- 例中无 deny 策略，若需要黑白名单混合控制，需要扩展 p 字段加 eft，effect 改为 some(where p.eft == deny) ? false : some(where p.eft == allow)
- 例中所有变量 r/p/g/m/e 是 Casbin PERM 元模型固定标识，不能随意改名

## 快速开始

### 安装

casbin 至今已经完成了多伦迭代，有多个版本可以使用，这里我们使用 v3 版本。

```bash
go get github.com/casbin/casbin/v3
```

### 模型文件

在 casbin 的实际使用中，模型文件一旦确定了就很好修改了，这里我每直接使用 [PERM 模型](#perm-模型) 章节中的模型文件，文件名称为 `quick_model.conf`。

### 策略文件

```ini quick_policy.conf
# alice 对数据 data1, data2 拥有读权限
p, alice, data1, read
p, alice, data2, read

# bob 对数据 data1 拥有读写权限
p, bob, data1, read
p, bob, data1, write
```

### 示例代码

下面通过一段简单的代码，演示一下 casbin 库的使用，验证 alice 是否拥有对 data1 的写权限：

```golang
package main

import (
	"log"

	"github.com/casbin/casbin/v3"
)

func main() {
	// model.conf 为模型定义文件
	// policy.conf 为策略定义文件
	e, err := casbin.NewEnforcer("./quick_model.conf", "./quick_policy.conf")
	if err != nil {
		log.Fatal("failed to create enforcer, err:", err)
	}

	sub := "alice"
	obj := "data1"
	act := "wrte"
	ok, err := e.Enforce(sub, obj, act)
	if err != nil {
		log.Fatal("failed to enfore permission, err:", err)
	}

	log.Print("enforece permission result:", ok)
	// Output:
	// xxxx/xx/xx xx:xx:xx enforece permission result:false
}
```

## Adapter

在实际生产环境中，对于策略文件的维护是必不可少的，使用文件来管理并不是一个好方案，`casbin` 提供了 `Adapter` 来为各类存储系统（数据库、配置中心等）提供访问接口。

> 这里使用 `file-adapter` 介绍如何使用适配器。其他适配器的使用可以参考对应的代码仓库。 官网文档中已经整理了大量的适配器 [传送门](https://casbin.apache.org/docs/adapters)

代码执行后会生成和快速开始章节中 [策略文件](#策略文件) 功能相同的策略文件内容 `file_policy.conf` (需要先行创建好一个空文件):

```golang
package main

import (
	"log"

	"github.com/casbin/casbin/v3"
	fileadapter "github.com/casbin/casbin/v3/persist/file-adapter"
)

func main() {
	a := fileadapter.NewAdapter("./file_policy.conf")
	e, err := casbin.NewEnforcer("./file_model.conf", a)
	if err != nil {
		log.Fatal("failed to create enforcer, err:", err)
	}

	e.AddPolicy("alice", "data1", "read")
	e.AddPolicy("alice", "data1", "write")
	e.AddPolicy("alice", "data2", "read")

	e.AddPolicy("bob", "data1", "read")
	e.AddPolicy("bob", "data1", "write")

	e.RemovePolicy("alice", "data1", "write")

	err = e.SavePolicy()
	if err != nil {
		log.Fatal("failed to save policy, err:", err)
	}

	err = e.LoadPolicy()
	if err != nil {
		log.Fatal("failed to load policy, err:", err)
	}

	log.Print(e.GetPolicy())
    ...
}
```

## 从字符串中加载模型

相较于策略文件，模型文件几乎不需要修改，但是使用一个本地文件进行管理并不够优雅。通常我们会使用配置中心来管理。
casbin 中提供了从字符串加载模型的方法，这可以帮助我们加载从配置中心获取的模型配置：

```golang
import (
    "github.com/casbin/casbin/v3"
    "github.com/casbin/casbin/v3/model"
)

// Model text (e.g. from config).
text :=
`
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[role_definition]
g = _, _

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = g(r.sub, p.sub) && r.obj == p.obj && r.act == p.act
`
m, _ := model.NewModelFromString(text)

// Load the policy rules from the .CSV file adapter.
// Replace it with your adapter to avoid using files.
a := fileadapter.NewAdapter("examples/rbac_policy.csv")

// Create the enforcer.
e := casbin.NewEnforcer(m, a)
```

## 参考资料

[Casbin 官方文档](https://casbin.apache.org/docs/overview)  
[Go 每日一库之 casbin](https://darjun.github.io/2020/06/12/godailylib/casbin/)  
[一种基于元模型的访问控制策略描述语言](http://www.jos.org.cn/html/2020/2/5624.htm)
