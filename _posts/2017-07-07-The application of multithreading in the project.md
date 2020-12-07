---
layout: post
title: "多线程在项目中的运用"
categories: Java
tags: 多线程
author: 张乘辉
---

* content
{:toc}
在项目开发过程中，我发现在业务中一次性需要多次访问数据库才能获取业务所需要的数据，又或者在一个业务点上需要同时大量向数据库插入或修改数据，这时如果不用并发处理，就会导致系统性能底下，数据库访问粗粒度大，单个线程负荷大，在分布式系统中容易导致雪崩效应，因此，需要对这些业务用并发加以处理。











## CompletableFuture API

CompletableFuture 类是Java8新增加的类，它继承了Future接口，但Future只能在线程执行任务时可异步处理，但只能通过阻塞的方式获取执行结果，期间主线程被阻塞，这相当于是半异步执行吧，而CompletableFuture则可实现全异步执行，通过以下静态方法创建CompletableFuture对象：

```java
public static CompletableFuture<Void> runAsync(Runnable runnable)
public static CompletableFuture<Void> runAsync(Runnable runnable, Executor executor)
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier)
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier, Executor executor)
```

通过以上方法创建一个CompletableFuture，传一个Runnable或者Supplier对象，这个对象区别就是Runnable无返回值，而Supplier有返回值，然后再通过thenxxx()方法执行上一个CompletableFuture的执行结果，这个方法可实现异步执行而不会阻塞主线程：

- 转换

```java
public <U> CompletableFuture<U> thenApply(Function<? super T,? extends U> fn)
public <U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn)
public <U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn, Executor executor)
```

- 纯消费

```java
public CompletableFuture<Void> 	thenAccept(Consumer<? super T> action)
public CompletableFuture<Void> 	thenAcceptAsync(Consumer<? super T> action)
public CompletableFuture<Void> 	thenAcceptAsync(Consumer<? super T> action, Executor executor)
```

以上两类方法都可以异步执行，区别是转换有返回值，纯消费无返回值，通常出消费在异步执行链中的末端位置，执行最终的处理，而转换通常处于还需要继续对执行结果进行下一步处理的时候用；无Async后缀的方法继续在当前线程处理，而有Async后缀的可以在其它线程处理上一个执行结果。

```java
CompletableFuture<Integer> f1 = CompletableFuture.supplyAsync(() -> {
    return 100;
});
CompletableFuture<Integer> f3 =  f2.thenApplyAsync(i -> return i + 100);
CompletableFuture<Void> f4 =  f3.thenApplyAsync(i -> System.out.println(i));
System.out.println(f1.get());// 这个输出不会被f3和f4阻塞
```



## 处理跨系统的数据拼装

```java
public JSONObject getOrdersByPage(int page, JSONObject jsonObject) {

  // 此处省略部分代码

  PageData<IntegralOrder4WebManage> pd = new PageData<>(page);
  List<IntegralOrder> integralOrders =
    orderMapper.getOrdersByPage(start, end, tradeNo, shopName, shopId, orderStatus, orderType, pd.getPageRow(), pd.getPageSize() + 1);
  
  // 创建线程池
  ExecutorService executor = Executors.newFixedThreadPool(5);
  // 异步执行数据拼装
  List<CompletableFuture<IntegralOrder4WebManage>> futures = integralOrders.stream()
    .map(integralOrder -> CompletableFuture.supplyAsync(() -> this.integralOrder4WebManage2Domain(integralOrder), executor))
    .collect(Collectors.toList());
  executor.shutdown();// 关闭线程，需要等待任务全部执行完才关闭，期间线程不会再接收任务

  List<IntegralOrder4WebManage> webManages = futures.stream()
    .map(future -> {
      try {
        return future.get();// 阻塞式获取执行结果
      } catch (InterruptedException | ExecutionException e) {
        e.printStackTrace();
      }
      return null;
    })
    .collect(Collectors.toList());
  
  // 此处省略部分代码
  
}
```

```java
 private IntegralOrder4WebManage integralOrder4WebManage2Domain(IntegralOrder integralOrder) {
//        logger.info("当前线程: " + Thread.currentThread().getName());
        IntegralOrder4WebManage webManage = new IntegralOrder4WebManage(integralOrder);
        JSONObject shopServerResult = restTemplate.getForObject("http://shop-server/api/shop/" + webManage.getShopId(), JSONObject.class);
        Shop shop = shopServerResult.getObject("shop", Shop.class);
        if (shop != null) {
            webManage.setShopCode(shop.getShopCode());
            webManage.setShopNo(shop.getShopNo());
            webManage.setShopFloor(shop.getShopFloor());
            webManage.setShopSno(shop.getShopSno());
        }
        return webManage;
    }
```


