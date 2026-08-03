---
date: "2026-08-03T14:05:33+08:00"
title: "Make -- 工程自动化脚本工具"
tags: ["Make", "Makefile"]
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

`make` 最初是设计用于大型 c 语言项目编译的脚本自动化工具，但在后续的发展中逐渐在代码工程自动化中得到广泛应用（不再局限需于 c 语言、项目构建）。<!--more-->

解决痛点：

1. 手动输入大量命令、参数
2. 大型项目构建中间文件复用复杂度高（人工判断哪些可以复用、哪些需要重新构建等）

## 安装

- linux

  ```bash
    # 一键安装 make + gcc/g++ 编译套件
    sudo apt install make gcc g++

    # CentOS7
    sudo yum install make gcc gcc-c++
    # CentOS8+/Rocky 用 dnf
    sudo dnf install make gcc gcc-c++

    # Arch
    sudo pacman -S make gcc
  ```

- windows
  Windows 中安装 `make` 通常使用一下方式
  1. MinGW-w64 中的 `mingw32-make.exe`
     `https://github.com/niXman/mingw-builds-binaries/releases` 中有编译好的压缩包，解压出来后重命名为 `make` 即可。  
     国内镜像分流：`https://mirror.ghproxy.com/https://github.com/niXman/mingw-builds-binaries/releases`
  2. 使用 `git bash` 中自带的 `make`
  3. 使用 `linux` 虚拟机，如：wsl、virtual box、vmware 等

## 核心逻辑

`make` 读取当前目录 `Makefile`，分析文件依赖关系，对比目标文件时间戳：

1. 依赖文件（源码）比目标产物新 → 执行编译命令；
2. 目标产物更新 → 跳过编译。

## 基础规则语法

`make` 通过规则描述文件依赖关系：

```makefile
# 标准规则模板
目标: 依赖1 依赖2 ...
	命令1
	命令2
```

1. 目标 (target)：要生成的文件（可执行程序、.o 中间文件、伪目标 clean/install）
2. 依赖 (prerequisites)：生成目标需要的源码 / 头文件
3. 命令：shell 指令，必须以 Tab 开头（空格会报错）

## 示例

```makefile
# 最终程序：main，依赖 main.c
main: main.c
	gcc main.c -o main

# 清理产物，伪目标
clean:
	rm -f main
```

使用方法：

```bash
make       # 编译生成 main
make clean # 删除可执行文件
```

## 变量

使用示例

```bash
# 1. 变量定义（简化重复代码）
# := 立即展开，= 用到时才展开
CC = gcc                # 编译器
TARGET = app            # 最终输出程序名
OBJS = main.o func.o    # 所有中间目标文件

# 2. 变量使用
# 终极目标：链接所有 .o 生成可执行程序
$(TARGET): $(OBJS)
	$(CC) $(OBJS) -o $(TARGET)
```

### 内置变量

| 变量 | 含义           |
| :--- | :------------- |
| $@   | 当前规则的目标 |
| $<   | 第一个依赖文件 |
| $^   | 全部依赖文件   |

## 补充

- 默认执行：

  `make` 没有传入目标时，会自动执行第一条命令

- `.PHONY`：声明伪目标，防止目录里存在和命令同名的文件导致 make 命令失效
  ```makefile
    .PHONY test build install clean
  ```
- 模式匹配规则:
  ```makefile
  # 批量编译所有 c 文件，不用逐个写规则
  %.o : %.c
  ...
  ```
- `@`: 控制是否打印命令本身

  ```makefile
    no_output:
    @echo '不打印命令本身'
    echo '打印命令本身'
  ```

- 执行多条命令：

  `make` 针对每条命令，都会创建一个独立的 `Shell` 环境，对于需要使用命令修改 `Shell` 环境后执行的命令可以使用一下方法执行：

  ```makefile
  cd_ok1:
  pwd; \
  cd ..; \
  pwd

  # 或者
  cd_ok2:
    cd .. && pwd
  ```

## 综合使用示例

```makefile
# ====================== 项目目录配置区（按需修改） ======================
SRC_DIR := src
INC_DIR := include
OBJ_DIR := obj
BIN_DIR := bin
LIB_DIR := lib
TARGET := $(BIN_DIR)/app

# 编译器与编译、链接参数
CC := gcc
# -Wall 全部警告 -g 调试信息 -I 指定头文件目录
CFLAGS := -Wall -g -I$(INC_DIR)
# -L 指定库目录 -lpthread 链接线程库，不需要可删除
LDFLAGS := -L$(LIB_DIR) -lpthread

# ====================== 自动文件扫描 ======================
# 匹配 src 下全部 .c 源码
SRCS := $(wildcard $(SRC_DIR)/*.c)
# src/xxx.c → obj/xxx.o
OBJS := $(patsubst $(SRC_DIR)/%.c,$(OBJ_DIR)/%.o,$(SRCS))
# 依赖文件：obj/xxx.d，记录头文件依赖关系
DEPS := $(OBJS:.o=.d)

# ====================== 初始化：自动创建输出目录 ======================
$(shell mkdir -p $(OBJ_DIR) $(BIN_DIR))

# ====================== 主编译规则 ======================
# 默认目标，直接 make 执行
all: $(TARGET)

# 链接所有 .o 生成最终可执行文件
# $@ = 当前目标 $(BIN_DIR)/app
# $^ = 所有依赖文件（全部obj/*.o）
$(TARGET): $(OBJS)
	$(CC) $(OBJS) -o $@ $(LDFLAGS)

# ====================== 单文件编译规则 + 自动生成头文件依赖 ======================
# 模板规则：任意 obj/xxx.o 由 src/xxx.c 编译生成
# -MMD 自动生成 .d 依赖文件；-c 仅编译不链接；$< 代表源文件src/xxx.c
$(OBJ_DIR)/%.o: $(SRC_DIR)/%.c
	$(CC) $(CFLAGS) -MMD -c $< -o $@

# 引入所有依赖文件，修改.h会自动重编译对应源码
# 前置减号：首次编译无.d文件时不报错
-include $(DEPS)

# ====================== 清理规则 ======================
clean:
	rm -rf $(OBJ_DIR) $(BIN_DIR)

# 伪目标声明：all/clean 不是文件，避免和同名文件冲突
.PHONY: all clean
```

## 参考资料

[廖雪峰 Makefile 教程](https://liaoxuefeng.com/books/makefile/introduction/index.html)  
[GNU Make 官方手册](https://www.gnu.org/software/make/manual/make.html)
