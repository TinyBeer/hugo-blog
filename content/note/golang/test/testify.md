---
date: "2026-01-12T13:49:18+08:00"
title: "Golang -- 单元测试之 testify"
tags: ["Golang", "Unit Test", "testify"]
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

`testify` 是 `Go` 语言生态中最主流、最常用的第三方单元测试库，也是标准库 `testing` 的绝佳增强补充，几乎是 `Go` 项目单元测试的标配。通过 `testify` 的使用可以让测试代码更简洁、断言更易读、测试逻辑更清晰。

<!--more-->

其核心作用是大幅简化单元测试代码、提供语义更清晰的断言、内置常用测试工具，解决了标准库 `testing` 断言能力弱、写法繁琐的痛点。`testify` 完全兼容 `Go` 原生的 `testing` 包，无缝接入现有测试流程，无需改造原有测试结构，同时提供 `Mock/Suite` 等高频测试能力。

## 安装 && 更新

```bash
# 安装命令
go get github.com/stretchr/testify
# 更新命令
go get -u github.com/stretchr/testify
```

## assert 包

`testify` 包含多个实用子包，日常开发中 99% 的场景只会用到 3 个核心组件，按使用频率排序：`assert`、`require`、`mock`，各司其职，分工明确。

由于 `assert` 包含大量语义化的断言工具，其功能完全可以通过函数/方法名称了解，这里出于篇幅考虑就不进行完整的介绍了，仅通过示例方式展示说明一些特性。

### 断言函数

简单示例：

```golang
import (
	"testing"

	"github.com/stretchr/testify/assert"
)

func TestSomething(t *testing.T) {
	// assert equality
	assert.Equal(t, 123, 123, "they should be equal")

	// assert inequality
	got := 456
	want := 123
	assert.NotEqualf(t, 123, 456, "%d should not be equal to %d", got, want)

	type Object struct {
		Value string
	}
	var objectNil *Object
	// assert for nil (good for errors)
	assert.Nil(t, objectNil)

	objectNotNil := &Object{
		Value: "Something",
	}
	// assert for not nil (good when you expect something)
	if assert.NotNil(t, objectNotNil) {
		// now we know that object isn't nil, we are safe to make
		// further assertions without causing any errors
		assert.Equal(t, "Something", objectNotNil.Value)
	}
}

```

特性说明：

1. `assert` 中的断言函数中，第一个参数是 `*testing.T`, 如果断言失败，`assert` 函数会通过 `*testing.T` 对象生成错误报告。
2. `assert` 中的断言函数和方法，无论断言成功失败都会返回一个 `bool` 值结果，这可以让我们能够进一步进行断言。
3. `assert` 中的断言函数和方法都有配套的 格式化 输出方法（在名字后加上 f），可以方便的自定义错误报告信息。
4. `assert` 即使断言失败也会继续测试，区别于 `require`。

### 断言方法

如果我们的此时函数包含大量断言，每次都将 `*testing.T` 手动传入代码会显得臃肿，`assert` 提供了 `New` 函数，使用 `*testing.T` 创建断言对象，从而避免手动传入 `*testing.T`：

使用 `assert` 对象该在刚才的示例：

```golang
func TestSomething(t *testing.T) {
	assert := assert.New(t)

	// assert equality
	assert.Equal(123, 123, "they should be equal")

	// assert inequality
	got := 456
	want := 123
	assert.NotEqualf(123, 456, "%d should not be equal to %d", got, want)

	type Object struct {
		Value string
	}
	var objectNil *Object
	// assert for nil (good for errors)
	assert.Nil(objectNil)

	objectNotNil := &Object{
		Value: "Something",
	}
	// assert for not nil (good when you expect something)
	if assert.NotNil(objectNotNil) {
		// now we know that object isn't nil, we are safe to make
		// further assertions without causing any errors
		assert.Equal("Something", objectNotNil.Value)
	}
}
```

### 好用的断言

这一节补充说明一些好用的断言函数/方法

- `ElementsMatch`  
  用于断言 数组/切片 中是否包含相同的元素，不考虑顺序。
- `Exactly`
  用于断言结构体类型、值相同
- `EqualExportedValues`  
   用于断言结构体对象类型相同且对外暴露字段相同
- `ErrorAs`
  用于断言 `error` 链中包含制定类型 `error`

## require 包

`require` 包与 `assert` 包是互补关系，它提供完全一模一样的断言方法（API 完全兼容，比如 `require.Equal`、`require.NotNil`），断言的语法、参数、语义都没有区别。  
区别在于 `require` 包断言失败时，会直接终止当前测试函数的执行（内部调用了 `t.FailNow()`），`后续代码不会再运行`。
这个特性在后续断言依赖之前的断言（换句话说就是如果此刻断言失败，继续后续断言没有意义）时使用。

> [!important] 需要注意的是：`require` 断言需要在执行测试的协程中执行。

## mock 包

提供强大的 Mock 能力，可以轻松「模拟实现」Go 中的接口 (interface)，无需编写真实的接口实现代码。

经典示例：

```golang
package main

import (
	"testing"

	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/mock"
)

// 1. 定义业务依赖的接口（比如：数据库操作接口）
type DB interface {
	Get(id int) (string, error)
}

// 2. 用mock包实现这个接口：定义MockDB，嵌入mock.Mock
type MockDB struct {
	mock.Mock
}

// 3. 实现接口的Get方法，核心是调用mock.Called()并返回预设值
func (m *MockDB) Get(id int) (string, error) {
	args := m.Called(id) // 捕获调用入参
	// 返回预设的第0个(string)、第1个(error)值
	// 如果这里返回的不是 mock 包中预设的类型 可以通过 `args.Get().(type)` 方式设置
	return args.String(0), args.Error(1)
}

// 4. 业务逻辑函数：依赖DB接口，我们要测试这个函数
func QueryData(db DB, id int) string {
	data, err := db.Get(id)
	if err != nil {
		return "error"
	}
	return data
}

// 5. 单元测试：用MockDB替代真实DB，解耦测试
func TestQueryData(t *testing.T) {
	// 初始化mock对象
	mockDB := new(MockDB)

	// 预设规则：当调用mockDB.Get(1)时，返回"test_data"和nil
	mockDB.On("Get", 1).Return("test_data", nil)
	// 预设规则：当调用mockDB.Get(2)时，返回""和error
	mockDB.On("Get", 2).Return("", assert.AnError)

	// 测试场景1：调用QueryData(1)，预期返回test_data
	res1 := QueryData(mockDB, 1)
	assert.Equal(t, "test_data", res1)
	// 校验：Get(1)是否被调用了1次
	mockDB.AssertCalled(t, "Get", 1)

	// 测试场景2：调用QueryData(2)，预期返回error
	res2 := QueryData(mockDB, 2)
	assert.Equal(t, "error", res2)
	// 校验：Get(2)是否被调用了1次
	mockDB.AssertCalled(t, "Get", 2)

	// 检验：是否符合 mockDB.On 预期输入输出
	mockDB.AssertExpectations(t)
}
```

<!-- Todo https://vektra.github.io/mockery/latest/ -->
<!-- Todo suite -->
<!-- Todo  -->

## 参考资料

[testify -- github 仓库](https://github.com/stretchr/testify)
