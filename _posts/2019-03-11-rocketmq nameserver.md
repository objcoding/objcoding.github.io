---
layout: post
title: "RocketMQ源码分析之路由中心"
categories: RocketMQ
tags: 注册发现 路由 netty
author: zch
---

* content
{:toc}
早期的rocketmq版本的路由功能是使用zookeeper实现的，后来rocketmq为了追求性能，自己实现了一个性能更高效且实现简单的路由中心NameServer，而且可以通过部署多个路由节点实现高可用，但它们之间并不能互相通信，这也就会导致在某一个时刻各个路由节点间的数据并不完全相同，但数据某个时刻不一致并不会导致消息发送不了，这也是rocketmq追求简单高效的一个做法。









## 路由启动

看了Nameserver的源码后大呼惊叹，整个NameServer总共就由这么几个类类组成：

![rocketmq nameserver](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/rocketmq_1.png)

其中NamesrvStartup为启动类，NamesrvController为核心控制器，RouteInfoManager为路由信息表。

知道了这几个类的功能之后，我们就直接定位到NamesrvStartup启动类的启动方法：

org.apache.rocketmq.namesrv.NamesrvStartup#main0：

```java
public static NamesrvController main0(String[] args) {
    try {
        // 步骤一
        NamesrvController controller = createNamesrvController(args);
        // 步骤二
        start(controller);
        String tip = "The Name Server boot success. serializeType=" + RemotingCommand.getSerializeTypeConfigInThisServer();
        log.info(tip);
        System.out.printf("%s%n", tip);
        return controller;
    } catch (Throwable e) {
        e.printStackTrace();
        System.exit(-1);
    }
    return null;
}
```

整个NameServer服务启动的流程代码都在main0(String[] args)方法了，看起来是不是很简单，我们继续往下撸它的具体实现：



步骤一：

org.apache.rocketmq.namesrv.NamesrvStartup#createNamesrvController：

这个方法的代码有点多，下面我会拆分成几段进行分析:

```java
// 创建命令行参数对象，这里定义了 -h 和 -n参数
Options options = ServerUtil.buildCommandlineOptions(new Options());
// 根据Options和运行时参数args生成命令行对象，buildCommandlineOptions定义了-c参数（Name server config properties file）和-p参数（Print all config item）
commandLine = ServerUtil.parseCmdLine("mqnamesrv", args, buildCommandlineOptions(options), new PosixParser());
if (null == commandLine) {
    System.exit(-1);
    return null;
}
```

这里使用了Apache Commons CLI 命令行解析工具，它可以帮助开发者快速构建启动命令，并且帮助你组织命令的参数、以及输出列表等。

这段主要是根据运行时传递的参数生成commandLine命令行对象，用于解析运行时类似于 -c 指定文件路径，然后填充到namesrvConfig和nettyServerConfig对象中。

```java
final NamesrvConfig namesrvConfig = new NamesrvConfig();
final NettyServerConfig nettyServerConfig = new NettyServerConfig();
nettyServerConfig.setListenPort(9876);
if (commandLine.hasOption('c')) {
    // 读取命令行-c参数指定的配置文件
    String file = commandLine.getOptionValue('c');
    if (file != null) {
        // 将文件转成输入流
        InputStream in = new BufferedInputStream(new FileInputStream(file));
        properties = new Properties();
        // 加载到属性对象
        properties.load(in);
        // 装载配置
        MixAll.properties2Object(properties, namesrvConfig);
        MixAll.properties2Object(properties, nettyServerConfig);
    }
}
```

这段是createNamesrvController(String[] args)方法最为核心的代码，从代码知道先创建NamesrvConfig和NettyServerConfig对象，接着利用commandLine命令行工具读取-c指定的配置文件路径，然后将其读取到流中，生成properties对象，最后将namesrvConfig和nettyServerConfig对象进行初始化。

```java
final NamesrvController controller = new NamesrvController(namesrvConfig, nettyServerConfig);

// remember all configs to prevent discard
controller.getConfiguration().registerConfig(properties);
```

到这里就水到渠成地利用namesrvConfig和nettyServerConfig对象创建NamesrvController对象，然后在注册一遍properties防止丢失。

createNamesrvController(String[] args)这一步也是启动nameserver最为关键的操作，它为我们启动时提供了namesrvConfig和nettyServerConfig配置对象，以及创建NamesrvController核心控制器。

步骤二：

org.apache.rocketmq.namesrv.NamesrvStartup#start：

```java
public static NamesrvController start(final NamesrvController controller) throws Exception {

    if (null == controller) {
        throw new IllegalArgumentException("NamesrvController is null");
    }

    // 对核心控制器进行初始化操作
    boolean initResult = controller.initialize();
    if (!initResult) {
        controller.shutdown();
        System.exit(-3);
    }

    // 注册一个钩子函数，用于JVM进程关闭时，优雅地释放netty服务、线程池等资源
    Runtime.getRuntime().addShutdownHook(new ShutdownHookThread(log, new Callable<Void>() {
        @Override
        public Void call() throws Exception {
            controller.shutdown();
            return null;
        }
    }));

    // 核心控制器启动操作
    controller.start();

    return controller;
}
```

步骤二也是一个次级的启动流程控制方法，该方法主要对核心控制器进行初始化操作，同时注册一个钩子函数，用于JVM进程关闭时，优雅地释放netty服务、线程池等资源，最后对核心控制器进行启动操作，接下来我们继续撸详细实现：

org.apache.rocketmq.namesrv.NamesrvController#initialize：

```java
public boolean initialize() {
    // 加载KV配置
    this.kvConfigManager.load();
    // 创建Netty网络服务对象
    this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);

    this.remotingExecutor =
        Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));
    this.registerProcessor();

    // 创建定时任务--每个10s扫描一次Broker，并定时剔除不活跃的Broker
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
        @Override
        public void run() {
            NamesrvController.this.routeInfoManager.scanNotActiveBroker();
        }
    }, 5, 10, TimeUnit.SECONDS);

    // 创建定时任务--每个10分钟打印一遍KV配置
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
        @Override
        public void run() {
            NamesrvController.this.kvConfigManager.printAllPeriodically();
        }
    }, 1, 10, TimeUnit.MINUTES);

    // ...

    return true;
}
```

该方法主要是对核心控制器进行启动前的一些初始化操作，包括根据NamesrvConfig的kvConfigPath存储KV配置属性的路径加载KV配置，创建定时任务：每个10s扫描一次Broker，并定时剔除不活跃的Broker；每个10分钟打印一遍KV配置。

这里的每个10s扫描一次Broker，并定时剔除不活跃的Broker，这里是路由删除的一些逻辑，后面会讲到。

org.apache.rocketmq.namesrv.NamesrvController#start：

```java
public void start() throws Exception {
    this.remotingServer.start();

    if (this.fileWatchService != null) {
        this.fileWatchService.start();
    }
}
```

到这里对启动进行最后一步操作，即创建Netty服务，我们知道rocketmq是通过netty来进行通信，对应netty的一些细节这里就不展开讲了，后面我也会计划写一些关于netty的系列文章，敬请期待。

路由启动时序图：

![nameServer startup](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/rocketmq_2.png)



## 路由注册

路由注册即是Broker向Nameserver注册的过程，它们是通过Broker的心跳功能实现的，那么既然Nameserver是用来存储Broker的注册信息，那么我们就先来看看Nameserver到底存储了哪些信息，回到文章最开始的那张结构图，我们知道RouteInfoManager为路由信息表：

org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager：

```java
public class RouteInfoManager {
    private static final InternalLogger log = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_LOGGER_NAME);
    private final static long BROKER_CHANNEL_EXPIRED_TIME = 1000 * 60 * 2;
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    private final HashMap<String/* topic */, List<QueueData>> topicQueueTable;
    private final HashMap<String/* brokerName */, BrokerData> brokerAddrTable;
    private final HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
    private final HashMap<String/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;
    private final HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
}
```

- topicQueueTable：Topic消息队列路由信息，包括topic所在的broker名称，读队列数量，写队列数量，同步标记等信息，rocketmq根据topicQueueTable的信息进行负载均衡消息发送。
- brokerAddrTable：Broker节点信息，包括brokername，所在集群名称，还有主备节点信息。
- clusterAddrTable：Broker集群信息，存储了集群中所有的Brokername。
- brokerLiveTable：Broker状态信息，Nameserver每次收到Broker的心跳包就会更新该信息。

这里也先讲一下rocketmq是基于订阅发布机制，我之前也写过一篇文章《rocketmq的消费模式》，我们可知一个Topic拥有多个消息队列，如果不指定队列的数量，一个Broker会为每个Topic创建4个读队列和4个写队列，多个Broker组成集群，Broker会通过发送心跳包将自己的信息注册到路由中心，路由中心brokerLiveTable存储Broker的状态，它会根据Broker的心跳包更新Broker状态信息。

步骤一：Broker发送心跳包

org.apache.rocketmq.broker.BrokerController#start：

```java
public void start() throws Exception {
    
    // 初次启动，这里会强制执行发送心跳包
    this.registerBrokerAll(true, false, true);
    
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
        @Override
        public void run() {
            try {
                BrokerController.this.registerBrokerAll(true, false, brokerConfig.isForceRegister());
            } catch (Throwable e) {
                log.error("registerBrokerAll Exception", e);
            }
        }
    }, 1000 * 10, Math.max(10000, Math.min(brokerConfig.getRegisterNameServerPeriod(), 60000)), TimeUnit.MILLISECONDS);
}
```

Broker在核心控制器启动时，会强制发送一次心跳包，接着创建一个定时任务，定时向路由中心发送心跳包。

org.apache.rocketmq.broker.BrokerController#registerBrokerAll：

```java
public synchronized void registerBrokerAll(final boolean checkOrderConfig, boolean oneway, boolean forceRegister) {
    // 创建一个topic包装类
    TopicConfigSerializeWrapper topicConfigWrapper = this.getTopicConfigManager().buildTopicConfigSerializeWrapper();

    // 这里比较有趣，如果该broker没有读写权限，那么会新建一个临时的topicConfigTable，再set进包装类
    if (!PermName.isWriteable(this.getBrokerConfig().getBrokerPermission())
        || !PermName.isReadable(this.getBrokerConfig().getBrokerPermission())) {
        ConcurrentHashMap<String, TopicConfig> topicConfigTable = new ConcurrentHashMap<String, TopicConfig>();
        for (TopicConfig topicConfig : topicConfigWrapper.getTopicConfigTable().values()) {
            TopicConfig tmp =
                new TopicConfig(topicConfig.getTopicName(), topicConfig.getReadQueueNums(), topicConfig.getWriteQueueNums(),
                                this.brokerConfig.getBrokerPermission());
            topicConfigTable.put(topicConfig.getTopicName(), tmp);
        }
        topicConfigWrapper.setTopicConfigTable(topicConfigTable);
    }

     // 判断是否该Broker是否需要发送心跳包
    if (forceRegister || needRegister(this.brokerConfig.getBrokerClusterName(),
                                      this.getBrokerAddr(),
                                      this.brokerConfig.getBrokerName(),
                                      this.brokerConfig.getBrokerId(),
                                      this.brokerConfig.getRegisterBrokerTimeoutMills())) {
        // 执行发送心跳包
        doRegisterBrokerAll(checkOrderConfig, oneway, topicConfigWrapper);
    }
}
```

该方法是Broker执行发送心跳包的核心控制方法，它主要做了topic的包装类封装操作，判断Broker此时是否需要执行发送心跳包，但我查了下org.apache.rocketmq.common.BrokerConfig#forceRegister字段的值永远等于true，也就是该判断永远为true，即每次都需要发送心跳包。

我们定位到needRegister远程调用到路由中心的方法：

org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager#isBrokerTopicConfigChanged：

```java
public boolean isBrokerTopicConfigChanged(final String brokerAddr, final DataVersion dataVersion) {
    DataVersion prev = queryBrokerTopicConfig(brokerAddr);
    return null == prev || !prev.equals(dataVersion);
}

public DataVersion queryBrokerTopicConfig(final String brokerAddr) {
    BrokerLiveInfo prev = this.brokerLiveTable.get(brokerAddr);
    if (prev != null) {
        return prev.getDataVersion();
    }
    return null;
}
```

发现，Broker是否需要发送心跳包由该Broker在路由中心org.apache.rocketmq.namesrv.routeinfo.BrokerLiveInfo#dataVersion决定，如果dataVersion为空或者当前dataVersion不等于brokerLiveTable存储的brokerLiveTable，Broker就需要发送心跳包。



步骤二：Nameserver处理心跳包

Nameserver的netty服务监听收到心跳包之后，会调用到路由中心以下方法进行处理：

org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager#registerBroker：

```java
public RegisterBrokerResult registerBroker(
    final String clusterName,
    final String brokerAddr,
    final String brokerName,
    final long brokerId,
    final String haServerAddr,
    final TopicConfigSerializeWrapper topicConfigWrapper,
    final List<String> filterServerList,
    final Channel channel) {
    RegisterBrokerResult result = new RegisterBrokerResult();
    try {
        try {
            this.lock.writeLock().lockInterruptibly();

            // 获取集群下所有的Broker，并将当前Broker加入clusterAddrTable，由于brokerNames是Set结构，并不会重复
            Set<String> brokerNames = this.clusterAddrTable.get(clusterName);
            if (null == brokerNames) {
                brokerNames = new HashSet<String>();
                this.clusterAddrTable.put(clusterName, brokerNames);
            }
            brokerNames.add(brokerName);

            boolean registerFirst = false;

            // 获取Broker信息，如果是首次注册，那么新建一个BrokerData并加入brokerAddrTable
            BrokerData brokerData = this.brokerAddrTable.get(brokerName);
            if (null == brokerData) {
                registerFirst = true;
                brokerData = new BrokerData(clusterName, brokerName, new HashMap<Long, String>());
                this.brokerAddrTable.put(brokerName, brokerData);
            }
            // 这里判断Broker是否是已经注册过
            String oldAddr = brokerData.getBrokerAddrs().put(brokerId, brokerAddr);
            registerFirst = registerFirst || (null == oldAddr);

            // 如果是Broker是Master节点吗，并且Topic信息更新或者是首次注册，那么创建更新topic队列信息
            if (null != topicConfigWrapper
                && MixAll.MASTER_ID == brokerId) {
                if (this.isBrokerTopicConfigChanged(brokerAddr, topicConfigWrapper.getDataVersion())
                    || registerFirst) {
                    ConcurrentMap<String, TopicConfig> tcTable =
                        topicConfigWrapper.getTopicConfigTable();
                    if (tcTable != null) {
                        for (Map.Entry<String, TopicConfig> entry : tcTable.entrySet()) {
                            this.createAndUpdateQueueData(brokerName, entry.getValue());
                        }
                    }
                }
            }

            // 更新BrokerLiveInfo状态信息
            BrokerLiveInfo prevBrokerLiveInfo = 
                this.brokerLiveTable.put(brokerAddr,new BrokerLiveInfo(System.currentTimeMillis(),topicConfigWrapper.getDataVersion(),channel,haServerAddr));
        } finally {
            this.lock.writeLock().unlock();
        }
    } catch (Exception e) {
        log.error("registerBroker Exception", e);
    }

    return result;
}
```

该方法是处理Broker心跳包的最核心方法，它主要做了对RouteInfoManager路由信息的一些更新操作，包括对clusterAddrTable、brokerAddrTable、topicQueueTable、brokerLiveTable等路由信息。



路由注册时序图：

![Broker register](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/rocketmq_3.png)



## 路由删除

前面部分我们分析了Nameserver启动时会创建一个定时任务，定时剔除不活跃的Broker。

org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager#scanNotActiveBroker：

```java
public void scanNotActiveBroker() {
    Iterator<Entry<String, BrokerLiveInfo>> it = this.brokerLiveTable.entrySet().iterator();
    while (it.hasNext()) {
        Entry<String, BrokerLiveInfo> next = it.next();
        long last = next.getValue().getLastUpdateTimestamp();
        // 如果当前时间大于最后修改时间加上Broker过期时间，那么就剔除该Broker
        if ((last + BROKER_CHANNEL_EXPIRED_TIME) < System.currentTimeMillis()) {
            // 关闭Broker对应的channel
            RemotingUtil.closeChannel(next.getValue().getChannel());
            it.remove();
            log.warn("The broker channel expired, {} {}ms", next.getKey(), BROKER_CHANNEL_EXPIRED_TIME);
            // 从brokerLiveTable、brokerAddrTable、topicQueueTable移除Broker相关信息
            this.onChannelDestroy(next.getKey(), next.getValue().getChannel());
        }
    }
}
```

剔除Broker信息的逻辑比较简单，首先从BrokerLiveInfo获取状态信息，判断Broker的心跳时间是否已超过限定值，若超过之后就执行剔除逻辑。



分析完了整个namesever源码，我的得知，其实我们自己实现一个注册发现貌似也不难。有时候我们发现公司有些项目可以独立拆分出来做成中间件的形式，也就是单独部署，其它业务依赖client包调用中间件服务，比如短信、推送、邮件、配置等模块，如果还需要将这些中间件做成高可用那么就需要自己实现一个注册发现中心。






