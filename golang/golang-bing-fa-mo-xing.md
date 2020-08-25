# Golang 并发模型

在介绍 golang 的并发调度模型之前，先来看下几个基础知识

### 进程和线程

首先按教科书上现代操作系统的定义：

* 进程是资源管理的最小单位
* 线程是程序执行（系统调度）的最小单位

简单来说就是进程提供计算资源，线程进行计算。从进程演化出线程，最主要的目的是更好的支持 SMP 及减少上下文切换开销

> SMP——Symmetric Multi-Processing \(SMP\)，对称多处理器结构 AMP——Asymmetric Multi-Processing \(AMP\) ，非对称多处理器结构

_SMP 最大的特点是：一个操作系统实例，多个 CPU，每个 CPU 结构相同，共享内存等系统资源。工作负载能够均匀的分配到所有处理器上_

进程这一概念的进入，对应用程序屏蔽了 CPU 调度、内存管理等硬件细节，极大的简化了上层复杂度。

现代操作系统概念中，一个进程至少需要一个线程作为它的指令执行体，进程管理系统资源（内存，文件等），线程被分配到具体 CPU 上执行。

#### 上下文切换

CPU 是通过时间片来分配执行资源的，当进程/线程时间片结束，或者因为 redis、mysql 等 IO 阻塞掉时，将会引发上下文切换

```text
             运行
         ^ /      \
        / /        \
       / / 时       \ IO
    调/ / 间         \ 等待
   度/ / 片           \
    / / 到             \
   / /                  \
  / v                    v
  就绪<-----------------阻塞
            IO结束
```

上下文切换的直接开销有：

* 切换页表全局目录
* 切换内核态堆栈
* 切换硬件上下文（主要是寄存器数据载入）
* 刷新 TLB\(页表缓存\)
* 系统调度器的代码执行

间接开销：

* 跨 CPU 调度时，L1、L2 缓存预热（cache 的代码、数据没用了需要穿透到内存读取）

> 操作系统中，通常 L1 L2 缓存是每个 CPU 一个的，L3 缓存由多个 CPU 共享 Cache 是一种又小又快的寄存器，用来弥补内存和 CPU 之间的速度差距

### Linux 进程和线程实现

不同操作系统对于进程线程的实现是有很大不同的。早期的 Linux 内核是没有线程概念的，它的最小调度单元是 Task。 Linux 2.x 版本提供了轻量级进程的支持。轻量级进程和进程一样，都有自己独立的 task\_struct 进程描述符，也都有自己的 pid，在操作系统调度器视角看，两者没有什么区别。 轻量级进程最大的区别是：可以共享同一内存地址空间、代码段、全局变量以及同一打开文件集（线程的特点）。两者都是通过调用 clone\(\) 创建的，区别在于创建轻量级进程时会通过设置一系列 clone\_flags 来规定其可以共享那些资源。而进程通常是资源隔离的。

#### 扩展：Nginx 多进程模型

Nginx 架构采用：多进程模型 + 异步非阻塞（IO 多路复用）事件处理 一般进程数会设置为与 CPU 核数一致（可以将进程与 CPU 核心绑定），好处：

* 避免了竞争 CPU 资源所带来的上下文切换开销
* 与多线程相比，进程间相互独立，资源相互不共享，无额外锁开销

异步非阻塞：利用 event 驱动方式使用 select, poll, epoll, kqueue 等系统调用来实现复用

工作进程在 accept\(\) 到一个 request 时，会进行一系列预处理工作，然后把 request 转发到下游服务器，并使用 epoll\(例\)注册事件\(文件描述符\)，接着就可以处理其他 request 了，等接收到 fd 就绪通知后再处理 response。

### Golang 调度模型

golang 的调度器通过使用和 CPU 核心 数量相等的线程（GOMAXPROCS），以减少线程频繁切换的开销，同时在每个线程上执行开销更小的 Goroutine 来提高执行效率。

#### GMP 模型

简化模型如下

```text
 +--------+    +--------+    +----------------------------+
 |        |    |        |    |       |       |       |     
 |    M   |  < |    P   |  < |   G   |   G   |   G   |     
 |        |    |        |    |       |       |       |     
 +--------+    +--------+    +----------------------------+
```

* M: 操作系统的线程\(max M number to 10000\)
* P: 本地处理器，运行在线程上
* G: Goroutine

```text
// runtime.p
type p struct {
    m           muintptr

    runqhead uint32
    runqtail uint32
    runq     [256]guintptr
    runnext guintptr
    ...
}
```

处理器 P 本地队列是一个使用数组构成的环形链表，最多可以存储 256 个待执行任务。

完整模型如下：

```text
                                                global
   |   .   |     |   .   |     |   .   |        
   |   .   |     |   .   |     |   .   |      |       |      
   +-------+     +-------+     +-------+      +-------+  
   |       |     |       |     |       |      |       |  
   |   G   |     |   G   |     |   G   |      |   G   |  
   |       |     |       |     |       |      |       |     
   +-------+     +-------+     +-------+      +-------+  
   |       |     |       |     |       |      |       |  
   |   G   |     |   G   |     |   G   |      |   G   |        
   |       |     |       |     |       |      |       |        
   +-------+     +-------+     +-------+      +-------+        
       |             |             |          |       |
       v             v             v          |   G   |
   +-------+     +-------+     +-------+      |       |
   |       |     |       |     |       |      +-------+ 
   |   P   |     |   P   |     |   P   |      |       | 
   |       |     |       |     |       |       
   +-------+     +-------+     +-------+       

   +-------+     +-------+     +-------+ 
   |       |     |       |     |       | 
   |   M   |     |   M   |     |   M   | 
   |       |     |       |     |       | 
   +-------+     +-------+     +-------+ 

   +-----------------------------------+
   |              Sys&CPU              |
   +-----------------------------------+
```

模型中有两个运行队列：

1. P 的本地队列
2. 全局运行队列

`runtime.runqput` 函数会将新创建的 Goroutine 放入运行队列

* 当 runnext 空闲，就将 G 设置到 P 的 `runnext` 作为下一个处理的任务
* 当 runnext 不空闲并且 local 队列未满时，将 G 加入 P 的 local 队列
* 当 P 的 local 队列满了，就把 local 中的一部分 G\(待确认\) + 该 G 放入到 global 队列

`runtime.schedule` 函数为核心调度循环，它会从不同地方查找待执行的 G

* 首先一定几率从 global 中拿 G \(`_g_.m.p.ptr().schedtick%61 == 0`\)
* 其次从 local 中获取

[runtime/proc.go](https://github.com/golang/go/blob/64c22b70bf00e15615bb17c29f808b55bc339682/src/runtime/proc.go#L2504)

```text
if gp == nil {
    // Check the global runnable queue once in a while to ensure fairness.
    // Otherwise two goroutines can completely occupy the local runqueue
    // by constantly respawning each other.
    if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
        lock(&sched.lock)
        gp = globrunqget(_g_.m.p.ptr(), 1)
        unlock(&sched.lock)
    }
}
```

**Workflow** 引用自 [speakerdeck by Brandon Gao](https://speakerdeck.com/retervision/go-runtime-scheduler?slide=14)

```text
                        +--------------------sysmon---------------//----+
                        |                                               |
                        |                                               |
            +---+    +---+-------+               +--------+        +---+---+
gofunc()--->| G |--->| P | local |<===balance===>| global |<--//---| P | M |
            +---+    +---+-------+               +--------+        +---+---+
                       |                              |              |
                       |     +---+                    |              |
                       +---->| M |<---findrunnable----+---steal<--//-+
                             +---+
                               |
                               |
           +---execute<-----schedule
           |                   |
           |                   |
           +-->G.fn-->goexit---+

1.go creates a new goroutine
2.newly created goroutine being put into local or global queue
3.A M is being waken or created to execute goroutine
4.Schedule loop
5.Try its best toget a goroutine to execute
6.Clear, reenter schedule loop
```

#### 协作式调度和抢占式调度

golang 1.2 之前的早期调度器是不支持打断 or 抢占的，也就是说只能依靠 Goroutine 自己让出 CPU 资源才能触发调度。这会造成几个很大的问题：

1. 一些 Goroutine 可能会长时间占用 M，造成饥饿
2. 垃圾回收需要暂停整个程序（Stop-The-World），如果没有抢占手段，这将可能耗时很久，导致整体不可用

**golang 1.2 版本引入了基于协作式的调度**，其主要思路是：goroutine 在进行函数调用时会有机会进行运行时检查，来查看判断是否需要执行协作调度。 在实现上则是利用编译器在分段栈上插入函数 `runtime.morestack`，在 goroutine 发生函数调用时，会执行插入的函数 `runtime.morestack` 该函数会检查一个状态位来判断是否有协作调度。 其他的还有 golang 运行时会在垃圾回收暂停程序，以及系统监控发现有 goroutine 运行超过 10ms 时就会触发设置状态位来让该 G 让出 M。 但是这种协作式调度依然无法覆盖所有情况，比如 goroutine 在进行长时间 for 循环计算处理，并不进行函数调用，就不会触发协作状态检查。

**golang 1.14 版本引入了基于信号的抢占式调度**，不过当前直被应用在垃圾回收暂停任务时。其主要思路：利用系统信号 `SIGURG` 来触发异步抢占。 在实现上：\[runtime.sighandler\]\[3\]

1. 在 sighandler 函数中注册了 `SIGURG` 信号的处理函数 `doSigPreempt(gp, c)`
2. 当垃圾回收发生时，会做两件事：1. 将 `_Grunning` 状态的 G 标记为将被抢占；2. 向线程发送 `SIGURG` 信号
3. 操作系统会中断正在运行的线程，并执行注册的信号处理函数 `doSigPreempt`

### 其他

golang 运行时会根据需要动态调整函数栈大小，每个 goroutine 在创建时只会分配很小的栈空间（2KB, 4KB, 8KB 具体依赖实现）

**僵尸进程和孤儿进程**

* 僵尸进程：作为子进程已经执行完毕，但是在操作系统进程表中仍然存在未被回收。
* 孤儿进程：在父进程执行完成后，仍继续运行的进程被称为孤儿进程。

`僵尸进程` 的成因主要是：子进程在结束后其退出状态并未被读取（`wait`系统调用），它需要保留表项以允许父进程读取自己的状态。一旦状态被通过 `wait`系统调用读取，僵尸进程就会从进程表中删除。 僵尸进程可能的危害：

* 进程长时间保持僵尸状态一般都是错误导致的，并可能引起资源泄漏
* 比如父进程 for 循环不停的 clone 子进程，一直不使用 `wait` 系统调用，进程表中的信息将一直不被释放，可能会导致进程号耗尽
* 处理方式一般 kill 掉父进程，让 init 进程回收掉其状态就可以了

`孤儿进程` 在父进程结束后，会被挂到 init 进程下（收养），并由 init 进程来完成对其状态的收集工作。 孤儿进程的应用：

* 一般情况下孤儿进程是没什么危害的（除了个别情况下属于空跑，白白消耗 cpu 资源）
* `守护进程` 就是其一种应用，通过刻意制造孤儿进程使之与用户会话脱钩，转至后台运行。

参考：

* [https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/)
* [https://assets.ctfassets.net/oxjq45e8ilak/48lwQdnyDJr2O64KUsUB5V/5d8343da0119045c4b26eb65a83e786f/100545\_516729073\_DMITRII\_VIUKOV\_Go\_scheduler\_Implementing\_language\_with\_lightweight\_concurrency.pdf](https://assets.ctfassets.net/oxjq45e8ilak/48lwQdnyDJr2O64KUsUB5V/5d8343da0119045c4b26eb65a83e786f/100545_516729073_DMITRII_VIUKOV_Go_scheduler_Implementing_language_with_lightweight_concurrency.pdf)
* [https://zhuanlan.zhihu.com/p/79772089](https://zhuanlan.zhihu.com/p/79772089)
* [https://www.ibm.com/developerworks/cn/linux/kernel/l-thread/](https://www.ibm.com/developerworks/cn/linux/kernel/l-thread/)
* [https://www.timqi.com/2020/05/15/how-does-gmp-scheduler-work/](https://www.timqi.com/2020/05/15/how-does-gmp-scheduler-work/)

\[3\]: [https://github.com/golang/go/blob/62e53b79227dafc6afcd92240c89acb8c0e1dd56/src/runtime/signal\_unix.go\#L494](https://github.com/golang/go/blob/62e53b79227dafc6afcd92240c89acb8c0e1dd56/src/runtime/signal_unix.go#L494)

