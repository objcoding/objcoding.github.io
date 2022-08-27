---
layout: post
title: "SpringBoot整合RabbitMQ"
categories: SpringBoot RabbitMQ
tags: rabbitmq springboot
author: 张乘辉
---

* content
{:toc}
之前一直使用 SpringCloud Stream 这个组件来整合 RabbitMQ，但发现其非常不好用，配置极其繁琐，对RabbiMQ 过度封装，导致对开发者极其不友好。所以，我在这里介绍一种 SpringBoot Starter 整合的方法。











## 创建项目

- 首先引入 SpringBoot 起步依赖：

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>

<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```



- 编写配置类：

```java
@Configuration
public class RabbitMQConfig {

  public final static String QUEUE_NAME = "spring-boot-queue";
  public final static String EXCHANGE_NAME = "spring-boot-exchange";
  public final static String ROUTING_KEY = "spring-boot-key";

  // 创建队列
  @Bean
  public Queue queue() {
    return new Queue(QUEUE_NAME);
  }

  // 创建一个 topic 类型的交换器
  @Bean
  public TopicExchange exchange() {
    return new TopicExchange(EXCHANGE_NAME);
  }

  // 使用路由键（routingKey）把队列（Queue）绑定到交换器（Exchange）
  @Bean
  public Binding binding(Queue queue, TopicExchange exchange) {
    return BindingBuilder.bind(queue).to(exchange).with(ROUTING_KEY);
  }

  @Bean
  public ConnectionFactory connectionFactory() {
    CachingConnectionFactory connectionFactory = new CachingConnectionFactory("xx", 5670);
    connectionFactory.setUsername("guest");
    connectionFactory.setPassword("guest");
    return connectionFactory;
  }

  @Bean
  public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
    return new RabbitTemplate(connectionFactory);
  }
}
```

这里我们创建 ConnectionFactory 填写 HAProxy 负载均衡地址与端口 。



- 生产者：

```java
@RestController
public class ProducerController {

  @Autowired
  private RabbitTemplate rabbitTemplate;

  @GetMapping("/sendMessage")
  public String sendMessage() {
    new Thread(() -> {
      for (int i = 0; i < 100; i++) {
        LocalDateTime time = LocalDateTime.now();
        System.out.println("send message: " + time.toString());
        rabbitTemplate.convertAndSend(RabbitMQConfig.EXCHANGE_NAME, RabbitMQConfig.ROUTING_KEY, time.toString());
      }
    }).start();
    return "ok";
  }

}
```

直接用 Spring 的 RabbitTemplate 模版，根据交换器和路由键，将消息路由到特定队列。



- 消费者：

```java
@Component
public class Consumer {

    @RabbitListener(queues = RabbitMQConfig.QUEUE_NAME)
    public void consumeMessage(String message) {
        System.out.println("consume message:" + message);
    }
}
```

使用 @RabbitListener 注解，指定需要监听的队列。



- 指定端口

```yaml
server:
  port: 5008
```

在 application.yml 自定义项目端口。



## 运行项目

启动项目，打开浏览器，请求`http://localhost:5008/sendMessage`，得到打印消息：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/rabbit_mq_13.png)



打开 RabbitMQ web 控制台，也可以看到刚才我们在代码里面配置的交换器和队列，以及绑定信息：



![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/rabbit_mq_14.png)



![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/rabbit_mq_15.png)



可以看出，我们这次创建的队列，被创建在集群的节点 2 上了，也验证了 HAProxy 的负载均衡。



**GitHub 项目地址**：[rabbitmq-tutorial](https://github.com/objcoding/rabbitmq-tutorial)