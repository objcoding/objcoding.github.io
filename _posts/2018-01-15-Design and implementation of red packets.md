---
layout: post
title: "「红包」的设计与实现"
categories: Redis
tags: 分布式缓存锁 并发
author: zch
---

* content
{:toc}


最近在做一个红包小程序的后台，总结一下发红包和抢红包的大致实现过程。









## 发红包

红包分发大概分两种，一种是将红包金额和数量存到缓存中，抢红包过程再通过红包算法算出抢到的金额，这种方式的优点就是可以节省缓存容量，缺点就是在抢红包时需要多一步计算抢到红包的大小。

第二种是发红包的时候就通过算法将红包存到队列和数据库中，抢红包时再将已算好的红包从队列中取出来，这种方式缺点就是增大了缓存的消耗，毕竟用户发的每个红包都要缓存到队列中，但是好处就是在抢过程中直接从队列中拿出来就好了，无需计算。

我采用的是第二种方式。

附上一个红包算法：

```java
/**
  * @param number 红包数量
  * @param total 红包总金额
  * @param min 红包最小金额
  * @return 红包金额分布集合
  */
public List<Integer> randomMoney(int number, int total, int min) {
  int money;
  int max;
  int i = 1;
  List<Integer> math = new ArrayList<>();
  DecimalFormat df = new DecimalFormat("###.##");
  while (i < number) {
    //保证即使一个红包是最大的了,后面剩下的红包,每个红包也不会小于最小值
    max = total - min * (number - i);
    int k = (number - i) / 2;
    //保证最后两个人拿的红包不超出剩余红包
    if (number - i <= 2) {
      k = number - i;
    }
    //最大的红包限定的平均线上下
    max = max / k;
    //保证每个红包大于最小值,又不会大于最大值
    money = (int) (min + Math.random() * (max - min + 1));
    //保留两位小数
    total = total - money;
    math.add(money);
    i++;
    //最后一个人拿走剩下的红包
    if (i == number) {
      math.add(total);
    }
  }
  return math;
}
```



## 抢红包

### 同步锁的选择

抢红包过程涉及到并发，所以需要用到同步锁，对锁的要求有如下几点：

1. 避免死锁，避免单点故障造成死锁，影响其他客户端获取锁。也就是可以设置一个时间，时间到了无论锁是否有释放，都强行进行解锁；
2. 分布式锁，也就是如果一个微服务获取了锁，对另一个微服务具有相同名字的锁同样有效；
3. 具有阻塞效果，即如果锁被另外一个用户获取了，该用户会等待锁的释放而不是立即返回。

因此，我选择了 Redis 的一个开源项目 Redisson 作为此次抢红包的同步锁：

- Redisson配置：

```yaml
---
singleServerConfig:
  idleConnectionTimeout: 10000 #连接空闲超时，单位：毫秒
  pingTimeout: 1000
  connectTimeout: 10000 #连接超时，单位：毫秒
  timeout: 10000 #命令等待超时，单位：毫秒
  retryAttempts: 3 #命令失败重试次数
  retryInterval: 1500 #命令重试发送时间间隔，单位：毫秒
  reconnectionTimeout: 3000 #重新连接时间间隔，单位：毫秒
  failedAttempts: 3 #执行失败最大次数
  password: null
  subscriptionsPerConnection: 5 #单个连接最大订阅数量
  clientName: null
  address: "redis://127.0.0.1:6379"
  subscriptionConnectionMinimumIdleSize: 1 #发布和订阅连接的最小空闲连接数
  subscriptionConnectionPoolSize: 50 #发布和订阅连接池大小
  connectionMinimumIdleSize: 10 #最小空闲连接数
  connectionPoolSize: 64 #连接池大小
  database: 0
  dnsMonitoring: false #是否启用DNS监测
  dnsMonitoringInterval: 5000 #DNS监测时间间隔，单位：毫秒
threads: 0
nettyThreads: 0
codec: !<org.redisson.codec.JsonJacksonCodec> {}
useLinuxNativeEpoll: false
```



- 创建 Bean：

```java
@Bean(destroyMethod = "shutdown")
public RedissonClient redissonClient() throws IOException {
  return Redisson.create(
    Config.fromYAML(new ClassPathResource("redisson-" + ProfileUtil.getProfile() + ".yml").getInputStream())
  );
}
```





### 抢红包过程拆分

有同步，那必然要考虑多用户同时请求造成的阻塞时间，基于这个目的，我将抢红包过程拆分成「抢过程」和「拆过程」，「抢过程」属于快的过程，「拆过程」属于慢的过程，因为「拆过程」涉及到数据库的事务操作。

因此我将「抢过程」放在 cache 层，也就是利用红包队列个长度来判断红包是否抢完，每进来一个用户，就从队列取出一个红包，当队列长度为 0 时，那么意味着红包已经抢完了，这时可以抵挡大部分用户的请求了，防止过多用户停留在「拆过程」。已经抢到红包的用户进入到「拆过程」，每次只能有一个用户进行拆操作，其余抢到红包的用户会在锁外面等待。

代码大致实现：

```java
@Override
@Transactional
public Map<String, Object> receive(Long luckyMoneyId, Long userId, String form_id, String appId) throws RestException {

  /**
    * 第一层过滤筛选，抵挡了大部分请求，属于抢过程
    */
  if (isReceived(luckyMoneyId, userId)) {
    throw new RestException("您已领取过红包");
  }
  String listKey = RedisCons.UNRECEIVED_LIST
    .replace("{LUCKYMONEYID}", String.valueOf(luckyMoneyId));
  String messageJson = listRedisTemplate.rpop(listKey);
  if (StringUtils.isBlank(messageJson)) {
    throw new RestException("红包被抢光啦");
  }


  /**
    * 第二层上锁，并进行领红包事务操作，属于拆过程
    */
  String lockKey = CommonCons.LockName.RECEIVE_LUCKY_MONEY
    .replace("{LUCKYMONEYID}", String.valueOf(luckyMoneyId));
  RLock lock = redissonClient.getLock(lockKey);

  try {
    if (lock.tryLock(30L, TimeUnit.SECONDS)) {
      // 2.事务操作
      try {
        // 2.1.更新领取的红包信息

        // 2.2.增加用户余额

        // 2.3.更新红包已领取数量

        // 2.4.缓存已领取红包用户

        // 2.5.form_id发送模板消息

        // 2.6. redis-mq 处理模版消息推送、流水记录、用户积分等操作

      } catch (Exception e) {
        // 重载红包到队列中
        listRedisTemplate.rpush(listKey, messageJson);
        throw new RestException("领取失败");
      } finally {
        lock.unlock();
      }

    } else {
      // 重载红包到队列中
      listRedisTemplate.rpush(listKey, messageJson);
      logger.error("锁超时了: >>>>>> {}, {}", luckyMoneyId, userId);
      throw new RestException("服务器繁忙，请稍后再试");
    }

  } catch (InterruptedException ie) {
    // 重载红包到队列中
    listRedisTemplate.rpush(listKey, messageJson);
    logger.error("锁有异常：{}", ie);
    throw new RestException("服务器繁忙，请稍后再试");
  }
}
```



