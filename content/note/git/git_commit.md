---
date: "2025-12-23T15:27:07+08:00"
title: "Git -- Angular 提交规范"
tags: ["git", "git_commit"]
categories: "笔记"
description: ""
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

一个好的 `Git Commit Message` 可以让提交记录更易读、易维护。业界比较公认的方式就是采用 `Angular 规范` 。

<!--more-->

## 提交模板

`Angular 规范` 提供了一套 `Commit Message` 的模板：

```plaintext
<type>(<scope>): <subject>

<body>

<footer>
```

- 必填：type + subject（类型 + 简短描述）
- 可选：scope（作用域）、body（详细描述）、footer（备注 / 关联信息）

## 细节说明

### type 类型

`type` 作为说明提交目的的部分，常用类型有以下 9 种：

> 一般情况小使用小写即可

| 类型     | 说明                                     |
| :------- | :--------------------------------------- |
| feat     | 新增功能（feature）                      |
| fix      | 修复 bug（bug fix）                      |
| docs     | 仅修改文档（如 README、注释）            |
| style    | 代码格式调整（无逻辑变更，如缩进、空格） |
| refactor | 代码重构（无新增功能 / 修复 bug）        |
| test     | 添加 / 修改测试代码                      |
| chore    | 构建 / 工具类变更（如依赖、配置文件）    |
| perf     | 性能优化                                 |
| revert   | 回滚之前的提交                           |

### scope 作用域（可选）

说明提交影响的模块 / 文件，得益于清晰明确的代码拆分，大部分情况下不需要使用。
如果遇到需要使用，则需要根据具体项目具体情况，选择合适的作用域拆分方式，比如：

- 前端：auth（认证模块）、button（按钮组件）、api（接口层）
- 后端：user（用户模块）、order（订单模块）、db（数据库层）
- 通用：global（全局）、utils（工具类）、config（配置）

### subject 主题

简短描述本次提交的内容，一般情况下需要遵循以下约定：

- 长度一般少于 50 个字符
- 开头小写
- 结尾不加句号
- 用祈使句（现在时态动词开头）：比如「add」而非「added」「adds」

### body 正文（可选）

说明「为什么改」「改了什么」「影响」，而非重复主题：

### footer 脚注（可选）

备注信息，一般只有以下两种情况下使用:

1. 关联 Issue：Closes #123（关闭 Issue 123）、Related to #456（关联 Issue 456）
2. 不兼容变更：BREAKING CHANGE: 接口参数格式从 xxx 改为 yyy

## 参考示例

- 最简版（日常小改动）
  ```bash
  git commit -m "fix: 修复登录时密码加密错误"
  ```
- 带作用域版
  ```bash
  git commit -m "feat(user): 新增用户手机号验证功能"
  ```
- 完整版（复杂变更）

  ```plaintext
  # git commit
  refactor(api): 重构订单接口参数格式

  1. 将订单参数从 JSON 字符串改为结构体传递，提升解析效率
  2. 统一接口错误码格式，便于前端处理
  3. 移除冗余的参数校验逻辑（已移至中间件）

  Closes #789
  BREAKING CHANGE: 订单接口入参的 `order_info` 字段不再接受字符串格式
  ```

## 实用技巧

### 修正最后一次提交（未推送到远程）

```bash
git commit --amend
```

### 合并多次提交

```bash
# 在编辑器中把需要合并的提交标记为 squash，再统一修改 Commit Message。
git rebase -i HEAD~3  # 编辑最近3次提交
```
