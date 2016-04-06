---
layout: post
title: Java 并发编程学习 - 线程
category: 技术
tags: Java
keywords: Java, Concurrency, Thread
description:
---

### 1. 什么是线程？
什么线程？关于线程的 [维基百科](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=2&cad=rja&uact=8&ved=0ahUKEwiru_Dn1-PLAhXjfHIKHTTtDlAQFggkMAE&url=https%3A%2F%2Fzh.wikipedia.org%2Fzh%2F%25E7%25BA%25BF%25E7%25A8%258B&usg=AFQjCNGkqeFdYVYwmBQgsunfkJPT1eIm_A&sig2=uQBRmsAwfXWpK-lsOCYGaw) 解释如下：


> 线程是操作系统能够进行运算调度的最小单位。它被包含在进程之中，是进程中的实际运作单位。一条线程指的是进程中一个单一顺序的控制流，一个进程中可以并发多个线程，每条线程并行执行不同的任务。在Unix System V及SunOS中也被称为轻量进程，但轻量进程更多指内核线程，而把用户线程称为线程。 线程是独立调度和分派的基本单位。

如何理解线程是 CPU 调度的最小单位呢？简单来说一个 CPU 只能运行一个线程。线程由线程ID、程序计数器、寄存器集合和堆栈组成。线程自己不拥有系统资源，只拥有一点在运行中必不可少的资源，但它可与同属一个进程的其他线程共享进程所拥有的全部资源。


### 2. 线程 vs 进程？

1) 调度。在传统的操作系统中，拥有资源和独立调度的基本单位都是进程。在引入线程的操作系统中，线程是独立调度的基本单位，进程是资源拥有的基本单位。在同一进程中，线程的切换不会引起进程切换。在不同进程中进行线程切换,如从一个进程内的线程切换到另一个进程中的线程时，会引起进程切换。

2) 拥有资源。不论是传统操作系统还是设有线程的操作系统，进程都是拥有资源的基本单位，而线程不拥有系统资源（也有一点必不可少的资源），但线程可以访问其隶属进程的系统资源。

3) 并发性。在引入线程的操作系统中，不仅进程之间可以并发执行，而且多个线程之间也可以并发执行，从而使操作系统具有更好的并发性，提高了系统的吞吐量。

4) 系统开销。由于创建或撤销进程时，系统都要为之分配或回收资源，如内存空间、 I/O设备等，因此操作系统所付出的开销远大于创建或撤销线程时的开销。类似地，在进行进程切换时，涉及当前执行进程CPU环境的保存及新调度到进程CPU环境的设置，而线程切换时只需保存和设置少量寄存器内容，开销很小。此外，由于同一进程内的多个线程共享进程的地址空间，因此，这些线程之间的同步与通信非常容易实现，甚至无需操作系统的干预。

5) 地址空间和其他资源（如打开的文件）：进程的地址空间之间互相独立，同一进程的各线程间共享进程的资源，某进程内的线程对于其他进程不可见。


### 3. Java 线程
#### 3.1 线程启动
Java 中启动线程的方式主要有以下两种：
继承：
        
        class MyThread extends Thread {
            @Override
            public void run() {
                while(true) {
                    System.out.println("myThread");
                }
            }
        }
        new MyThread().start()
        
Runnable 接口：

		new Thread(new Runnable() {
            public void run() {
                while(true) {
                    System.out.println("myThread");
                }
            }
        }).start();
        
传入 Runnable 接口的方式相对简单一些，并且更加 OO （面向对象） 一些，所以通常推荐采用传入 Runnable 的方式。 
               
#### 3.2 线程方法

Thread 中包含很多成员方法和静态方法， 这里主要介绍下 Thread 的个人感觉比较重要的几个方法。

##### 3.2.1 成员方法 Thread.setName
这个方法本身比较简单， 但是对于调试和发现问题比较重要，很多时候我们都需要为线程取一个好的名字来标识它， 这样在调试问题的时候我们能够快速的定位到相关问题。 如下例：
	    
	    Thread thread = new Thread(new Runnable() {
            public void run() {
                while(true) {
                    System.out.println("hello world");
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                        Thread.currentThread().interrupt();
                    }
                }
            }
        });
        thread.start();
 使用 jstack 看到启动线程相关的信息如下：
 
 	"Thread-0" prio=5 tid=0x00007ff2e4053800 nid=0x5303 waiting on condition [0x0000000116f2f000]
   	java.lang.Thread.State: TIMED_WAITING (sleeping)
	at java.lang.Thread.sleep(Native Method)
	at ThreadTest$1.run(ThreadTest.java:26)
	at java.lang.Thread.run(Thread.java:745)

如果我们添加 thread.setName("my-hello-world-thread") 采用 jstack 看到的线程信息则如下：

	"my-hello-world-thread" prio=5 tid=0x00007fe4f9134800 nid=0x5303 waiting on condition [0x0000000122c42000]
   	java.lang.Thread.State: TIMED_WAITING (sleeping)
	at java.lang.Thread.sleep(Native Method)
	at ThreadTest$1.run(ThreadTest.java:26)
	at java.lang.Thread.run(Thread.java:745)
	
##### 3.2.2 成员方法 Thread.setDaemon

JVM 中存在两种线程：用户线程和守护线程。用户线程比较好理解，就是我们通常使用的线程都是用户线程。守护线程是指程序在运行的时候后台提供的一种通用服务线程，这些线程通常不是用户线程执行逻辑不可或缺的线程，比如和垃圾回收相关的一些线程就是守护线程，在 3.2.1 部分中例子，采用 jstack $pid, 我们还会看到类似的 stack 信息：

	"Finalizer" daemon prio=5 tid=0x00007fe4f980a000 nid=0x3503 in Object.wait() [0x00000001207e5000]
   	java.lang.Thread.State: WAITING (on object monitor)
	at java.lang.Object.wait(Native Method)
	- waiting on <0x00000007aaa84858> (a java.lang.ref.ReferenceQueue$Lock)
	at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:135)
	- locked <0x00000007aaa84858> (a java.lang.ref.ReferenceQueue$Lock)
	at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:151)
	at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:209)

这里的 Finalizer 就是一个守护线程。那么守护线程和用户线程的表现行为有哪些不同呢？ 最大的区别如果进程中存在用户线程，进程必须等待用户线程执行完成之后才能正常退出。但如果进程中只有守护线程了，那么程序能够直接完成正常退出。验证也比较简单，将 3.2.1 中例子运行起来之后， main 函数的逻辑已经执行完成，但是程序并没有退出。 如果我们在 thread start 之前添加 ```thread.setDaemon(true)``` 程序在 main 执行完成之后立马就退出了。 注意  ```thread.setDaemon(true)``` 只能添加在 thread.start() 之前，否则会抛出如下异常：

	java.lang.IllegalThreadStateException
		at java.lang.Thread.setDaemon(Thread.java:1388)
		
因为我们不能将已经在运行的线程设置为守护线程。 另外还有一个有意思的问题就是如果在守护线程里面启动的线程是什么线程呢？ （读者可以自行验证）

##### 3.2.3 静态方法 Thread.holdsLock
holdsLock 可以检测当前线程是否持有某个特定对象的监视器锁， 程序可以通过该方法实现如下断言， 有处于程序发现和调试问题：

	assert Thread.holdsLock(obj);
	
[注意]： holdsLock 只针对 synchronized 获取的监视器锁起作用， 对于 concurrent 包中的 Lock 并不适用。

##### 3.2.3 静态方法 Thread.sleep

顾名思义就是让当前线程进入 sleep 状态，让出 CPU 在一定时候之后再被唤醒执行，但是 sleep 能够达到多大的精确度呢？ 看下 sleep 的定义：

    public static native void sleep(long millis) throws InterruptedException;
    public static void sleep(long millis, int nanos) throws InterruptedException

第二个接口的代码如下：

    public static void sleep(long millis, int nanos)
    throws InterruptedException {
        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (nanos < 0 || nanos > 999999) {
            throw new IllegalArgumentException(
                                "nanosecond timeout value out of range");
        }

        if (nanos >= 500000 || (nanos != 0 && millis == 0)) {
            millis++;
        }

        sleep(millis);
    }

从代码中可以看出虽然支持传入纳秒，但是似乎只是做了简单的一个四舍五入， 所以从函数定义上看似乎 sleep 支持到毫秒级别的精度，但是事实究竟如何呢？我们看如下例子:

	    for (int i = 0; i < 10; i++) {
            long start = System.currentTimeMillis();
            try {
                Thread.sleep(10 * 1000); // sleep 10 seconds
            } catch (InterruptedException e) {
                // pass
            }
            System.out.println("sleep " + (System.currentTimeMillis() - start));
        }
        
 在我本机测试结果如下：
 	
 	sleep 10000
	sleep 10002
	sleep 10004
	sleep 10005
	sleep 10002
	sleep 10001
	sleep 10005
	sleep 10002
	sleep 10001
	sleep 10004

可见 sleep 并不保证毫秒级别的监控性，事实上 Thread.sleep 并不直接做精确度的保证， 其精确度依赖其底层的操作系统。 关于 sleep 的详细介绍可以 [点击链接](http://docs.oracle.com/javase/tutorial/essential/concurrency/sleep.html)。

##### 3.2.4 成员方法 Thread.interupt

从名字上看这个方法为线程的中断方法，但是它是否能正在中断一个正在运行线程呢？首先我们来看看这个方法的作用，然后回过头来解答这个问题。 interrupt 方法的作用主要分为如下几种情况：

1. 线程 block 在  Object#wait, Thread#join, Thread#sleep 方法的时， 当前线程的 interrupted status 被清除的同时，收到 InterruptedException 异常。
2. 线程 block 在 I/O 操作 java.nio.channels.InterruptibleChannel 上，channel 会被关闭，当前线程的 interrupted status 被设置，同时收到 java.nio.channels.ClosedByInterruptException 异常。
3. 线程 block 在 java.nio.channels.Selector 上， 当前线程的 interrupted status 被设置，并立即返回 select 操作。
4. 其他情况下当前线程的 interrupted status 被设置。

所以 interupt 到底能否终止一个线程呢？ 答案是视情况而定，例如当前线程 block 在 sleep 上：

	    Thread thread = new Thread(new Runnable() {
            public void run() {
                while (true) {
                    System.out.println("hello world");
                    try {
                        Thread.sleep(5 * 1000);
                    } catch (InterruptedException e) {
                        break;
                    }
                }
            }
        });

        thread.start();
        try {
            Thread.sleep(1 * 1000);
        } catch (InterruptedException e) {
            // pass
        }
        thread.interrupt();

这时候程序捕捉到 InterruptedException 并执行了 break 就可以退出 ，如果把这里的 break 注释掉，那么该线程并不会退出。那么杀死或者退出线程的正确姿势是什么呢？ 3.3 线程控制部分会讨论这个问题。

#### 3.3 线程控制
##### 3.3.1 线程生命周期

线程的生命周期包含 “新建（New）”、“就绪（Runnable）”、“运行（Running'）”、“阻塞（Blocked）”和“死亡（Dead）”五种状态。下图是各个状态只之间的转化关系： 

![](/public/img/posts/thread_cycle.jpg)

* 新建 New: 使用 new 关键字创建线程对象之后，该线程处于新建状态。
* 就绪 Runnable: 调用 start 之后，线程处于 Runnable 状态。
* 运行 Running: 如果处于就绪状态的线程获取到 CPU， 开始执行 Run 方法， 就处于运行状体了。当分配的 CPU 时间片用完后，线程变又处于就绪状态了。
* 阻塞 Blocked: 线程处于阻塞状态，一般有如下几种情况： 1. 调用 sleep 2. 调用阻塞 IO 操作 3. 等待监视器锁 4. 等待 notify 通知 5. 调用 suspend 方法（容易造成死锁， 慎用）
* 死亡 Dead: 一般当线程执行完 run 方法之后线程变进入死亡状态，除此以外线程抛出为捕获的异常，或者之间调用 stop 方法都会使得线程进入死亡状态。

##### 3.2.2 怎样停止一个线程

当我们想要停止或者取消一个线程正在执行的任务的时候，我们应该怎么做呢？ 从 Thread 提供的接口看来， 有两个看上去可以完成该功能的防范： stop 和 interrupt。

###### 1. stop


stop 的作用是强制停止一个正在运行的线程， 当时该函数已经不推荐使用了，这是为什么呢？在回答这个问题之前，首先我们来看一下调用 stop 停止一个线程的实现原理。先来看如下一个例子：

        Thread thread = new Thread(new Runnable() {
            public void run() {
                try {
                    while (true) {
                        int a = 1 + 1;
                    }
                } catch (Error error) {
                    error.printStackTrace();
                }
            }
        });

        thread.start();
        try {
            Thread.sleep(1 * 1000);
        } catch (InterruptedException e) {
            // pass
        }
        thread.stop();
        
程序的输出为：

	java.lang.ThreadDeath
		at java.lang.Thread.stop(Thread.java:836)
		at ThreadTest.main(ThreadTest.java:48)
		at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
		at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)
		at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
		at java.lang.reflect.Method.invoke(Method.java:606)
			at com.intellij.rt.execution.application.AppMain.main(AppMain.java:140)

从上述例子可以看出 stop 是通过抛出 java.lang.ThreadDeath 错误来结束线程执行的。

仔细看上面的例子可以看出我们认为从来不会抛出异常的 1 + 1 操作，居然会抛出异常，这会造成一个问题就是调用 stop 之后程序的行为变得不可预期，会破坏很多数据上的一致或者对象状态的不一致， 举一个简单的例子，a 和 b 各有 5 块钱， 两个人总共有 10 块钱，线程的作用是 a 拿出 2 块钱给 b， 这样 a 手中的钱变成 3 块， b 变为 7 块，如果在 a -= 3 和 b += 3 之间抛出异常，那么钱会凭空少了 3 块，所以调用 stop 会破坏数据的一致性。 

另外 stop 在调用之后会解锁所有的监视器锁，并不会因为 stop 导致锁不释放而死锁的问题。

关于为什么 stop 不推荐使用的详情可以参考：[Why stop, suspend, & resume of Thread are Deprecated](http://geekexplains.blogspot.com/2008/07/why-stop-suspend-resume-of-thread-are.html)

###### 2. interrupt

interrupt 在 [3.2.4 成员方法 Thread.interupt] 部分已经介绍了一部分，interrupt 其实是 java 引入的一种停止线程的**协商机制**。 “协商” 体现在 Thread 中可以选择合适的时机对 interupt 做出响应或者不作响应，这样用户可以自己保证数据的一致性前提下优雅的结束一个线程。 如上文所述一般情况下 interupt 只是设置线程的 interrupted status，我们在合适的时间判断 interrupted flag 做出响应即可。如下例：

	    Thread thread = new Thread(new Runnable() {
            public void run() {
                for (int i = 0; i < 100000; i++) {
                    if (Thread.interrupted()) break;
                    System.out.println("hello world");
                }
            }
        });

        thread.start();
        try {
            Thread.sleep(1 * 1000);
        } catch (InterruptedException e) {
            // pass
        }
        thread.interrupt();

处理中断的时机决定着程序的效率与中断响应的灵敏性。频繁的检查中断状态可能会使程序执行效率下降，相反，检查的较少可能使中断请求得不到及时响应。如果发出中断请求之后，被中断的线程继续执行一段时间不会给系统带来灾难，那么就可以将中断处理放到方便检查中断，同时又能从一定程度上保证响应灵敏度的地方。当程序的性能指标比较关键时，可能需要建立一个测试模型来分析最佳的中断检测点，以平衡性能和响应灵敏性。

InterruptedException 怎么处理比较好呢？ 从上文中我们可以看到 Thread 在 sleep 的时候如果执行 interrupt 操作，会使得 Thread 清空 interruptted status，同时抛出 InterruptedException 异常。 在代码中我们应该怎样处理 InterruptedException 异常呢？ 如果处理异常的地方能够明确的知道线程的所有行为，并且线程拥有线程管理权，那么可以自行处理。 但是一般情况下在处理 InterruptedException 的地方我们并不具备相应的条件，比如提交给线程池的 task :

	    Executor executor = Executors.newFixedThreadPool(2);

        executor.execute(new Runnable() {
            public void run() {
                System.out.print("Hello world");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    // Log it
                    Thread.currentThread().interrupt();
                }
            }
        });

这种情况下，在收到中断请求之后，我们并不能决定是否应该停止该线程，换句话说我们并不能对中断做出直接的处理。这个时候我们一般有两种选择：
	
1. 继续向上层抛出 InterruptedException，有上层代码进行处理。 但是很多时候接口上并没有定义该种异常，我们可以选择方案 2。
2. 执行 Thread.currentThread().interrupt() 重新设置线程的 interrupted status， 留给其他地方处理中断的机会。 

#### 3.4 线程同步和通信

这部分内容较多， 可以单独成篇。 (链接待补充)

#### 参考

[1] http://www.infoq.com/cn/articles/java-interrupt-mechanism

[2] http://geekexplains.blogspot.com/2008/07/why-stop-suspend-resume-of-thread-are.html

