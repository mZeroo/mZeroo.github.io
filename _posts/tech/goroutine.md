Go paper:  

[Communicating Sequential Processes](http://spinroot.com/courses/summer/Papers/hoare_1978.pdf)  

[Analysis of the Go runtime scheduler](http://www.cs.columbia.edu/~aho/cs6998/reports/12-12-11_DeshpandeSponslerWeiss_GO.pdf)

[Go Preemptive Scheduler Design Doc](https://docs.google.com/document/d/1ETuA2IOmnaQ4j81AtTGT40Y4_Jr6_IDASEKg0t0dBR8/edit?pref=2&pli=1)

[Scheduling Multithreaded Computations by Work Stealing](http://supertech.csail.mit.edu/papers/steal.pdf)

[The Go scheduler](https://morsmachine.dk/go-scheduler)

[Go Preemptive Scheduler Design Doc](https://docs.google.com/document/d/1ETuA2IOmnaQ4j81AtTGT40Y4_Jr6_IDASEKg0t0dBR8/edit?pref=2&pli=1)

[Go程序调试、分析与优化](http://tonybai.com/2015/08/25/go-debugging-profiling-optimization/)


摘要：
Go runtime 已经最近修改的想法都是来源于之前的提高并发度和性能的经验。 本文主要设计之前研究的几个具体实例，有一些已经应用于现有的 go runtime， 并产生了积极的作用。我们提出的针对 runtime 的额外扩展是基于竞争敏感的调度技术。同时本文也讨论了那些修改不仅利用了目前工作中已经提出的优化，并探讨它们如何能够提高 runtime 调度算法的效率。

介绍：
Go 采用了 SCP 模型。采用轻量级的 goroutines  支持并发。 goroutines 之间采用 channel （本质上采用有锁消息队列） 进行通信。本文的重点主要涉及的是 go 的 runtime 调度器。 
第二部分：go 语言简史
第三部分：runtime 调度器的实现
第四部分：runtime 调度器实现的缺陷
第五部分：修复 runtime 缺陷的建议
第六部分：go runtime 可以借鉴的几篇论文
第七部分：继续讨论提高 go runtime 好的想法
第八部分：提出 go runtime 演进的想法
第九部分：总结

** 2. Go 语言简史 

Go 语言大量采用了 CSP 论文中的原语，例如 Goroutines， channal， select 。 CSP 描述了许多共有的计算机科学和逻辑中的问题，以及如何采用 communicating processes 模型来解决的方法。

Newsqueak 是贝尔实验室开发的的基于 CSP 模型的程序语言，它对 Go 的设计有重要的影响。Rob Pike 参与过几种类似语言的开发工作，Newsqueak 是第一个将 channel 作为一等公民的语言。

** 3. Go Runtime
Go runtime 的主要功能：1. 调度 2. 垃圾回收 3. goroutines 运行环境。本文主要探讨的是调度部分，在此之前对于 runtime 需要有一个基本的认识。首先我们将要介绍什么是 runtime，特别是它如何将底层操作系统和 Go 语言代码关联起来的。

Go 程序降本 Go 编译器编译为机器码。Go 提供了 goroutines, channels 和 垃圾回收，runtime 需要对这些进行支持。因为 runtime 是 C 语言代码，在连接阶段被静态链接到用户的程序， 所以 Go 程序是一个独立可运行的用户程序。然而出于本文的目的，我们将一个 Go 程序分为两层： 用户编写的代码和 runtime ，通过函数调用的接口来实现对 goroutines , channels 和其他高阶组件的管理。为了方面调度，用户代码的任何系统调用都会被 runtime 拦截。

调度是 runtime 中很重要的部分， runtime 监控每一个 goroutine 并调度它们在线程池上执行。goroutines 和 线程是逻辑上分离的，但是 goroutines 需要在线程中执行。如果高效在 goroutines 在线程上执行决定 go 语言的性能。goroutines 背后的重要思想是它们可以像线程一样并行执行，它们相对于线程要轻量级得多。在 go 程序中可能创建了几个线程， 但是却创建了很多 goroutines， goroutines 的数量可能是线程数会数倍。多线程最大的并行度决定于 GOMAXPROCS？

记住 OS 看不到 goroutines， 它能够看到是的线程。至于 goroutines 在线程上的调度由 runtime 完成，对 OS 并不可见。

在 runtime 中有三个主要的 C 结构体负责跟踪用于完成 runtime 的调度：

G：代表一个 goroutine， 主要用于保存 goroutine 的 stack 和当前的状态。另外它还保存着指向代码的指针。

M：代表一个 用户线程。包含 G 的全局队列指针，当前正在运行的 G 的指针，自身的 Cache 和 调度器的句柄等。

SCHED：全局唯一。 包含指向两个 G 队列和 一个 M 队列指针，其他调度器裕运行需要的信息，比如全局 Sched 锁。G 的队列氛围两种：1 可运行的 G 队列。2 free list 的 G （blocking 中的 G 么？）。 M 的队列是空闲的 M 队列。如果需要修改这些队列都需要加一个全局 Sched 锁.

Runtime 在开始的时候会启动如下几个 G：
1. GC goroutine
2. 调度 goroutine
3. 用户主 goroutine

一般情况下在开始的时候只启动一个线程。 随着用户代码创建越来越多的 goroutines，会创建越来越多的线程。线程的数目上限由 GOMAXPROCS 决定。

由于 M 代表线程， 一般情况下一个 M 需要执行一个 goroutine。一个没有在运行 G 的 M 会从可运行的的 G 队列中拿出一个任务进行执行。如果运行着 G 的代码需要 M 阻塞，比如触发一个系统调用，这时在空闲 M 队列中的线程会被唤醒。这是为了保证 goroutines 在线程缺乏的情况下也能够继续运行。（所以此处陷入阻塞的线程会怎样？）

系统调用强制在运行的线程陷入内核，造成它在系统调用期间被阻塞。 如果 G 中的代码进行了阻塞系统调用， 那么运行这个 G 的 M 将会陷入阻塞，在系统调用返回之前，不能继续运行当前 G 的后续代码和其他的 G。值得注意的是 M 在通过 channel 进行通信时候不会陷入阻塞，虽然 goroutine 会阻塞。 OS 并不知道知道 channel 通信的存在，channel 通信完全由 runtime 实现。如果一个 G 调用一个 channel 的接口，它可能会阻塞，但是线程并没有任何阻塞的理由。在这种情况下 G 的状态会被设置为 waiting ， M 会继续处理下一个 G。 当通信完成时， 阻塞的 G 重新被置为 runnable 状态。

** 4 可提高的点
目前的 runtime 调度相对比较简单。 Go 语言相对比较年轻， 目前仅达到可以工作的状态(get job done)。 Dmitry Vyukov 在他的设计文档中针对目前的调度器提到四个主要的问题，同时也提供了改进的想法。

问题一：调度器依赖全局 Sched 锁。 当修改 M 和 G 的队列或者其他全局的 Sched 字段的都需要加锁。这造成 go 在处理大规模系统，特别是针对高吞吐服务，高并发计算程序扩展性有问题。

问题二：即使在 M 没有执行代码的时候，仍然会给给它跟配一个最大 2MB 的 cache， 这通常是没有必要的， 特别是当 M 空想的时候。如果空闲 M 的数量标的很大，这将会对对系统产生明显的影响。

问题三：对于系统调用的处理不够好，这造成了大量的阻塞和接触阻塞操作，浪费了 CPU。

问题四：目前的设计导致 M 将一个它执行的 G 给另外一个 M 执行，而不是它继续执行。这造成了很多额外开销和延迟。

** 5 改进


Vyukov 提出再创建一个抽象层 P， P 相当于对处理器的抽象。M 还是代表一个线程， G 还是代表一个 goroutine。 一个 go 程序的进程中包含 GOMAXPROCS 个 P。 P 是 M 执行 go 代码必要的一种资源。

P 的结构体包含的字段很多来自于之前的 M 和 Sched。比如 MCache 被移动到 P 中， 每一个 P 包含一个本地的可运行的 G 队列。这个本地 G 队列解决有助于之前的对于全局锁的依赖（问题一）。将 Cache 移动到 P 中减少了之前每个 M 都拥有 Cache 的带来浪费（问题二）。当一个 G 被创建的时候，它被放到 P 中 G 队列的队尾，调度器从 P 的本地队列获取 G 进行执行。此外，一个 P 的 G 队列可能为空，但是比如的 P 可能有很多 G 待执行。这就需要实现 stealing 算法， 当 P 的 G 队列为空的时候，任意从别人的 P 从拿到一半的 G 到当前的空闲 P 中。在搜索下一个待执行的 G 的时候， 如果 M 遇到了一个 G 绑定到一个空闲的 M 执行，它会唤醒这个空闲的 M， 并将相关的 G 和 P 转交给空闲的 M （？？没看懂）。

关于问题三 M 不断阻塞和给阻塞切换造成的开销。Vyukov 通过自旋的方式代替真正的阻塞的方式来解决。他提出了两种类型的自旋：

1. 拥有关联 P 的 M 在寻找一个新的 G 的时候自旋。
2. 未关联 P 的 M 在等待可用 P 的时候自旋。

当还有空闲的 M 没有关联的 P 存在的时候， 任何已经与 P 相关联的 空闲 M 都不能阻塞。有三种主要的时间 会造成 M 早暂时性的失去运行代码的能力。它们是当 G 产生的时候，M 进入系统调用呃时候， M 从空闲过度到繁忙状态的时候。在因为这些原因进入阻塞状态的之前，M 必须首先保证至少有一个自旋的 M， 除非所有的 P 都是繁忙状态。 这帮助解决了 M 不断地进行阻塞和非阻塞状态的切换，同时保证每一个 P 在有可运行的 G 的时候和一个 G 相关联。因此引入 M 自旋减少了系统调用造成的开销（but how？）。

Vyukov 同时建议如非必要，尽量不要分配 G。 他指出了用 6 个字来总结创建一个 goroutine： completion without making function calls or allocating
memory (??)。这会明显减少类似 goroutine 的内存开销。 另外一个建议是尽量提高 G 和 P 亲和力 （即尽量让同一个 G 运行在同一个 P 上）。

** 6 几篇论文

1. 协同调度

Scheduling Techniques for Concurrent Systems, written by
John K. Ousterhout in 1982。 介绍了系统调度的思想，在通信压力下的进程调度？ 这篇文章介绍了三种强制调度进程的不同算法。重点介绍一下即将应用于 go routine 的“矩阵算法”。

矩阵算法将进程放在一个数组里， 一个矩阵	