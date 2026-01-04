---
date: "2026-01-04T19:57:51+08:00"
title: "Git -- 基本操作"
tags: ["git"]
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

`Git` 是一款分布式版本控制系统（DVCS），核心用途是追踪文件（代码为主）的修改历史、协同多人开发、管理项目版本，是目前软件开发领域最主流的版本管理工具。

<!--more-->

## 核心概念

1. 工作区（Workspace）：本地电脑上的项目文件夹，日常编写代码、修改文件的区域。
2. 暂存区（Staging Area/Index）：临时存放工作区修改的「缓冲区」，用于标记哪些修改需要提交到版本库，相当于「提交前的确认清单」。
3. 本地仓库（Repository）：`Git` 本地存储版本历史的核心区域（隐藏文件夹 `.git`），包含所有提交记录、分支、标签等信息，暂存区的修改提交后会进入本地仓库。
4. 远程仓库（Remote Repository）：用于多人共享的远程版本库（如 GitHub、Gitee、GitLab 上的仓库），本地仓库可通过 `git push`（推送本地修改）、`git pull`（拉取远程修改）实现协同。
5. 分支（Branch）：默认主分支为 main（早期为 master），其他分支有开发者根据需求创建，用于独立开发任务，开发完成后可合并到主分支。
6. 提交（Commit）：将暂存区的修改保存到本地版本库的操作，每个提交对应一个唯一哈希值，包含修改内容、提交人、提交时间、提交说明等信息，是版本回溯的依据。

## 常用命令

### 初始化/克隆仓库

- `git init`
  将当前文件夹初始化为一个空的 `git` 仓库，生成 `.git` 文件夹，用于存放项目的配置以及快照等。
- `git clone <远程仓库地址> [<directory>]`
  克隆远程仓库到本地，可以通过 `directory` 指定克隆到什么地方，如果没有 `directory` 参数，则将远程参考克隆到当前文件夹。

### 工作区/暂存区操作

- `git add <文件名/文件夹>`
  将工作区 指定文件 或 指定文件夹下所有文件 的变化添加到暂存区。
- `git add .`
  将工作区所有修改（新增、修改、删除）添加到暂存区。
- `git status`
  看工作区和暂存区的状态（哪些文件被修改、是否已添加到暂存区）。

### 提交暂存区内容

- `git commit`
  通过交互模式提交暂存区内容。命令执行后会打开文本编辑器，确认完成输入后(保存并退出)暂存区的内容会被添加到本地仓库，编辑器中的内容会作为提交说明。
- `git commit -m "提交说明"`
  通过 `-m` 添加提交说明，使用对 `git commit` 命令的简化

### 查看提交记录

- `git log`
  通过交互模式查看所有本地提交记录（按时间倒序显示）。

### 同步仓库

- `git pull`
  拉取远程仓库的最新修改并合并到本地当前分支（等同于 `git fetch` + `git merge`）。
- `git push`
  将本地仓库的提交推送到远程仓库。

### 分支操作

- `git branch`
  查看本地所有分支（当前分支前标 \*）。
- `git branch <分支名>`
  基于当前分支创建新分支
- `git checkout <分支名> / git switch <分支名>`
  将当前分支切换到指定分支
- `git merge <分支名>`
  将指定分支的修改合并到当前分支。
- `git checkout <分支名> / git switch -c <分支名>`
  创建并切换到新的分支

## 学习推荐

这里推荐一个非常优秀的学习工具 [Learn Git Branching](https://learngitbranching.js.org/)   
它是一款免费、交互式的 `Git` 可视化学习工具，专门用于帮助开发者快速掌握 `Git` 的核心用法，尤其是对 `Git` 分支操作（创建、切换、合并、变基等）的理解，是入门 `Git` 分支的首选工具之一。

## 参考资料

[Git 官方文档](https://git-scm.com/docs)  
[Learn Git Branching](https://learngitbranching.js.org/)  
[阮一峰 Git 教程](https://www.bookstack.cn/books/git-tutorial)
