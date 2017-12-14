---
layout: post
title: "利用Redis中的List实现消息队列"
categories: Redis
tags: Redis 消息队列 List
author: zch
---

* content
{:toc}
目前项目的消息事件异步处理采用了Android事件总线的EventBus来实现，但它只能单机上执行，不能在其他子模块上执行该消息任务，且如果重启项目过程中，会导致消息事件处理不到。随着业务的发展，EventBus已经不那么适用了，毕竟它是专门用于Android的。因此现在项目改成用Redis的list实现消息队列。







## 工具类

```java
public class EventHandlerRedisUtil {
  public static void push(EventMessage message, String key) {
    ListRedisTemplate listRedisTemplate = SpringUtil.getBean(ListRedisTemplate.class);
    SetRedisTemplate setRedisTemplate = SpringUtil.getBean(SetRedisTemplate.class);
    String jsonStr = JSONArray.toJSONString(message);
    listRedisTemplate.lpush(key, jsonStr);
    setRedisTemplate.sadd(key, jsonStr);
  }
}
```

该工具类用于将消息事件push进Redis的List中，同时再将该消息事件保存在Set数据结构中，防止消息事件取出来的过程中突然重启导致消息事件丢失，由于Redis已做持久化处理，所以项目再次重会重新加载Set的消息到List中。

```java
/**
 * 消息事件处理异步工具类
 * Created by zch on 2017/9/6.
 */
public class EventHandlerAsyncUtil {
  private static ExecutorService executorService = null;

  public static void submit(Runnable runnable) {
    if(null == executorService) {
      executorService = Executors.newFixedThreadPool(300, new CustomizableThreadFactory("event-handler-async-pool-"));
    }
    executorService.submit(runnable);
  }
}
```

该工具类用于异步执行消息事件。





## 添加消息事件到List中

```java
/**
 * 添加话题
 *
 * @param topic
 * @return
 */
@Override
public TopicView createTopic(Topic topic) {
  TopicView topicView = null;
  topic.setStatus(CommonConstants.ACTIVED);
  topic.setPreviewType(CommonConstants.PreviewType.THREE);
  topic.setCreateTime(System.currentTimeMillis());
  Topic t = create(topic);
  if (t != null) {

    // TODO 为话题添加管理员

    delTopicListCache(topic.getCreateBy());
    // 推送模板消息
    DataAsyncUtil.submit(() -> {
      pushTopicMessage(topic);
    });
    topicView = TopicViewConverter.convertList(t);
  }

  //新建后处理事件 TopicEventHandler
  TopicEventMessage topicEventMessage = new TopicEventMessage();
  topicEventMessage.setTopic(topic);
  topicEventMessage.setEvent(TopicEventMessage.Event.CREATE);
  topicEventMessage.setUserId(topic.getCreateBy());
  EventHandlerRedisUtil.push(topicEventMessage, RedisConstants.Event.TOPIC_EVENT);

  return topicView;
}
```





## 消息事件处理类

```java
/**
 * 话题事件处理
 * Created by zch on 2017-9-8.
 */
@Component
public class TopicEventHandler {

  private final Logger logger = LoggerFactory.getLogger(TopicEventHandler.class);

  private final TopicExtendService topicExtendService;
  private final ChatroomRemoteService chatroomRemoteService;

  public TopicEventHandler(TopicExtendService topicExtendService, ChatroomRemoteService chatroomRemoteService) {
    this.topicExtendService = topicExtendService;
    this.chatroomRemoteService = chatroomRemoteService;
  }

  public void handle(TopicEventMessage message) {

    logger.info("执行话题操作事件 event：{}， message：{}", message.getEvent(), JSON.toJSONString(message));

    switch (message.getEvent()) {
      case TopicEventMessage.Event.CREATE: {
        topicExtendService.add(message.getTopic());

        //创建聊天室
        chatroomRemoteService.add(CommonConstants.ActivityType.TOPIC + message.getTopic().getId(), message.getTopic().getName());
        break;
      }
      case TopicEventMessage.Event.DELETE:

        //销毁聊天室
        chatroomRemoteService.delete(CommonConstants.ActivityType.TOPIC + message.getTopic().getId());
        break;
    }
  }
}
```





## 处理消息事件任务

```java
/**
 * 消息事件处理任务
 * Created by zch on 2017/9/6.
 */
@Service
public class TopicEventHandlerTask {

  private final Logger msgLogger = LoggerFactory.getLogger("MESSAGE_LOG");
  private final Logger logger = LoggerFactory.getLogger(TopicEventHandlerTask.class);

  @Resource
  private TopicEventHandler topicEventHandler;
  @Resource
  private ListRedisTemplate listRedisTemplate;
  @Resource
  private SetRedisTemplate setRedisTemplate;

  public void openTask() {
    while (true) {
      String topicMessage = listRedisTemplate.rpop(RedisConstants.Event.TOPIC_EVENT);
      topicEventHandler(topicMessage);
    }
  }

  private void topicEventHandler(String message) {
    if (StringUtils.isBlank(message)) {
      try {
        Thread.sleep(1000);
      } catch (InterruptedException e) {
        msgLogger.error("TopicEventHandlerTask topicEventHandler >>>> Thread.sleep error", e);
      }
    } else {
      EventHandlerAsyncUtil.submit(() -> {
        TopicEventMessage topicEventMessage;
        try {
          topicEventMessage = JSONArray.parseObject(message, TopicEventMessage.class);
        } catch (Exception e) {
          return;
        }
        topicEventHandler.handle(topicEventMessage);
        // 处理完成后同时删除set的缓存
        setRedisTemplate.srem(RedisConstants.Event.TOPIC_EVENT, message);
      });
    }
  }
}
```

处理消息事件任务采用死循环方式不断的读取List中的消息事件，然后通过异步的方式处理消息事件，处理完后同时把Set中的消息事件删除。



## 初始化事件任务列表方法

```java
/**
 * 初始化事件任务列表
 */
private static void initHandlerTask() {

  ListRedisTemplate listRedisTemplate = SpringUtil.getBean(ListRedisTemplate.class);
  listRedisTemplate.del(RedisConstants.Event.TOPIC_EVENT);

  // 删除list key，加载set到list
  CompletableFuture.runAsync(() -> {
    Logger logger = LoggerFactory.getLogger(TopicServer.class);
    try {
      TopicEventHandlerTask topicEventHandlerTask = SpringUtil.getBean(TopicEventHandlerTask.class);
      SetRedisTemplate setRedisTemplate = SpringUtil.getBean(SetRedisTemplate.class);

      if (setRedisTemplate.exists(RedisConstants.Event.TOPIC_EVENT)) {
        Set<String> topicEventSet = setRedisTemplate.member(RedisConstants.Event.TOPIC_EVENT);
        if (null != topicEventSet && topicEventSet.size() > 0) {
          for (String message : topicEventSet) {
            listRedisTemplate.lpush(RedisConstants.Event.TOPIC_EVENT, message);
          }
        }
      }
      topicEventHandlerTask.openTask();
    } catch (Exception e) {
      logger.error("重新加载事件队列失败");
    }
  });

}
```

项目启动时删除List中数据，并将Set中数据加载到List中，这里也采用异步执行，以防止拖慢项目启动速度。

