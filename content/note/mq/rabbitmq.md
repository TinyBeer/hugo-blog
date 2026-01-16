---
date: "2026-01-16T14:08:43+08:00"
title: "RabbitMQ"
tags: ["RabbitMQ", "Golang", "MQ"]
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

`RabbitMQ` 是一款基于 `Erlang` 语言开发的、开源的高性能消息中间件（`Message Broker`），遵循行业标准的 `AMQP` 协议（高级消息队列协议），也是目前最主流的消息队列之一。

<!--more-->

> 本文将使用 `Go` 语言客户端进行 `RabbitMQ` 的功能演示，并提供相应的 `Go` 语言代码。

## 安装

这里提供使用 `docker compose` 快速安装 `RabbitMQ` 的配置文件（配置项目已经进行注释说明，如有特殊需求可自行修改）。  
如果需要使用其他方式安装，可参考 [官方文档](https://www.rabbitmq.com/docs/download)：

```yml compose.yml
version: "3.8"
services:
  # rabbitmq 服务名称
  rabbitmq:
    # 指定镜像：management 版本自带web管理界面
    image: rabbitmq:4.2.2-management
    # 容器名称，自定义即可
    container_name: rabbitmq-server
    # 重启策略：容器退出时总是重启（生产推荐）
    restart: always
    # 端口映射【核心】，缺一不可
    ports:
      # RabbitMQ 核心通信端口 (AMQP协议)，程序连接用这个端口
      - "5672:5672"
      # RabbitMQ Web可视化管理后台端口，浏览器访问用这个端口
      - "15672:15672"
    # 环境变量配置【核心】：账号、密码、虚拟主机、时区
    environment:
      # 登录管理后台的账号
      - RABBITMQ_DEFAULT_USER=dev
      # 登录管理后台的密码
      - RABBITMQ_DEFAULT_PASS=dev123
      # RabbitMQ默认虚拟主机，默认就是/，可自定义
      - RABBITMQ_DEFAULT_VHOST=/
      # 时区配置，解决容器内时间和本地不一致问题
      - TZ=Asia/Shanghai
    # 数据持久化【重要】：容器删除后，mq的队列/消息/配置不会丢失
    volumes:
      - ./data:/var/lib/rabbitmq
    # 分配容器资源，按需配置，可选
    deploy:
      resources:
        limits:
          cpus: "1"
          memory: 1G
```

## 生产者/消费者模型

`RabbitMQ` 使用经典的 `生产者-消费者` 模型，核心是通过共享缓冲区解耦数据生成与处理流程，实现异步协作、缓冲削峰和负载均衡，广泛用于多线程、分布式消息队列及各类异步系统。其中包含三个角色/组件：

1. 生产者（Producer）  
   生成数据、任务或事件，写入共享缓冲区，无需等待消费者处理

   ```mermaid
   graph LR
        P((P))
   ```

2. 消费者（Consumer）  
   从缓冲区读取数据并处理，不直接与生产者交互
   ```mermaid
   graph LR
     C((C))
   ```
3. 缓冲区（Buffer/Queue）  
   暂存数据的中间容器，平衡生产与消费速率，提供同步机制
   ```mermaid
   graph LR
     B[[Buffer]]
   ```

这三个角色可以不在同一主机上，事实上大部分情况下他们都处在不同的主机上。

`RabbitMQ` 的工作模式如下图所示：

```mermaid
graph LR
    P1((P1))
    P2((P2))
    P3((P...))

    C1((C1))
    C2((C2))
    C3((C...))

    B[[RabbitMQ]]


    P1 & P2 & P3 -->|消息|B -->|消息|C1 & C2 & C3
```

## 快速使用

参考官网提供的示例，这里给出一个基础的使用示例，业务模型如下：

> 这里假设本机已经启动 `RabbitMQ` 服务  
> 服务端口：5672  
> 账号：dev  
> 密码：dev123

```mermaid
graph LR
    P((P))
    C((C))
    B[[MQ]]
    Action[打印]
    P-->|"hello word"|B-->|"hello word"|C ==>Action
```

### 工程准备

```bash
# 创建工程框架
mkdir -p rabbitmq-quickstart/{sender,receiver}
cd rabbitmq-quickstart
# 初始化go mod 这里的模块名称可以自行定义
go mod init rabbitmq-quickstart
# 安装依赖库文件
go get github.com/rabbitmq/amqp091-go
# 创建 main.go 文件
touch receiver/main.go sender/main.go
```

最终工程结构如下：

```bash
.
├── go.mod
├── go.sum
├── receiver
│   └── main.go
└── sender
    └── main.go
```

### 消费者代码

我们在 `receiver/main.go` 文件中向 `RabbitMQ` 注册一个消费者，来监听并消费缓冲区的消息，代码如下：

```golang receiver/main.go
package main

import (
	"log"

	amqp "github.com/rabbitmq/amqp091-go"
)

func main() {
	// 使用amqp协议连接本地RabbitMQ
	conn, err := amqp.Dial("amqp://dev:dev123@localhost:5672/")
	if err != nil {
		log.Fatalf("Failed to connect to RabbitMQ, err: %v", err)
	}
	defer conn.Close()

	// 创建管道
	ch, err := conn.Channel()
	if err != nil {
		log.Fatalf("Failed to open a channel, err: %v", err)
	}
	defer ch.Close()

	// 声明消息队列 sender声明时需要使用相同的name
	q, err := ch.QueueDeclare(
		"hello", // name
		false,   // durable
		false,   // delete when unused
		false,   // exclusive
		false,   // no-wait
		nil,     // arguments
	)
	if err != nil {
		log.Fatalf("Failed to declare a queue, err: %v", err)
	}

	// 消费消息
	msgs, err := ch.Consume(
		q.Name, // queue
		"",     // consumer
		true,   // auto-ack
		false,  // exclusive
		false,  // no-local
		false,  // no-wait
		nil,    // args
	)
	if err != nil {
		log.Fatalf("Failed to register a consumer, err: %v", err)
	}

	forever := make(chan struct{})

	go func() {
		// 打印消息
		for d := range msgs {
			log.Printf("Received a message: %s", d.Body)
		}
	}()

	log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
	// 等待
	_ = <-forever
}
```

### 生产者代码

我们在 `sender/main.go` 文件中编写生产者代码，用以向缓冲区投递消息（生产消息），代码如下：

```golang sender/main.go
package main

import (
	"context"
	"log"
	"time"

	amqp "github.com/rabbitmq/amqp091-go"
)

func main() {
	// 使用 amqp 协议 连接本地 RabbitMQ
	conn, err := amqp.Dial("amqp://dev:dev123@localhost:5672/")
	if err != nil {
		log.Fatalf("Failed to connect to RabbitMQ, err: %v", err)
	}
	defer conn.Close()

	// 创建管道
	ch, err := conn.Channel()
	if err != nil {
		log.Fatalf("Failed to open a channel, err: %v", err)
	}
	defer ch.Close()
	// 声明消息队列  注意保持 name 和 receiver 中一致
	q, err := ch.QueueDeclare(
		"hello", // name
		false,   // durable
		false,   // delete when unused
		false,   // exclusive
		false,   // no-wait
		nil,     // arguments
	)
	if err != nil {
		log.Fatalf("Failed to declare a queue, err: %v", err)
	}

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	// 发布消息
	body := "Hello World!"
	err = ch.PublishWithContext(ctx,
		"",     // exchange
		q.Name, // routing key
		false,  // mandatory
		false,  // immediate
		amqp.Publishing{
			ContentType: "text/plain",
			Body:        []byte(body),
		})
	if err != nil {
		log.Fatalf("Failed to publish a message, err: %v", err)
	}
	log.Printf(" [x] Sent %s\n", body)
}
```

### 测试

测试的时候我们需要用到两个终端，这里使用一个终端启动消费者（`receiver`），开启监听（当然，先启动生产者也是可以的）：

```bash
go run ./receiver
2026/01/16 15:08:05  [*] Waiting for messages. To exit press CTRL+C
```

然后我们运行生产者（`sender`）,生产消息：

```bash
go run ./sender
2026/01/16 15:09:22  [x] Sent Hello World!
```

此时再回到消费者所在终端可以看到消息已经被打印：

```bash
go run ./receiver
2026/01/16 15:08:05  [*] Waiting for messages. To exit press CTRL+C
2026/01/16 15:09:22 Received a message: Hello World!
```

## 消息分发策略

### Round-robin

`RabbitMQ` 的默认消息策略为轮询（`Round-robin`）,即无论消费者是否完成消息处理，都按照注册顺序逐一均匀地分发消息。

测试轮询分发策略我们需要简单修改一下之前的代码：

我们在 `receiver/main.go` 中消息打印部分添加一些延时，用来模拟处理消息耗时，消息中每一个点（`.`） 表示执行一秒：

```golang
// 打印消息
for d := range msgs {
    log.Printf("Received a message: %s", d.Body)
    dotCount := bytes.Count(d.Body, []byte("."))
    time.Sleep(time.Second * time.Duration(dotCount))
    log.Printf("Message: %s done", d.Body)
}
```

我们让生产者 `sender/main.go` 生产多条消息：

```golang
// 发布消息
for _, body := range []string{
    "First message.",
    "Second message..",
    "Third message...",
    "Fourth message....",
    "Fifth message.....",
} {
    err = ch.PublishWithContext(ctx,
        "",     // exchange
        q.Name, // routing key
        false,  // mandatory
        false,  // immediate
        amqp.Publishing{
            ContentType: "text/plain",
            Body:        []byte(body),
        })
    if err != nil {
        log.Fatalf("Failed to publish a message, err: %v", err)
    }
    log.Printf(" [x] Sent %s\n", body)
}
```

我们先开启两个终端，启动两个消费者，然后启动生产者，可以得到如下日志输出：

```bash
# receiver1
go run ./receiver
2026/01/16 16:23:08  [*] Waiting for messages. To exit press CTRL+C
2026/01/16 16:23:25 Received a message: First message.
2026/01/16 16:23:26 Message: First message. done
2026/01/16 16:23:26 Received a message: Third message...
2026/01/16 16:23:29 Message: Third message... done
2026/01/16 16:23:29 Received a message: Fifth message.....
2026/01/16 16:23:34 Message: Fifth message..... done


# receiver2
go run ./receiver
2026/01/16 16:23:16  [*] Waiting for messages. To exit press CTRL+C
2026/01/16 16:23:25 Received a message: Second message..
2026/01/16 16:23:27 Message: Second message.. done
2026/01/16 16:23:27 Received a message: Fourth message....
2026/01/16 16:23:31 Message: Fourth message.... done


# sender
go run ./sender
2026/01/16 16:23:25  [x] Sent First message.
2026/01/16 16:23:25  [x] Sent Second message..
2026/01/16 16:23:25  [x] Sent Third message...
2026/01/16 16:23:25  [x] Sent Fourth message....
2026/01/16 16:23:25  [x] Sent Fifth message.....
```

### prefetch

使用 `Round-robin` 进行消息分发，很可能发生负载倾斜（大量工作负载被分发到少数消费者上）如：两个消费者时，偶数消息都为高负载任务。这是我们可以使用 `prefetch` 进行负载平衡。



## 消息确认机制

`RabbitMQ` 提供了消息确认机制（`Message acknowledgment`）,即消费者在消息处理完成后告知 `RabbitMQ` 消息已经处理完成的的机制。

在之前的代码中，我们开启了自动消息确认：

```golang
...
// 消费消息
msgs, err := ch.Consume(
    q.Name, // queue
    "",     // consumer
    // 自动消息确认
    true,   // auto-ack
    false,  // exclusive
    false,  // no-local
    false,  // no-wait
    nil,    // args
)
...
```

在这种配置下，当消费者 `receiver` 接收到消息后，就会立即告知 `RabbitMQ` 消息已经处理完成，此时 `RabbitMQ` 会删除对应的消息。如果在后续的处理过程中发生了异常，这条消息的处理将彻底失败。这显然不是我们期望的，这是我们就需要在消费者代码 `receiver/main.go` 中使用手动消息确认了：

首先我们应取消自动消息确认：

```golang
...
// 消费消息
msgs, err := ch.Consume(
    q.Name, // queue
    "",     // consumer
    // 取消自动消息确认
    false,   // auto-ack
    false,  // exclusive
    false,  // no-local
    false,  // no-wait
    nil,    // args
)
...
```

此后我们需要在消息完成后进行消息确认：

```golang
...
go func() {
    // 打印消息
    for d := range msgs {
        log.Printf("Received a message: %s", d.Body)
        dotCount := bytes.Count(d.Body, []byte("."))
        time.Sleep(time.Second * time.Duration(dotCount))
        log.Printf("Message: %s done", d.Body)

        // 手动进行消息确认
        d.Ack(false)
    }
}()
...
```

开启手动消息确认后，如果客户端 通道（channel）关闭，或 连接（conn）断开，`RabbitMQ` 会感知到消息可能没有被正确处理，并立即开始对消息的重新分发。

> [!notice] `RabbitMQ` 的消息确认超时时间为 30 分钟，如果超过这个时间任然没有确认消息，`RabbitMQ` 将重新对消息进行分发。

### 注意事项

- 一定要在接受消息的同一通道 `channel` 进行消息确认，否则会引起通道级别的异常。
- 及时的对消息进行确认。未被确认的消息将持续滞留在 `RabbitMQ` 中，会造成消息堆积，加大内存占用。  
  `RabbitMQ` 提供了查看未确认消息的方法，我们需要进入 `RabbitMQ` 所在主机执行一下命令：

  ```bash
  sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged
  # windows 中使用下方命令
  rabbitmqctl.bat list_queues name messages_ready messages_unacknowledged
  ```

## 消息持久化

通过消息确认机制，我们可以有效的决绝消费者异常产生的消息丢失。但如果 `RabbitMQ` 关闭或者发生异常，`RabbitMQ` 内存中的消息将无法恢复。此时我们就需要用到 `RabbitMQ` 的消息持久化机制。

在之前的示例中，我们未开启消息持久化功能，让消息仅存储存储在内存中：

```golang
...
q, err := ch.QueueDeclare(
    "hello", // name
    // 持久化
    false,   // durable
    false,   // delete when unused
    false,   // exclusive
    false,   // no-wait
    nil,     // arguments
)
...
err = ch.PublishWithContext(ctx,
    "",     // exchange
    q.Name, // routing key
    false,  // mandatory
    false,  // immediate
    amqp.Publishing{
        // 使用了默认分发
        // DeliveryMode: amqp.Persistent,
        ContentType: "text/plain",
        Body:        []byte(body),
    })
...
```

要开启消息的持久化，我们需要进行如下修改：

> 队列（`Queue`） 和 消息（`messages`） 的持久化需要同时开启才能实现消息的持久化。  
> 如果我们仅开启队列的持久化，可能会发生报错，因为 `RabbitMQ` 不允许声明相同名称而配置的队列，这里通过修改名称解决。  
> 即使开启了消息持久化，在消息进入 `RabbitMQ` 到完成磁盘写入这段时间任然可能发生消息丢失，此时我们可能需要使用 发布者确认 机制。

```golang
...
q, err := ch.QueueDeclare(
    "hello2", // name
    // 持久化
    true,   // durable
    false,   // delete when unused
    false,   // exclusive
    false,   // no-wait
    nil,     // arguments
)
...
err = ch.PublishWithContext(ctx,
    "",     // exchange
    q.Name, // routing key
    false,  // mandatory
    false,  // immediate
    amqp.Publishing{
        // 使用了默认分发
        DeliveryMode: amqp.Persistent,
        ContentType: "text/plain",
        Body:        []byte(body),
    })
...
```

## 参考资料

[RabbitMQ 官方教程](https://www.rabbitmq.com/tutorials)
