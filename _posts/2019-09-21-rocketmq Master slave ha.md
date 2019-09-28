---
layout: post
title: "RocketMQ主从同步源码分析"
categories: RocketMQ
tags: HA Master Slave 
author: zch
---

* content
{:toc}
之前写了一篇关于 RocketMQ 队列与 Kafka 分区副本的区别文章，里面提到了 RocketMQ 的消息冗余主要是通过主备同步机制实现的，这跟 Kafka 分区副本的 Leader-Follower 模型不同，HA（High Available) 指的是高可用性，而 RocketMQ 的HA机制是通过主备同步实现消息的高可用。









## HA 核心类

HA 的实现逻辑放在了 store 存储模块的ha目录中，其核心实现类如下：

![](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/rocketmq_14.png)

1. HAService：主从同步的核心实现类
2. HAService$AcceptSocketService：主服务器监听从服务器连接实现类
3. HAService$GroupTransferService：主从同步通知类，实现同步复制和异步复制的功能
4. HAService$HAClient：从服务器连接主服务实现类
5. HAConnection：主服务端 HA 连接对象的封装，当主服务器接收到从服务器发过来的消息后，会封装成一个 HAConnection 对象，其中里面又封装了读 Socket 连接实现与 写 Socket 连接实现：

- HAConnection$ReadSocketService：主服务器读实现类
- HAConnection$WriteSocketService：主服务器写实现类



RocketMQ 主从同步的整体工作机制大致是：

1. 从服务器主动建立 TCP 连接主服务器，然后每隔 5s 向主服务器发送 commitLog 文件最大偏移量拉取还未同步的消息；
2. 主服务器开启监听端口，监听从服务器发送过来的信息，主服务器收到从服务器发过来的偏移量进行解析，并返回查找出未同步的消息给从服务器；
3. 客户端收到主服务器的消息后，将这批消息写入 commitLog 文件中，然后更新 commitLog 拉取偏移量，接着继续向主服务拉取未同步的消息。



## Slave -> Master 过程

从 HA 实现逻辑可看出，可大致分为两个过程，分别是从服务器上报偏移量，以及主服务器发送未同步消息到从服务器。

从上面的实现类可知，从服务器向主服务器上报偏移量的逻辑在 HAClient 类中，HAClient 类是一个继承了 ServiceThread 类，即它是一个线程服务类，在 Broker 启动后，Broker 启动开一条线程定时执行从服务器上报偏移量到主服务器的任务。

org.apache.rocketmq.store.ha.HAService.HAClient#run:

```java
public void run() {
  log.info(this.getServiceName() + " service started");

  while (!this.isStopped()) {
    try {
      // 主动连接主服务器，获取socketChannel对象
      if (this.connectMaster()) {
        if (this.isTimeToReportOffset()) {
          // 执行上报偏移量到主服务器
          boolean result = this.reportSlaveMaxOffset(this.currentReportedOffset);
          if (!result) {
            this.closeMaster();
          }
        }
				// 每隔一秒钟轮询一遍
        this.selector.select(1000);

        // 处理主服务器发送过来的消息
        boolean ok = this.processReadEvent();
        if (!ok) {
          this.closeMaster();
        }
        
        // ......
        
      } else {
        this.waitForRunning(1000 * 5);
      }
    } catch (Exception e) {
      log.warn(this.getServiceName() + " service has exception. ", e);
      this.waitForRunning(1000 * 5);
    }
  }

  log.info(this.getServiceName() + " service end");
}
```

以上是 HAClient 线程 run 方法逻辑，主要是做了主动连接主服务器，并上报偏移量到主服务器，以及处理主服务器发送过来的消息，并不断循环执行以上逻辑。

org.apache.rocketmq.store.ha.HAService.HAClient#connectMaster:

```java
private boolean connectMaster() throws ClosedChannelException {
  if (null == socketChannel) {
    String addr = this.masterAddress.get();
    if (addr != null) {
      SocketAddress socketAddress = RemotingUtil.string2SocketAddress(addr);
      if (socketAddress != null) {
        this.socketChannel = RemotingUtil.connect(socketAddress);
        if (this.socketChannel != null) {
          this.socketChannel.register(this.selector, SelectionKey.OP_READ);
        }
      }
    }
    this.currentReportedOffset = HAService.this.defaultMessageStore.getMaxPhyOffset();
    this.lastWriteTimestamp = System.currentTimeMillis();
  }
  return this.socketChannel != null;
}
```

该方法是从服务器连接主服务器的逻辑，拿到主服务器地址并且连接上以后，会获取一个 socketChannel 对象，接着还会记录当前时间戳为上次写入的时间戳，lastWriteTimestamp 的作用时用来计算主从同步时间间隔，这里需要注意一点，如果没有配置主服务器地址，该方法会返回 false，即不会执行主从复制。

该方法还会调用 DefaultMessageStore 的 getMaxPhyOffset() 方法获取 commitLog 文件最大偏移量，作为本次上报的偏移量。

org.apache.rocketmq.store.ha.HAService.HAClient#reportSlaveMaxOffset:

```java
private boolean reportSlaveMaxOffset(final long maxOffset) {
  this.reportOffset.position(0);
  this.reportOffset.limit(8);
  this.reportOffset.putLong(maxOffset);
  this.reportOffset.position(0);
  this.reportOffset.limit(8);

  for (int i = 0; i < 3 && this.reportOffset.hasRemaining(); i++) {
    try {
      this.socketChannel.write(this.reportOffset);
    } catch (IOException e) {
      log.error(this.getServiceName()
                + "reportSlaveMaxOffset this.socketChannel.write exception", e);
      return false;
    }
  }
  return !this.reportOffset.hasRemaining();
}
```

该方法向主服务器上报已拉取偏移量，具体做法是将 ByteBuffer 读取位置 position 值为 0，其实跳用 flip() 方法也可以，然后调用 putLong() 方法将 maxOffset 写入 ByteBuffer，将 limit 设置为 8，跟写入 ByteBuffer 中的 maxOffset（long 型）大小一样，最后采取 for 循环将 maxOffset 写入网络通道中，并调用 hasRemaining() 方法，该方法的逻辑为判断 position 是否小于 limit，即判断 ByteBuffer 中的字节流是否全部写入到通道中。



## Master -> Slave 过程

org.apache.rocketmq.store.ha.HAService.AcceptSocketService#run:

```java
public void run() {
  log.info(this.getServiceName() + " service started");

  while (!this.isStopped()) {
    try {
      this.selector.select(1000);
      Set<SelectionKey> selected = this.selector.selectedKeys();

      if (selected != null) {
        for (SelectionKey k : selected) {
          if ((k.readyOps() & SelectionKey.OP_ACCEPT) != 0) {
            SocketChannel sc = ((ServerSocketChannel) k.channel()).accept();

            if (sc != null) {
              HAService.log.info("HAService receive new connection, "
                                 + sc.socket().getRemoteSocketAddress());

              try {
                HAConnection conn = new HAConnection(HAService.this, sc);
                conn.start();
                HAService.this.addConnection(conn);
              } catch (Exception e) {
                log.error("new HAConnection exception", e);
                sc.close();
              }
            }
          } else {
            log.warn("Unexpected ops in select " + k.readyOps());
          }
        }

        selected.clear();
      }
    } catch (Exception e) {
      log.error(this.getServiceName() + " service has exception.", e);
    }
  }

  log.info(this.getServiceName() + " service end");
}
```

主服务器收到从服务器的拉取偏移量后，会封装成一个 HAConnection 对象，前面也说过 HAConnection 封装主服务端 HA 连接对象的封装，其中有读实现类和写实现类，start() 方法即开启了读写线程：

org.apache.rocketmq.store.ha.HAConnection#start：

```java
public void start() {
  this.readSocketService.start();
  this.writeSocketService.start();
}
```

org.apache.rocketmq.store.ha.HAConnection.ReadSocketService#processReadEvent:

```java
private boolean processReadEvent() {
  int readSizeZeroTimes = 0;

  if (!this.byteBufferRead.hasRemaining()) {
    this.byteBufferRead.flip();
    this.processPostion = 0;
  }

  while (this.byteBufferRead.hasRemaining()) {
    try {
      int readSize = this.socketChannel.read(this.byteBufferRead);
      if (readSize > 0) {
        readSizeZeroTimes = 0;
        this.lastReadTimestamp = HAConnection.this.haService.getDefaultMessageStore().getSystemClock().now();
        if ((this.byteBufferRead.position() - this.processPostion) >= 8) {
          int pos = this.byteBufferRead.position() - (this.byteBufferRead.position() % 8);
          // 从网络通道中读取从服务器上报的偏移量
          long readOffset = this.byteBufferRead.getLong(pos - 8);
          this.processPostion = pos;

          // 同步从服务器偏移量
          HAConnection.this.slaveAckOffset = readOffset;
          if (HAConnection.this.slaveRequestOffset < 0) {
            HAConnection.this.slaveRequestOffset = readOffset;
            log.info("slave[" + HAConnection.this.clientAddr + "] request offset " + readOffset);
          }

          // 这里主要是同步后需要唤醒相关消息发送线程，实现主从同步是异步还是同步的功能
          HAConnection.this.haService.notifyTransferSome(HAConnection.this.slaveAckOffset);
        }
      } else if (readSize == 0) {
        if (++readSizeZeroTimes >= 3) {
          break;
        }
      } else {
        log.error("read socket[" + HAConnection.this.clientAddr + "] < 0");
        return false;
      }
    } catch (IOException e) {
      log.error("processReadEvent exception", e);
      return false;
    }
  }

  return true;
}
```

从以上源码可看出，主服务器接收到从服务器上报的偏移量后，主要作了两件事：

1. 获取从服务器上报的偏移量；
2. 唤醒主从同步消费者发送消息同步返回的线程，该方法实现了主从同步-同步复制的功能。

org.apache.rocketmq.store.ha.HAConnection.WriteSocketService#run:

```java
public void run() {
  HAConnection.log.info(this.getServiceName() + " service started");

  while (!this.isStopped()) {
    try {
      this.selector.select(1000);

      // 如果slaveRequestOffset=-1，说明读线程还没有获取从服务器的偏移量，继续循环等待
      if (-1 == HAConnection.this.slaveRequestOffset) {
        Thread.sleep(10);
        continue;
      }

      // 如果nextTransferFromWhere=-1，说明线程刚开始执行数据传输
      if (-1 == this.nextTransferFromWhere) {
        // 如果slaveRequestOffset=0，说明从服务器是第一次上报偏移量
        if (0 == HAConnection.this.slaveRequestOffset) {
          // 获取最后一个 commitLog 文件且还未读取消费的偏移量
          long masterOffset = HAConnection.this.haService.getDefaultMessageStore().getCommitLog().getMaxOffset();
          // 求出最后一个commitLog偏移量的初始偏移量
          masterOffset =
            masterOffset
            - (masterOffset % HAConnection.this.haService.getDefaultMessageStore().getMessageStoreConfig()
               .getMapedFileSizeCommitLog());

          if (masterOffset < 0) {
            masterOffset = 0;
          }

          // 更新 nextTransferFromWhere
          this.nextTransferFromWhere = masterOffset;
        } else {
          // 如果slaveRequestOffset!=0,则将该值赋值给nextTransferFromWhere
          this.nextTransferFromWhere = HAConnection.this.slaveRequestOffset;
        }

        log.info("master transfer data from " + this.nextTransferFromWhere + " to slave[" + HAConnection.this.clientAddr
                 + "], and slave request " + HAConnection.this.slaveRequestOffset);
      }

      // 判断上次写事件是否已全部写完成
      if (this.lastWriteOver) {

        // 计算是否已到发送心跳包时间
        long interval =
          HAConnection.this.haService.getDefaultMessageStore().getSystemClock().now() - this.lastWriteTimestamp;
        // 发送心跳包，以保持长连接
        if (interval > HAConnection.this.haService.getDefaultMessageStore().getMessageStoreConfig()
            .getHaSendHeartbeatInterval()) {
          // Build Header
          this.byteBufferHeader.position(0);
          this.byteBufferHeader.limit(headerSize);
          this.byteBufferHeader.putLong(this.nextTransferFromWhere);
          this.byteBufferHeader.putInt(0);
          this.byteBufferHeader.flip();
          this.lastWriteOver = this.transferData();
          if (!this.lastWriteOver)
            continue;
        }
      } else {
        this.lastWriteOver = this.transferData();
        if (!this.lastWriteOver)
          continue;
      }

      // 获取同步消息数据
      SelectMappedBufferResult selectResult =      HAConnection.this.haService.getDefaultMessageStore().getCommitLogData(this.nextTransferFromWhere);
      if (selectResult != null) {
        int size = selectResult.getSize();
        if (size > HAConnection.this.haService.getDefaultMessageStore().getMessageStoreConfig().getHaTransferBatchSize()) {
          size = HAConnection.this.haService.getDefaultMessageStore().getMessageStoreConfig().getHaTransferBatchSize();
        }

        long thisOffset = this.nextTransferFromWhere;
        this.nextTransferFromWhere += size;

        selectResult.getByteBuffer().limit(size);
        this.selectMappedBufferResult = selectResult;

        // Build Header
        this.byteBufferHeader.position(0);
        this.byteBufferHeader.limit(headerSize);
        this.byteBufferHeader.putLong(thisOffset);
        this.byteBufferHeader.putInt(size);
        this.byteBufferHeader.flip();

        // 传输消息到从服务器
        this.lastWriteOver = this.transferData();
      } else {

        HAConnection.this.haService.getWaitNotifyObject().allWaitForRunning(100);
      }
    } catch (Exception e) {

      HAConnection.log.error(this.getServiceName() + " service has exception.", e);
      break;
    }
  }

  if (this.selectMappedBufferResult != null) {
    this.selectMappedBufferResult.release();
  }

  this.makeStop();

  readSocketService.makeStop();

  haService.removeConnection(HAConnection.this);

  SelectionKey sk = this.socketChannel.keyFor(this.selector);
  if (sk != null) {
    sk.cancel();
  }

  try {
    this.selector.close();
    this.socketChannel.close();
  } catch (IOException e) {
    HAConnection.log.error("", e);
  }

  HAConnection.log.info(this.getServiceName() + " service end");
}
```

读实现类实现逻辑比较长，但主要做了以下几件事情：

1. 计算需要拉取的偏移量，如果从服务器第一次拉取，则从最后一个 commitLog 文件的初始偏移量开始同步；
2. 传输消息到从服务器；
3. 发送心跳包到从服务器，保持长连接。

关于第一步，我还需要详细讲解一下，因为之前有想到一个问题：

把 brokerA 的从服务器去掉，再启动一台新的从服务器指向brokerA 主服务器，这时的主服务器的消息是否会全量同步到从服务？

org.apache.rocketmq.store.MappedFileQueue#getMaxOffset:

```java
public long getMaxOffset() {
    MappedFile mappedFile = getLastMappedFile();
    if (mappedFile != null) {
        return mappedFile.getFileFromOffset() + mappedFile.getReadPosition();
    }
    return 0;
}
```

org.apache.rocketmq.store.ha.HAConnection.WriteSocketService#run:

```java
// 求出最后一个commitLog偏移量的初始偏移量
masterOffset =
  masterOffset
            - (masterOffset % HAConnection.this.haService.getDefaultMessageStore().getMessageStoreConfig()
               .getMapedFileSizeCommitLog());
```

从以上逻辑可找到答案，如果有新的从服务器同步主服务器消息，则从最后一个 commitLog 文件的初始偏移量开始同步。

回到最开始开启 HAClient 线程上报偏移量的方法，我们发现里面还做了一件事：

```java
// 处理主服务器发送过来的消息
boolean ok = this.processReadEvent();
```

org.apache.rocketmq.store.ha.HAService.HAClient#processReadEvent:

```java
private boolean processReadEvent() {
  int readSizeZeroTimes = 0;
  while (this.byteBufferRead.hasRemaining()) {
    try {
      int readSize = this.socketChannel.read(this.byteBufferRead);
      if (readSize > 0) {
        lastWriteTimestamp = HAService.this.defaultMessageStore.getSystemClock().now();
        readSizeZeroTimes = 0;
        // 读取消息并写入commitLog文件中
        boolean result = this.dispatchReadRequest();
        if (!result) {
          log.error("HAClient, dispatchReadRequest error");
          return false;
        }
      } else if (readSize == 0) {
        if (++readSizeZeroTimes >= 3) {
          break;
        }
      } else {
        // TODO ERROR
        log.info("HAClient, processReadEvent read socket < 0");
        return false;
      }
    } catch (IOException e) {
      log.info("HAClient, processReadEvent read socket exception", e);
      return false;
    }
  }

  return true;
}
```

该方法用于处理主服务器发送回来的消息数据，这里用了 while 循环的处理，不断地从 byteBuffer 读取数据到缓冲区中，最后调用 dispatchReadRequest 方法将消息数据写入 commitLog 文件中，完成主从复制最后一个步骤。

最后贴上《RocketMQ 技术内幕》这本书的一张 RocketMQ HA 交互图：

![](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/rocketmq_15.png)





