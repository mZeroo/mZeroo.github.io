---
layout: post
title: Node Addon 实战 (1) - 简介与入门
category: 技术
tags: c++, node
keywords: node addon, node-gyp 
description: 
---  



### 1. Node 及其线程模型

*关键词： Google V8, Event-driven, Non-blocking I/O Model, Callback Hell, Promise, CPU-intensive* 

要编写 Node Addon， 我们首先需要了解 Node：
 
>官方介绍: Node.js is a JavaScript runtime built on [Chrome's V8 JavaScript engine](https://developers.google.com/v8/). Node.js uses an event-driven, non-blocking I/O model that makes it lightweight and efficient. Node.js' package ecosystem, [npm](https://www.npmjs.com/), is the largest ecosystem of open source libraries in the world.

简单来说 Node 一个 javascript 语言的 runtime，本来在 chrome 中使用的， 后来大家发现其还可以用于服务端， 于是又打开了另一扇大门，Node 在 Web 服务端开始大行其道， 比较著名的框架有 [Express](http://expressjs.com/)。 Node 是基于事件和异步 IO 模型的，所以它能够高效的处理并发。 由于 Node 异步 IO 模型， 所以代码基本都是基于回调的， 此种思维方式有一些反人类正常思维， 编写相关代码的时候有些费劲， 会很容易陷入 [Callback Hell](http://callbackhell.com/)。解决 Callback Hell， 有两个比较有名的工具 [Q](https://github.com/kriskowal/q) 和 [Async](https://github.com/caolan/async) 。 （解决思路是基于 [Promise Model](https://en.wikipedia.org/wiki/Futures_and_promises) 的方式）  

Node 采用 [libuv](https://github.com/libuv/libuv) 来实现其基于事件的模型， 主要的功能等同老牌的 libevent。
libuv is a multi-platform support library with a focus on asynchronous I/O. It was primarily developed for use by [Node.js](https://nodejs.org/en/), but it's also used by [Luvit](https://luvit.io/), [Julia](http://julialang.org/), [pyuv](https://github.com/saghul/pyuv), and [others](https://github.com/libuv/libuv/wiki/Projects-that-use-libuv).
大家常说的 Node 是单线程的，并不是说 Node 进程只用了一个线程， 而是因为写的 js 代码只在一个线程中运行。其他阻塞， IO 操作等在其他线程中执行， 在任务完成以后再回调到用户主线程。Node 大概的线程模型如下图：

![Node 线程模型](/public/img/posts/node-thread-model.png)
 
由于用户只有一个线程，如果在用户主线程中执行 CPU 密集型的操作， 那就悲剧了，因为它阻塞其他部分的代码执行，这也是 Node 一直被人诟病的软肋。 如果有 CPU 密集型的操作可以通过写 addon 的方式进行处理，或者采用开源工具 [node-threads-a-gogo](https://github.com/xk/node-threads-a-gogo)。（解决思路是自己开新的线程执行相关的任务，然后采用 libuv 的方式回调）
以上只是对 Node 进行了一个简要的介绍， 不过可以发现提到了好多开源工具， 说明 Node 社区还是比较活跃的，工具也比较全。（在造轮子之前可以先看看有没有开源的方案）


### 2. Node Addon 简介

Node Addon 本质就是动态链接库，提供在 Node 中使用 c/c++ 代码和库的能力。如果要编写 addon，我们首先需要如下几个主要的类库的使用：

1. V8 JavaScript，C++类库，作为JavaScript的接口类，主要用于创建对象、调用方法等功能。大部分功能在头文件```v8.h```。（[见文档](https://developers.google.com/v8/?csw=1))
2. libuv 基于C的事件循环库，当需要等待的文件描述符可读时，等待定时器，或者等到接受信号时，会调用libuv的接口，也可以说，任何I/O操作，都需要调用libuv库.
3. 内部Node的库，可以通过node::ObjectWrap来调用Node.js内部的库。
4. 其他的一些类库同样可以在deps/ 中找到。

上面的介绍都有一些抽象，按照惯例我们还是通过一个简单的 hello world 的例子对 addon 有一个更为直观的认识。

#### 2.1 Hello World

以下例子针对 node 1.2+ 环境方可正常运行， node 1.1 之前采用完全不同的 API， 后文将对 node 版本的情况进行详细介绍。

##### 2.1.1 安装环境
* 安装 node 1.2+ 
* 安装 node-gyp

	node-gyp 是一个款平台的 node addon 构建工具，其安装和使用都非常简单（[详细文档](https://github.com/nodejs/node-gyp)）。
	1. 安装
	 	npm install -g node-gyp # 安装
	2. 使用
		添加 binding.gyp 配置文件， 运行 ```node-gyp configure``` 和 ```node-gyp build``` 即可。

##### 2.1.2 动手写一个 hello world

创建 binding.gyp,  并输入如下内容：

	
	    {
          "targets": [
            {
              "target_name": "hellowold",
              "sources": [ "helloworld.cc" ]
            }
          ]
        }

binding.gyp 为 node-gyp 的配置文件， node-gyp 为 node addon 的构建工具。

创建 helloworld.cc， 并输入如下内容：

		#include <node.h>
        
        using namespace v8;
        
        void HelloWorld(const FunctionCallbackInfo<Value>& args) {
            Isolate* isolate = args.GetIsolate();
            args.GetReturnValue().Set(String::NewFromUtf8(isolate, "hello world"));
        }
        
        void init(Local<Object> exports) {
            NODE_SET_METHOD(exports, "helloworld", HelloWorld);
        }
        
        NODE_MODULE(addon, init);

构建 addon：
	
	node-gyp configure
	node-gyp build

如果一切顺利的话，在当前目录下产生 build 文件夹， ```./build/Release/hellowold.node ``` 即为我们需要的文件， 该文件事实上就是一个动态链接库，我们可以用 ldd 看看其都依赖于哪些其他的 .so 文件：  


	[vagrant@vagrant-centos65 node-addon]$ ldd ./hello_world/build/Release/hellowold.node 
            linux-vdso.so.1 =>  (0x00007fff46fff000)
            libstdc++.so.6 => /usr/lib64/libstdc++.so.6 (0x00007f383864b000)
            libm.so.6 => /lib64/libm.so.6 (0x00007f38383c7000)
            libgcc_s.so.1 => /lib64/libgcc_s.so.1 (0x00007f38381b0000)
            libpthread.so.0 => /lib64/libpthread.so.0 (0x00007f3837f93000)
            libc.so.6 => /lib64/libc.so.6 (0x00007f3837bff000)
            /lib64/ld-linux-x86-64.so.2 (0x00007f3838b59000)


在 node 里面调用 helloworld 程序看一切是否正常运行， 创建  test.js 并输入如下内容： 

        var HelloWorld = require("./build/Release/hellowold.node")
        
        console.log(HelloWorld.helloworld())

```node test.js``` 输出 "hello world"， 说明一切 ok。

在上述例子用用到看一些 magic 的类，函数还有宏， 在接下来的文章都会一一详细展开。

### 参考资料

* [Node.js v5.1.0 Documentation](https://nodejs.org/api/addons.html): Node 5.1 Addon 简单官方手册
* [Node.js Addon Examples](https://github.com/nodejs/node-addon-examples): 代码实例