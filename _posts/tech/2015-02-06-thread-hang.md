---
layout: post
title:  记一次请求外部服务失败率增加的问题
category: 技术
tags: 故障
keywords: 故障恢复, 问题定位
description: 解决故障记录
---

#### 现象

从 Dashboard 上看到 service 请求依赖的某个服务 Exception 数量变得比较高， 异常数量达到了几百的样子。由于机房刚从一次网络故障中恢复，刚开始并没有特别注意，以为是该外部服务仍然处于不健康的状态导致的。后来前端反馈问题说一些重要的推荐结果没有了，由于改服务是提供用户行为的， 在请求用户行为失败之后，service 拿不到用户行为，某些推荐结果就为空了。

#### 定位

在前端反馈一些推荐结果缺失之后，我们开始着手调查为什么请求用户行为的 Request 会出现异常。 首先我们上到 Logstash 查看 Exception 的具体信息, 发现大部分的 Exception 的具体信息如下：
    
    com.hulu.behavior.exception.UserBehaviorFetcherException:Fetch user behavior failed
    exceptionType: UserBehaviorFetchException
    errorLevel: NOTIFY
    requestUrl: "$XXXX"
    HystrixExceptionType: SHORTCIRCUIT
    at com.hulu.behavior.rawfetcher.SingleRawUserBehaviorFetcher.getUserBehaviorRawData(SingleRawUserBehaviorFetcher.java:68)
    at com.hulu.behavior.rawfetcher.SingleRawUserBehaviorFetcher.getRawBehaviorData(SingleRawUserBehaviorFetcher.java:129)
    at com.hulu.behavior.rawfetcher.fetcherrunner.ConcurrentFetcherRunner$1.call(ConcurrentFetcherRunner.java:57)
    at com.hulu.behavior.rawfetcher.fetcherrunner.ConcurrentFetcherRunner$1.call(ConcurrentFetcherRunner.java:51)
    at java.util.concurrent.FutureTask.run(FutureTask.java:262)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
    at java.lang.Thread.run(Thread.java:745)
    Caused by: com.netflix.hystrix.exception.HystrixRuntimeException: BehaviorFetch.Binary.200 short-circuited and no fallback available.
    
    
从上述信息可以看到异常的类型是 SHORTCIRCUIT， 这个异常是 Hystrix 中的一种异常类型。  

Hystrix 是 Netflix 开源的一个外部依赖服务的管理工具。它会监控所有到某个外部服务的请求，然后对一段时间内的请求状态进行统计，之后根据统计信息进行一些管理和操作。比如发现请求的失败率已经超多 50% （可配置） ， 则断定该外部服务处于极度繁忙或不健康的状态，会将所有的请求该服务的 Request 进行短路处理，即不将请求发送到该服务，而是返回一个默认结果。在一段时间之后开关会重新打开，并清空之前的统计状态，Request 会重新请求到相应的服务（如果统计信息继续不乐观，则进行又一次短路操作）。采用这样的方式会减轻本来已经处于繁忙或者不健康状态的外部服务的负担。 

上面看到的 SHORTCIRCUIT 异常就是 hytrix 断定外部服务处于不健康的状态（即之前一段时间有超过 50% 的请求都失败了）， 然后直接对请求该服务的 Request 进行了短路处理。 我们去该外部服务的 Metric 发现响应时间并没有什么异常，同时我们注意到不是所有的机器都有问题，有些机器并没有相关的异常，并且一切都处于正常的状态。也就是说只有部分的机器出现了问题，这时候我通过 Metric 找出了出问题的机器，有 6（总共 25 台） 台机器处于不正常的状态， 这个时候可以断定是我们的 Service 出了某种问题，接下来开始相应的处理： 

* Dump 出问题机器的线程 （利用 jstack）。
* 重启出问题的 service，并重 lb 上拿下来 2 台机器方便进行分析。

利用 Hytrix 的 Web Tool 发现某个问题 service 上， 针对改外部依赖的线程池中的所有或者绝大部分线程都处于 Active 状态（总共 40 个线程）， 而其他正常 Service 上对应的线程池基本只有 1–2 个线程处于 Active 状态。 Hytrix 采用了线程池的模式，每一个 request 请求任务都会发送到一个线程池进行处理。如果请求的时候线程池中的所有线程都处于 Active 状态的话，则任务会被直接拒绝掉，而不是进入队列进行等待， 在 logstash 中我们也发现了大量的 REJECTED_THREAD_EXECUTION 类型的异常， 具体异常信息如下： 

    
    com.hulu.behavior.exception.UserBehaviorFetcherException: Fetch user behavior failed
    exceptionType: UserBehaviorFetchException errorLevel: NOTIFY
    requestUrl: "$XXXXX" HystrixExceptionType: REJECTED_THREAD_EXECUTION
    at com.hulu.behavior.rawfetcher.SingleRawUserBehaviorFetcher.getUserBehaviorRawData(SingleRawUserBehaviorFetcher.java:68)
    at com.hulu.behavior.rawfetcher.SingleRawUserBehaviorFetcher.getRawBehaviorData(SingleRawUserBehaviorFetcher.java:129)
    at com.hulu.behavior.rawfetcher.fetcherrunner.ConcurrentFetcherRunner$1.call(ConcurrentFetcherRunner.java:57)
    at com.hulu.behavior.rawfetcher.fetcherrunner.ConcurrentFetcherRunner$1.call(ConcurrentFetcherRunner.java:51)
    at java.util.concurrent.FutureTask.run(FutureTask.java:262)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
    at java.lang.Thread.run(Thread.java:745)
    Caused by: com.netflix.hystrix.exception.HystrixRuntimeException: BehaviorFetch.Binary.100 could not be queued for execution and no fallback available.
    
问题更进一步，现在可以肯定的是 Hytrix 的线程池处于不正常的状态，由于某种原因被 hang 住了，同时我们利用之前 dump 出来的堆栈信息发现，所有属于该线程池的线程都在 hang 在了同一个地方 : java.net.SocketInputStream.socketRead0 。具体的堆栈信息如下：

    
    "hystrix-UserBehaviorFetch-32" daemon prio=10 tid=0x00007f5e2403d800 nid=0x2636 runnable [0x00007f5df60de000]
    java.lang.Thread.State: RUNNABLE at java.net.SocketInputStream.socketRead0(Native Method) at java.net.SocketInputStream.read(SocketInputStream.java:152)
    at java.net.SocketInputStream.read(SocketInputStream.java:122)
    at java.io.BufferedInputStream.fill(BufferedInputStream.java:235)
    at java.io.BufferedInputStream.read1(BufferedInputStream.java:275)
    at java.io.BufferedInputStream.read(BufferedInputStream.java:334) - locked <0x000000078958d468> (a java.io.BufferedInputStream)
    at sun.net.www.http.ChunkedInputStream.fastRead(ChunkedInputStream.java:244)
    at sun.net.www.http.ChunkedInputStream.read(ChunkedInputStream.java:689) - locked <0x000000078958d490> (a sun.net.www.http.ChunkedInputStream)
    at java.io.FilterInputStream.read(FilterInputStream.java:133)
    at sun.net.www.protocol.http.HttpURLConnection$HttpInputStream.read(HttpURLConnection.java:3053)
    
从以上信息可以看出是因为 HTTP 请求的过程中 hang 在了 socketRead0， 但是为什么会 hang 在这里呢？ Timeout 没有起作用吗? 

这里就涉及到 Timeout 的设置问题，我们确实有设置相应的 Timeout （值为: 200ms）， 这个 Timeout 是 Hystrix 的一项配置。 查看代码我们发现了该 Timeout 的工作原理，Hystrix 采用线程池模式，超时机制是由向线程池提交任务之后的 Future 来实现，即 future.get(timeout)。在超时之后 future.get 会返回， 但是这个时候的线程并没有停止，它还在继续执行着它的任务。 对应到我们的场景， 在超时之后线程仍然在进行 HTTP 请求。 

为什么所有的线程都 hang 在了 socketRead0呢？ 之前提到改机房刚刚经历了一次网络问题， 在网络问题发生的时候机器可能正在发出 HTTP 请求， 但是由于网络的问题， 这些线程没有收到任何回复（成功或失败都没收到），这可能是导致这些线程 hang 的原因。 由于无法重现当时的情形， 所以这些也就基于上述现象和当时情形的一个猜测。还有一个问题就是 Socket Read 应该有默认的超时设置， 但是经过简单的测试，发现在 HTTP 请求到一个迟迟不返回的 API 的时候，HTTP 请求的线程确实会被一直 hang 住（有待进一步确认）。

#### 解决

我们需要解决的问题就是消除 Threadpool 中线程的 hang 的情况， 这里我们可以设置 socket.connect_timeout 和 socket.read_timeout 来尽量避免 socket 挂住的问题。但是这样并不能完全解决这个问题， 据 Google 到的信息来看， 就算设置了 Timeout 还是会出现 socket 挂住的情况， 最好的方式就是将所有对外部依赖的请求放到一个单独的线程去做，然后在超时之后直接终止掉该线程。但是采用这样的方式可能需要频繁的创建和杀死线程，特别是在出现超时情况比较多的时候。所以我们暂时也不打算采用这种方式，而是先设置了 socket 相关的 timeout，以期望减少类似情况的发生。相信机房网络出问题的情况不会太多，而且以后在机房出现网络状况之后，我们需要仔细检查 service 有没有受到网络的影响而进入一些异常的状态，如果有能够需要及时地发现和解决。

---

#### 总结
* 在机房出现了问题之后（例如：网络问题）一定要仔细查看 service 的状态， 确保 service 仍然处于健康的状态。

* Timeout 的设置和网络相关的知识需要继续进一步加强。


