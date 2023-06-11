# 异常

## 什么是异常

一般情况下，进程会通过exit函数退出，而如果进程是因为收到 unix 信号，或是被内核直接杀死而结束，就属于是异常情况，可认为是发生了异常。crash 一般是进程收到信号而异常退出。

## 异常分类

- **Mach异常**：是指最底层的内核级异常。用户态的开发者可以直接通过Mach API设置thread，task，host的异常端口，来捕获Mach异常。
- **Unix信号**：又称BSD 信号，如果开发者没有捕获Mach异常，则会被host层的方法ux_exception()将异常转换为对应的UNIX信号，并通过方法threadsignal()将信号投递到出错线程。可以通过方法signal(x, SignalHandler)来捕获signal。
- **NSException**：应用级异常，它是未被捕获的Objective-C异常，导致程序向自身发送了SIGABRT信号而崩溃，是app自己可控的，对于未捕获的Objective-C异常，是可以通过try catch来捕获的，或者通过NSSetUncaughtExceptionHandler()机制来捕获。
- **C++Exception**：C++ 语言层面的异常，与 NSException 类似。

异常信息包括异常类型(`exception type`)、异常子类型(`exception subtype`)、异常编码(`exception code`)、 异常描述(`exception note`)异常线程号(`Triggered by Thread`)和其它异常信息，`crashlog` 文件会给出以上几种或者全部异常字段。

- **Mach异常**：是指最底层的内核级异常。用户态的开发者可以直接通过Mach API设置thread，task，host的异常端口，来捕获Mach异常。
- **Unix信号**：又称BSD 信号，如果开发者没有捕获Mach异常，则会被host层的方法ux_exception()将异常转换为对应的UNIX信号，并通过方法threadsignal()将信号投递到出错线程。可以通过方法signal(x, SignalHandler)来捕获signal。
- **NSException**：应用级异常，它是未被捕获的Objective-C异常，导致程序向自身发送了SIGABRT信号而崩溃，是app自己可控的，对于未捕获的Objective-C异常，是可以通过try catch来捕获的，或者通过NSSetUncaughtExceptionHandler()机制来捕获。
- **C++Exception**：C++ 语言层面的异常，与 NSException 类似。

![](/Users/bytedance/Library/Application Support/typora-user-images/image-20210611184357921.png)

Crashlog 中常见的 Mach 异常有以下几种：

- 1 `EXC_BAD_EXCESS`： 该类异常通常是由于访问了不该访问的内存导致。
- 2 `EXC_CRASH`： app 进行了系统不支持的操作。
- 3 `EXC_BAD_INSTRUCTION`： 执行非法指令。
- 4 `EXC_ARITHMETIC`： 计算异常，比如除零操作。
- 5 `EXC_RESOURCE`： 该种异常并非一个真正的 crash，而是系统告诫进程正在使用的资源过多。 如果 subtype 类型为 wakeup， 就表示某个线程被唤醒次数过多。如果 subtype 类型为 memory， 就表示超过了系统内存限制，可能会导致系统强制 Kill 进程。
- 6`EXC_GUARD`： 进程侵犯了被保护的资源。
- 7 `EXC_BREAKPOINT`:断点调试时异常退出。

Crashlog 中常见的 Unix 信号有以下几种：

- 1 `SIGSEGV`： 访问了未分配的内存或者往没有写权限的内存地址中写数据，比如重复释放对象， 多线程操作同一个对象。
- 2 `SIGABRT`： 收到了 abort 信号中止进程，比如调用了未实现的方法或者数组越界。
- 3 `SIGBUS`：总线访问错误，访问地址有效，但是地址未对齐等。
- 4 `SEGILL` ：执行了非法指令。
- 5 `SEGKILL`：立即结束进程运行的信号，比如内存不足被 Kill 掉。
- 6 `SEGTRAP`：由断点指令或者其他 trap 指令产生。
- 7 `SEGFPE`：计算错误，如浮点错误、溢出、除数为0等。
- 8 `SIGPIPE`：管道破裂。这个信号通常在进程间通信产生，比如采用FIFO(管道)通信的两个进程，读管道没打开或者意外终止就往管道写，写进程会收到SIGPIPE信号。

Unix 信号还有很多，此处仅列举 Crashlog 文件中常见的信号类型。

异常编码
异常类型对应的字段是 exception code，通过异常编码可以大致确认 crash 引发的原因，常见的异常编码有一下几种。

- ① `0x8badf00d`：程序被 watchdog 终止掉，比如程序启动或者恢复超时。
- ② `0xbad2222`：指 VoIP 应用(后台网络)在后台过于频繁的被激活。
- ③ `0xbaaaaaad`： 代表该 crash 并非一个真正的 crash，仅仅保存了应用某一个时刻的运行状态。
- ④ `0xdeadfa11`：程序无响应，用户强制 Kill。
- ⑤ `0xc00010ff`：程序执行大量耗费 CPU 和 GPU 运算，导致设备过热，被系统中止。
- ⑥ `0xdead10cc`：程序退至后台，仍然占用着系统资源。

# [异常对应表格](https://juejin.cn/post/6860022809646039053)

| Exception Type      | Signal  | 描述                                                         |
| ------------------- | ------- | ------------------------------------------------------------ |
| EXC_BAD_ACCESS      | SIGSEGV | 一般是非法内存访问错误，比如访问了已经释放的野指针，或者C数组越界 |
|                     | SIGBUS  | 非法地址，意味着指针所对应的地址是有效地址，但总线不能正常使用该指针。通常是未对齐的数据访问所致 |
| EXC_BAD_INSTRUCTION | SIGILL  | 非法指令。 是当一个进程尝试执行一个非法指令时发送给它的信号。可执行程序含有非法指令的原因，一般也就是cpu架构不对，编译时指定的march和实际执行的机器的march不同。这种情况，因为工具链一样，连接脚 本一样，所以可执行程序可以执行，不会发生exec format error。但是会包含一些不兼容的指令。还有另外一种可能，就是程序的执行权限不够，比如在用户态下运行的程序只能执行非特权指令，一旦CPU遇到特权指 令，将产生illegal instruction错误 |
| EXC_ARITHMETIC      | SIGFPE  | 当一个进程执行了一个错误的算术操作时发送给它的信号           |
| EXC_EMULATION       | SIGEMT  | 一个实现定义的硬件故障                                       |
| EXC_SOFTWARE        | SIGSYS  | 指示一个无效的系统调用。由于某种未知原因，进程执行了一条系统调用指令， 但其指示系统调用类型的参数却是无效的 |
|                     | SIGPIPE | 管道破裂。这个信号通常在进程间通信产生，比如采用FIFO(管道)通信的两个进程，读管道没打开或者意外终止就往管道写，写进程会收到SIGPIPE信号。此外用Socket通信的两个进程，写进程在写Socket的时候，读进程已经终止 |
|                     | SIGABRT | 这种异常类型的崩溃最常见的原因是未捕获的Objective-C/ c++异常和对abort()的调用。 如果应用程序扩展 Extension启动所需时间过久(例如被看门狗 Watch Dog终止)，则将使用此异常类型终止应用程序扩展。如果一个扩展在启动时由于挂起而终止，那么生成的崩溃报告的异常子类型将是LAUNCH_HANG。因为扩展没有主函数，所以花在初始化上的任何时间都是在静态构造函数和扩展库中的+load方法中进行的。你应该尽可能多地推迟这项工作 |
|                     | SIGKILL | 在系统的请求下，进程被终止。查看终止原因字段以更好地理解终止的原因。 终止原因字段将包含后跟代码的名称空间。以下代码是针对watchOS的: 终止码0xc51bad01表示一个监视应用程序被终止了，因为它在执行后台任务时占用了太多的CPU时间。要解决这个问题，可以优化执行后台任务的代码以提高CPU效率，或者减少在后台运行应用程序时执行的工作量。 终止代码0xc51bad02表示一个监视应用程序被终止，因为它未能在分配的时间内完成一个后台任务。要解决这个问题，减少应用程序在后台运行时执行的工作量。 终止码0xc51bad03表示一个监视应用程序未能在分配的时间内完成一个后台任务，并且系统总体上非常繁忙，应用程序可能没有收到足够多的CPU时间来执行后台任务。虽然一个应用程序可以通过减少它在后台任务中执行的工作量来避免这个问题，但是0xc51bad03并不表示这个应用程序做错了什么。更有可能的是，由于整个系统的负载，该应用程序无法完成其工作。 |
| EXC_BREAKPOINT      | SIGTRAP | 跟踪陷阱，由断点指令或其它trap指令产生，由debugger使用。与异常退出类似，此异常旨在使附加的调试器有机会在进程执行的特定位置中断进程。您可以使用_builtin_trap()函数从您自己的代码中触发此异常。如果没有附加调试器，则终止进程并生成崩溃报告。swift的空指针或类型转换失败也会导致此信号 |

[[iOS] Crash来集合啦](https://www.jianshu.com/p/5b2bd04e4414)

[iOS 如何快速有效的定位App carsh以及防护](https://www.jianshu.com/p/71a1a3ae74af)
