---
layout: post
title: kafka consumer 退出时死循环问题
category: 技术
tags: Kafka
keywords: Kafka, Thread, Interrupt
description:
---

### 1. 问题背景

这两天在做消息中间件和 storm 的集成，这里的消息中间件是基于 kafka 的改造版本 （改造细节暂且不表）， 由于调度方式的变化 （原有的客户端调度存在一些问题，比如容易出现试图不一样， 不能跨集群调度等问题， 我们抽取了一个中心服务作为客户端的调度），需要改造 storm-kafka 使得其也能够使用新的功能和特性。主要的工作包括替换 storm-kafka 的调度模块，以及添加 offset 提交到 broker 的功能支持。在完成相关代码之后发现了一个问题，就是测试程序间歇性的不能正常退出，打开 INFO 日志级别发现当出现程序不能正常退出的时候会不停的打印如下日志：

	Failed to fetch consumer metadata from $ip:$port.


### 2. 问题定位

开发是基于 kafka 0.8.2.1 根据打印的日志搜索相关代码定位到到应该的代码如下：
	
	def channelToOffsetManagerWithBrokers(group: String, brokers: Seq[Broker], socketTimeoutMs: Int = 3000, retryBackOffMs: Int = 1000) = {
     var queryChannel = channelToRandomOneBroker(brokers)

     var offsetManagerChannelOpt: Option[BlockingChannel] = None

     while (!offsetManagerChannelOpt.isDefined) {

       var coordinatorOpt: Option[Broker] = None

       while (!coordinatorOpt.isDefined) {
         try {
           if (!queryChannel.isConnected)
             queryChannel = channelToRandomOneBroker(brokers)
           debug("Querying %s:%d to locate offset manager for %s.".format(queryChannel.host, queryChannel.port, group))
           queryChannel.send(ConsumerMetadataRequest(group))
           val response = queryChannel.receive()
           val consumerMetadataResponse =  ConsumerMetadataResponse.readFrom(response.buffer)
           debug("Consumer metadata response: " + consumerMetadataResponse.toString)
           if (consumerMetadataResponse.errorCode == ErrorMapping.NoError)
             coordinatorOpt = consumerMetadataResponse.coordinatorOpt
           else {
             debug("Query to %s:%d to locate offset manager for %s failed - will retry in %d milliseconds."
               .format(queryChannel.host, queryChannel.port, group, retryBackOffMs))
             Thread.sleep(retryBackOffMs)
           }
         }
         catch {
           case ioe: IOException =>
             info("Failed to fetch consumer metadata from %s:%d.".format(queryChannel.host, queryChannel.port))
             queryChannel.disconnect()
         }
         ...
		
		
**[说明]:** 代码位于 kafka.client.ClientUtils 中，channelToOffsetManagerWithBrokers 是我们修改的一个函数，基本等同于原有的 channelToOffsetManager，只是去掉了对 zookeeper 的依赖。

函数中有一个 while 循环，在 while 循环的部分如果 catch 到异常会打印上文中出现的日志。 首先我们需要知道是何种类型的异常，通过调试发现此处抛出的 IO 异常为 ClosedChannelException，是由函数```queryChannel.send(ConsumerMetadataRequest(group))``` 抛出的。 send 方法的代码如下：  

	def send(request: RequestOrResponse):Int = {
      if(!connected)
        throw new ClosedChannelException()

      val send = new BoundedByteBufferSend(request)
      send.writeCompletely(writeChannel)
    }
    
从代码中可以看出 queryChannel 处于未连接状态， 这是为什么呢？ queryChannel 由 ```channelToRandomOneBroker(brokers)``` 返回：

	def channelToRandomOneBroker(brokers: Seq[Broker], socketTimeoutMs: Int = 3000) : BlockingChannel = {
     var channel: BlockingChannel = null
     var connected = false
     while (!connected) {
       val allBrokers = brokers
       Random.shuffle(allBrokers).find { broker =>
         trace("Connecting to broker %s:%d.".format(broker.host, broker.port))
         try {
           channel = new BlockingChannel(broker.host, broker.port, BlockingChannel.UseDefaultBufferSize, BlockingChannel.UseDefaultBufferSize, socketTimeoutMs)
           channel.connect() // 此处执行了 connect 操作
           debug("Created channel to broker %s:%d.".format(channel.host, channel.port))
           true
         } catch {
           case e: Exception =>
             if (channel != null) channel.disconnect()
             channel = null
             warn("Error while creating channel to %s:%d.".format(broker.host, broker.port))
             false
         }
       }
       connected = if (channel == null) false else true
     }

     channel
    }

在返回 channel 之前，执行了 channel 的 connect 方法，但是好像并没有连接成功，而且此处并没有抛出异常，否则返回的 channel 应该为 null。 看一下 BlockingChannel.connect 方法：

	def connect() = lock synchronized  {
    if(!connected) {
      try {
        channel = SocketChannel.open()
        if(readBufferSize > 0)
          channel.socket.setReceiveBufferSize(readBufferSize)
        if(writeBufferSize > 0)
          channel.socket.setSendBufferSize(writeBufferSize)
        channel.configureBlocking(true)
        channel.socket.setSoTimeout(readTimeoutMs)
        channel.socket.setKeepAlive(true)
        channel.socket.setTcpNoDelay(true)
        channel.socket.connect(new InetSocketAddress(host, port), connectTimeoutMs) // 此处异常

        writeChannel = channel
        readChannel = Channels.newChannel(channel.socket().getInputStream)
        connected = true
        // settings may not match what we requested above
        val msg = "Created socket with SO_TIMEOUT = %d (requested %d), SO_RCVBUF = %d (requested %d), SO_SNDBUF = %d (requested %d), connectTimeoutMs = %d."
        debug(msg.format(channel.socket.getSoTimeout,
                         readTimeoutMs,
                         channel.socket.getReceiveBufferSize, 
                         readBufferSize,
                         channel.socket.getSendBufferSize,
                         writeBufferSize,
                         connectTimeoutMs))

      } catch {
        case e: Throwable => disconnect()
      }
    }
	
这个方法包裹在一个大的 try ... catch 块中确实不会抛出异常，但是如果在执行代码的过程中有异常，便会调用 disconnect() 使得 channel 处于 unconnected 状态。 这里上述问题的似乎就说得通了，但是是什么原因导致 BlockingChannel 执行 connect 方法一直失败呢？ 一步一步调试的结果发现是在 ```channel.socket.connect``` 的时候抛出了 ClosedByInterruptException 异常。异常栈如下：

	java.nio.channels.ClosedByInterruptException
			at java.nio.channels.spi.AbstractInterruptibleChannel.end(AbstractInterruptibleChannel.java:202)
			at sun.nio.ch.SocketChannelImpl.connect(SocketChannelImpl.java:681)
			at sun.nio.ch.SocketAdaptor.connect(SocketAdaptor.java:107)
			at kafka.network.BlockingChannel.liftedTree1$1(BlockingChannel.scala:59)
			at kafka.network.BlockingChannel.connect(BlockingChannel.scala:49)
			at kafka.producer.SyncProducer.connect(SyncProducer.scala:140)
			at kafka.producer.SyncProducer.getOrMakeConnection(SyncProducer.scala:155)
			at kafka.producer.SyncProducer.kafka$producer$SyncProducer$$doSend(SyncProducer.scala:69)
			at kafka.producer.SyncProducer.send(SyncProducer.scala:113)
			at kafka.client.ClientUtils$.fetchTopicMetadata(ClientUtils.scala:58)
			at kafka.client.ClientUtils$.fetchTopicMetadata(ClientUtils.scala:93)
			at kafka.client.ClientUtils.fetchTopicMetadata(ClientUtils.scala)
			at storm.kafka.CastleCoordinator.getNewPartitions(CastleCoordinator.java:164)
			at storm.kafka.CastleCoordinator.refresh(CastleCoordinator.java:112)
			at storm.kafka.KafkaSpout.nextTuple(KafkaSpout.java:161)
			at backtype.storm.daemon.executor$fn__4476$fn__4491$fn__4522.invoke(executor.clj:607)
			at backtype.storm.util$async_loop$fn__545.invoke(util.clj:479)
			at clojure.lang.AFn.run(AFn.java:22)
			at java.lang.Thread.run(Thread.java:745)

最上层是在 AbstractInterruptibleChannel.end 抛出了异常， 其代码如下：

	protected final void end(boolean completed)
        throws AsynchronousCloseException
    {
        blockedOn(null);
        Thread interrupted = this.interrupted;
        if (interrupted != null && interrupted == Thread.currentThread()) {
            interrupted = null;
            throw new ClosedByInterruptException();
        }
        if (!completed && !open)
            throw new AsynchronousCloseException();
    }

当 interrupted == Thread.currentThread() 的时候会抛出 ClosedByInterruptException。从代码中可以看到 interrupted 在 AbstractInterruptibleChannel.begin 中被设置：

	protected final void begin() {
        if (interruptor == null) {
            interruptor = new Interruptible() {
                    public void interrupt(Thread target) {
                        synchronized (closeLock) {
                            if (!open)
                                return;
                            open = false;
                            interrupted = target;
                            try {
                                AbstractInterruptibleChannel.this.implCloseChannel();
                            } catch (IOException x) { }
                        }
                    }};
        }
        blockedOn(interruptor);
        Thread me = Thread.currentThread();
        if (me.isInterrupted())
            interruptor.interrupt(me);
    }
    
从代码逻辑中我们可以清楚的看到当 Thread.currentThread().isInterrupted() 为 true 时，interrupted 会被设置为当前线程。 SocketChannelImpl.connect 有一段代码的大致逻辑如下：
	
	...
	try {
		...
		begin()
		...
	} finally {
		...
		end()
		...
	}
	...

所以当线程被 interrupt 之后， SocketChannel.connect 便会抛出 ClosedByInterruptException()。 

看了这么多代码之后我们简单总结一下问题的原因：当程序在退出的时候提交 offset 的线程被其他线程 interrupt 了， 被打断的线程执行 Socket.connect 会抛出 ClosedByInterruptException 导致连接不成功， 所以 channel 在发送 request 的时候便会抛出 ClosedChannelException 异常，使得 while(!coordinatorOpt.isDefined) 循环一直不能跳出，最终导致程序不能正常退出。
 

### 3. 问题修复

既然找出了问题的原因，那么修复问题就变得容易了， 这里主要是由于没有正确的处理 interrupt 导致进入了死循环，所以在 channelToOffsetManagerWithBrokers 的 while(!coordinatorOpt.isDefined) 循环中添加如下代码即可：

	         if (Thread.currentThread().isInterrupted) {
               throw new InterruptedException()
             }


### 4. 思考

1. 正确的处理线程的 interrupt 是一门学问， 可以参见：[详细分析Java中断机制
](http://www.infoq.com/cn/articles/java-interrupt-mechanism) or [Java 并发编程学习 - 线程](/2016/03/28/java_concurrency_thread.html) 关于 interrupt 的部分。
2. 需要多多了解 socket 的一些行为特性， 比如线程中断标志被设置之后不能建立连接。
3. 似乎可以算是 kafka client 的一个 bug， 考虑是否在邮件列表里就这个问题讨论一下。


