---
Title: Seastar tutorial中文翻译
---

原文链接：https://github.com/scylladb/Seastar/blob/master/doc/tutorial.md

翻译中会有一些不准确的地方，有望谅解和指正。对于一些我个人认为英文术语更方便阅读的地方，对原文进行了保留，例如socket。

# 引言

我们在本篇文档中介绍的Seastar是一个用于在现代多核机器上实现高性能复杂服务端应用的C++库。

传统来说，针对服务端医用的库和框架主要分为2大阵营：一些专注于性能，另一些专注于处理复杂性。一些框架性能非常好，但是只能用来搭建简单的应用（例如，DPDK只允许应用独立地处理包 _(DPDK allows applications which process packets individually)_），而其他一些框架通过牺牲运行效率来实现搭建非常复杂的应用。Seastar是我们集两者之长的一次尝试：创造一个可以用于搭建复杂服务端应用并达到最优性能的库。

Syclla是Seastar最初的灵感和用例，它是一份重写的Apache Cassandra。虽然Cassandra是一个非常复杂的应用，我们仍然通过使用Seastar达到了10倍的吞吐量提升以及显著降低且平稳的延迟。

Seastar使用2个概念——__future__和__continuation__——提供了一个完整的异步编程框架。它们提供了对任何类型的异步事件的一种一致的表达和处理方法，包括但不限于网络I/O，磁盘I/O，以及各种事件的复杂组合。

因为在现代多核多socket机器上在核间共享数据会带来严重的性能损失_(steep penalties)_（原子指令，cache line bouncing和memory fences），Seastar程序采用了share-nothing模型，也就是说，内存被分给各个核，每个核只处理自己的那份内存中的数据，核间通信需要通过显示的消息传输完成（which itself happens using the SMP's shared memory hardware, of course）。

# 异步编程

用于某个网络协议的server，例如经典的HTTP（Web）或者SMTP（e-mail），自然需要处理并行问题：多个client并行地发送请求的时候，我们不能顺序地处理它们：一个请求可能，也常常会因为某个原因阻塞——一个满的TCP window（即一个很慢的连接），磁盘I/O，甚至client保持了inactive connection——server需要能够去处理其他的连接。

处理这样的并行连接们的最直接的方法，也是如Inetd, Apache Httpd和Sendmail这样的经典server所采用的，就是对于每个连接使用一个独立的进程。这个技术的性能随着时间的发展在不断提升：最初，每个新链接会产生_(spawn)_一个新进程；之后，通过维护进程池，将新连接分配给空闲进程来实现；最后，这些进程被线程替代。然而，这些想法背后的共同点在于：每个进程在每个时刻只处理一个连接。因此，server代码可以使用阻塞性的系统调用，例如像一个连接读写，或者从磁盘读取。如果这个进程被阻塞了，我们还有其余的进程用于处理其他连接。

一个进程（或一个线程）处理一个连接的这种实现server的方式被称为_异步编程_。因为代码是线程结构的，一行代码会在上一行结束后开始运行。例如，代码可能会从socket读取数据，parse请求，然后零碎的从磁盘读取，并把处理结果写回给socket。这样的代码很容易写，写起来就像是传统的非并行程序。事实上，运行一个外部的非并行的程序来处理各个请求也是可能的——例如Apache HTTPd就是用这种方法来运行的"CGI"程序，动态网页生成的首个实现。

> 注意：尽管同步程序是线性、非并行的方式写的，kernel在底层保证了并行的发生与资源的充分使用。在进程并行（多个进程并行处理多个连接）之外，kernel可能会去并行处理一个连接需要的工作——例如处理把一个磁盘请求（如读取磁盘文件）和处理网络连接（发送在缓存区但仍未发送的数据，以及缓存仍为被读取的新收到的数据）并行起来。

但是同步、每个进程一个连接的这种server模式不是没有缺点和成本的。慢慢地，server的作者们发现创建一个新进程是很慢的，context switch很慢，每次处理都会有显著的overhead——最明显的是对堆栈大小的要求。Server和内核的作者们非常努力地去缓解这些overhead：他们从进程切换至线程，从创建新线程切换至使用线程池，降低了每个线程的默认堆栈大小，以及提高了虚拟内存大小来partially-utilize stacks。但是仍旧，使用同步设计的server的性能不够理想，扩展性不佳。在1999年，Dan Kigel普及了"the C10K problem"，需要单个server能够高效处理10000个并发连接——其中大多数为慢甚至inactive的连接。

这个问题的解决方法，也是在后续的几十年中逐渐变得流行的方法，就是抛弃方便但是低效的同步设计，转为一种新型的设计——异步，或者说是事件驱动的server。一个事件驱动的server仅有一个线程，或者更准确的说，每个CPU一个线程。这个线程会运行一个循环，在循环的每个迭代步里，通过`poll()`（或者是更高效的`epoll`）来监察绑定在开启的file descriptor上的新事件，如sockets。举例来说，一个socket变得可读（有新数据抵达）或者变得可写（我们可以用这个连接发发送更多数据了）都是一个事件。应用通过进行一些非阻塞性的操作，修改一个或多个fd，后者维护这个连接的状态来处理这些事件。

然而，写异步server应用的人们在今天仍然会遇到2个主要问题。

- **复杂性**：写一个简单的异步server是很简单的。然而写复杂异步server的难度重名昭著。我们不再能用一些简单易读的函数调用来处理一个连接，而是需要引入大量回调函数，和一个复杂的状态机，用于记录对于那些事件应该调用那些函数。
- **非阻塞**：因为context switch很慢，所以一个核只有一个线程是对性能很重要的。然而，如果我们每个核只有一个线程，处理事件的函数就_永远_不能阻塞，不然这个核就会被闲置。例如，Cassandra是一个异步server应用，但是因为磁盘I/O使用了`mmap`，其可能会不受控制地阻塞整个线程，他们不得不在一个CPU上运行多个线程。

除此以外，当需求更高的性能的时候，server应用，以及其使用的框架，不得不考虑一下的2件事：

- **现代机器**：现代的机器和约10年前机器有着非常大的区别。他们有很多核和很深的内存层级（从L1 cache到NUMA），这种结构更适合特定的编程范式_(programming practice)_：不可扩展性的编程范式（如，加锁）可能会极大地影响程序在多核上的性能；共享内存和无锁同步primitives虽然是可用的（比如原子操作和memory-ordering fences），但是比只用一个核的cache中的数据进行操作要慢很多，并且也会让程序不好扩展至多核。

- **编程语言**：Java, Javascript这样的“现代”高级语言虽然很方便，但是他们各自依照了和上述要求相悖的假设。这些语言为了能够更加轻便_(portable)_，也让程序员对核心代码的性能有着更低的控制能力。如果我们需要足够优化的性能，我们需要使用让程序员可以有完全控制权、零运行时overhead的编程语言，在另一方面——有复杂的编译优化。

Seastar是一个旨在解决上述的4个问题的异步框架：它是一个用于实现同时包括磁盘和网络I/O的复杂server的框架。它的fast path完全是单线程的（每核），可扩展至多核并最小化了代价高昂的核间内存共享。Seastar是一个C++14的库，让用户能利用上复杂的编译优化特征与充分的控制力，且没有运行时overhead。

# Seastar

Seastar是一个让人可以比较直接地实现非阻塞、异步代码的事件驱动框架。它的API基于future。Seastar利用了如下的概念以获得极致的性能。

- Cooperative micro-task scheduler
- Share-nothing SMP架构
- 基于future的APIs
- **Share-nothing TCP stack**：Seastar可以直接使用本机操作系统的TCP stack。在此之外，它也提供了一套基于tash scheduler和share-nothing架构的高性能TCP/IIP stack。这套stack提供双向零拷贝：你可以直接用TCP stack的buffer处理数据，并在不进行拷贝的情况下发送你的数据结构。
- DMA-based存储APIs

本教程针对于熟悉C++的开发者，并会覆盖如何用Seastar创建一个新的应用程序。

TODO: copy text from https://github.com/scylladb/seastar/wiki/SMP https://github.com/scylladb/seastar/wiki/Networking

# Getting started

最简单的Seastar程序如下：

```c++
#include <seastar/core/app-template.hh>
#include <seastar/core/reactor.hh>
#include <iostream>

int main(int argc, char** argv) {
    seastar::app_template app;
    app.run(argc, argv, [] {
            std::cout << "Hello world\n";
            return seastar::make_ready_future<>();
    });
}
```

如我们在例子中所示，每个Seastar程序必须定义并运行一个`app_template`对象。这个对象会在1个或多个CPU上启动主事件循环_(event loop)_（the Seastar engine），并运行给定的函数一次——在本例中，是一个lambda函数。

`return make_ready_future<>();`会使事件循环以及整个程序在打印"Hello world"之后立即退出。在一个更典型的Seastar程序中，我们会希望事件循环持续运行，并处理收到的包，直到显式退出。这样的程序会返回一个用于判断何时退出的_future_。我们将在下文介绍future以及如何使用它。无论何时都**不要**使用常规的C `exit()`，因为其会阻止Seastar正确地在退出时进行清理工作。

如例子所示，所有Seastar的函数都处于"`seastar`" namespace中。我们建议使用namespace prefix而不是`using namespace seastar`，在之后的例子中也会如此。

在编译本程序之前，请确认您已下载、编译或可选地安装了Seastar，并将上述的程序放于某一源文件中，例如`getting-started.cc`。

如果Seastar被编译在了`$SEASTAR`目录下，您可以通过如下指令编译`getting-started.cc`：

```bash
c++ getting-started.cc `pkg-config --cflags --libs --static $SEASTAR/build/release/seastar.pc`
```

如果您安装了Seastar，`pkg-config`可以更短：

```bash
c++ getting-started.cc `pkg-config --cflags --libs --static seastar`
```

另外，您也可以使用如下`CMakeLists.txt`通过CMake编译该程序：

```cmake
cmake_minimum_required (VERSION 3.5)

project (SeastarExample)

find_package (Seastar REQUIRED)

add_executable (example
  getting-started.cc)

target_link_libraries (example
  PRIVATE Seastar::seastar)
```

在这种情况下，您可以用如下指令编译该例子：

```bash
$ mkdir build
$ cd build
$ cmake ..
$ make
```

这时程序应该可以通过如下方式正式运行：

```bash
$ ./example
Hello world
$
```

# 线程和内存

## Seastar 线程

正如在引言中提到的，基于Seastar的程序每个CPU上运行单一线程。每个线程有自己的事件循环，在Seastar的术语中被称为_engine_。默认情况下，Seastar程序会占据所有可用的核，每个核启动一个线程。我们可以通过如下程序来验证这点，其中`seastar::smp::count`是启动的线程总数：

```c++
#include <seastar/core/app-template.hh>
#include <seastar/core/reactor.hh>
#include <iostream>

int main(int argc, char** argv) {
    seastar::app_template app;
    app.run(argc, argv, [] {
            std::cout << seastar::smp::count << "\n";
            return seastar::make_ready_future<>();
    });
}
```

在一个4硬件线程的机器上（2核并开启超线程），Seastar默认会使用4个engine thread：

```bash
$ ./a.out
4
```

这4个engine thread会被绑定至（a la `taskset(1)`）不同的硬件线程。注意，如我们提到的，启动函数只在一个线程上运行，所以我们只看到"4"被打印了1次。之后的教程会告诉大家该如何使用所有的线程。

用户可以通过传入一个命令行参数`-c`来告诉Seastar去启动更少的线程数。例如，可以通过如下方式只启动2个线程：

```bash
$ ./a.out -c2
2
```

在这种情况下，Seastar会保证每个线程绑定在不同的核上，我们不会让这2个线程在同一个核上作为超线程相互争夺的（不然会影响性能）。

我们不能分配超过硬件线程总数的线程，这么做会报错：

```bash
$ ./a.out -c5
terminate called after throwing an instance of 'std::runtime_error'
  what():  insufficient processing units
abort (core dumped)
```

上面的程序来自于app.run的异常。为了能更好的catch这种启动异常并在不生成core dump的情况下优雅退出，我们可以这样写：

```c++
#include <seastar/core/app-template.hh>
#include <seastar/core/reactor.hh>
#include <iostream>
#include <stdexcept>

int main(int argc, char** argv) {
    seastar::app_template app;
    try {
        app.run(argc, argv, [] {
            std::cout << seastar::smp::count << "\n";
            return seastar::make_ready_future<>();
        });
    } catch(...) {
        std::cerr << "Failed to start application: "
                  << std::current_exception() << "\n";
        return 1;
    }
    return 0;
}
```

```bash
$ ./a.out -c5
Couldn't start application: std::runtime_error (insufficient processing units)
```

注意这样**不能**catch到程序实际的异步代码引发的异常。对于那些异常我们会在后文介绍。

## Seastar 内存

正如在引言中介绍的，Seastar程序会将他们的内存分片_(shard)_。每个线程会被预分配一大块内存（在运行这个线程的那个NUMA node上），并且只使用这块被分配的内存（例如在`malloc()`或`new`中）。

默认除去给OS保留的1.5G或7%的内存外的**全部内存**都会被通过这种方式分配给应用。这个默认值可以通过`--reserve_memory`来改变给系统剩余的内存，或者`-m`来改变给Seastar应用分配的内存来改变。内存值可以以字节为单位，或者用"k", "M", "G", "T"为单位。这些单位均遵从2的幂，即"M"为**mebibyte**, 2^20字节，而非`megabyte`（10^6 字节）。

试着给Seastar超过物理内存的内存值会直接异常退出：

```bash
$ ./a.out -m10T
Couldn't start application: std::runtime_error (insufficient physical memory)
```

# future和continuation简介

future和continuation是Seastar的异步编程模型的基石。通过组合它们可以轻松地组件大型、复杂的异步程序，并保证代码可读、易懂。

future是一个还不可用的计算结果。例如：

- 我们从网络读取的数据buffer
- 计时器的到时
- 磁盘写操作的完成
- 一个需要其他一个或多个future的值来进行的计算的值

`future<int>`类型承载了一个终将可用的int——现在可能已经可用，或者还不能。成员函数`available()`可以用来测试一个值是否已经可用，`get()`可以用来获取它的值。`future<>`类型表示一些终将完成的事件，但是不会返回任何值。

future往往是一个**异步函数**的返回值。异步函数是指一个返回future，并将会将这个future的值确定下来的函数。因为异步函数_保证_将确定future的值，有时他们被称作"promise"。但我们不采用这一称呼，因为其会导致的误解。

Seastar的`sleep()`函数就是一个简单的异步函数的例子：

```c++
future<> sleep(std::chrono::duration<Rep, Period> dur);
```

这个函数设置了一个计时器，从而使在经过指定时间之后future变得可用（并没有携带的值）。

**continuation**是一个在future变得可用时会运行的回调函数（常为lambda函数）。continuation会用`then()`方法附于一个future。例如：

```c++
#include <seastar/core/app-template.hh>
#include <seastar/core/sleep.hh>
#include <iostream>

int main(int argc, char** argv) {
    seastar::app_template app;
    app.run(argc, argv, [] {
        std::cout << "Sleeping... " << std::flush;
        using namespace std::chrono_literals;
        return seastar::sleep(1s).then([] {
            std::cout << "Done.\n";
        });
    });
}
```

在这个例子里，我们从`seastar::sleep(1s)`中获得了一个future，并把一个打印"Done."的continuation附于其上。1s后future会变得可用，这时continuation就会被执行。运行这个程序，我们的确会看到立即被打印的"Sleeping..."，和1s后打印的"Done."，之后程序退出。

`then()`的返回值也是一个future，从而使得链式的continuation变得可能，这点我们之后会提到。现在我们只需要注意我们把这个future作为了`app.run`的返回值，所以程序会在运行完sleep以及其continuation后才会退出。

为了避免在左右的例子里都重复"app_engine"部分的代码，让我们创建一个可以复用的模板：

```c++
#include <seastar/core/app-template.hh>
#include <seastar/util/log.hh>
#include <iostream>
#include <stdexcept>

extern seastar::future<> f();

int main(int argc, char** argv) {
    seastar::app_template app;
    try {
        app.run(argc, argv, f);
    } catch(...) {
        std::cerr << "Couldn't start application: "
                  << std::current_exception() << "\n";
        return 1;
    }
    return 0;
}
```

使用这个`main.cc`，上述的sleep例子变成了：

```c++
#include <seastar/core/sleep.hh>
#include <iostream>

seastar::future<> f() {
    std::cout << "Sleeping... " << std::flush;
    using namespace std::chrono_literals;
    return seastar::sleep(1s).then([] {
        std::cout << "Done.\n";
    });
}
```

至此，这个样例非常平凡——没有并行、我们用普通的POSIX `sleep()`也能做到。事情会在我们启动多个sleep的时候变得有趣。future和continuation让并行变得非常简单与自然：

```c++
#include <seastar/core/sleep.hh>
#include <iostream>

seastar::future<> f() {
    std::cout << "Sleeping... " << std::flush;
    using namespace std::chrono_literals;
    seastar::sleep(200ms).then([] { std::cout << "200ms " << std::flush; });
    seastar::sleep(100ms).then([] { std::cout << "100ms " << std::flush; });
    return seastar::sleep(1s).then([] { std::cout << "Done.\n"; });
}
```

每个`sleep()`和`then()`都会立即退出：`sleep()`仅仅启动计时器，而`then()`只是设置到到时的时候应该调用什么函数。所以3行代码都会马上被执行，f也会直接返回。在那之后，时间循环会开始等3个future变得可用，且每个可用的时候都会运行他们对应的continuation。上述代码的输出显然会是：

```bash
$ ./a.out
Sleeping... 100ms 200ms Done.
```

`sleep()`返回的是`future<>`，也就是在完成时不会返回任何值。更有趣的future们会在可用时返回一个（或多个）任意类型的值。在下面的例子里，我们有一个返回`future<int>`的函数，以及对应的continuation。注意continuation是如何得到future中包着的值的：

```c++
#include <seastar/core/sleep.hh>
#include <iostream>

seastar::future<int> slow() {
    using namespace std::chrono_literals;
    return seastar::sleep(100ms).then([] { return 3; });
}

seastar::future<> f() {
    return slow().then([] (int val) {
        std::cout << "Got " << val << "\n";
    });
}
```

我们需要再解释一下`slow()`。和往常一样，这个函数立即返回一个`future<int>`，不会等sleep完。但是其中continuation返回的是3，而非future。我们在下文介绍`then()`是怎么返回future的，以及这种机制是怎么让链式future变得可能的。

这个例子展示了future这种编程模型的便捷性。因为其让程序员可以很简洁地包装复杂的异步程序。`slow()`可能实际是调用了一个复杂的设计多步的异步操作，但是它可以和`sleep()`同样被使用，并且Seastar的engine会保证在正确的时间运行continuation。

## Ready futures

一个future可能在运行`then()`之前就已经准备好了。这种重要的情况被优化过，_往往_continuation会被直接运行，而不再用被注册，并等待下一个迭代步的event loop了。

在_大多数时候_都会进行这种优化，除了：`then()`的实现维护了一个会立刻执行的continuation的计数器，在超过一定量continuation被直接运行后（目前限制为256个），下一个continuation一定会被放入event loop。这种机制是因为我们发现在一些情况下（例如后文会讨论的future循环），我们会发现每个准备好的continuation会立刻生成一个新的，没有这种限制我们就会starve事件循环。让事件循环可以运行非常重要，不然就无法运行之后准备好的continuation了，也会starve事件循环会进行的重要的**polling**（例如，检查是否网卡有新的活动_(activity)_）。

`make_ready_future<>`可以被用来返回一个已经准备好了的future。下面的例子和之前的几乎完全相同，只是`fast()`会返回一个已经准备好了的future。所幸future的接受者不在乎这个区别，并会用相同的方式处理这两种情况：

```c++
#include <seastar/core/future.hh>
#include <iostream>

seastar::future<int> fast() {
    return seastar::make_ready_future<int>(3);
}

seastar::future<> f() {
    return fast().then([] (int val) {
        std::cout << "Got " << val << "\n";
    });
}
```

# Continuations

## 在continuation中捕获_(capture)_状态



## 关于求值顺序的一些考虑

C++14（及更低版本的C++）_不能_保证continuation中的lambda capture会在他们相关的future被计算之后才被求值（见 https://en.cppreference.com/w/cpp/language/eval_order）。

因此，请避免下述的模式：

```c++
    return do_something(obj).then([obj = std::move(obj)] () mutable {
        return do_something_else(std::move(obj));
    });
```

这个例子中`[obj = std::move(obj)]`可能会先于`do_something(obj)`，导致出现使用了被move出去之后的`obj`。

为了保证我们期望的顺序，上面的表达式可以拆分为：

```c++
    auto fut = do_something(obj);
    return fut.then([obj = std::move(obj)] () mutable {
        return do_something_else(std::move(obj));
    });
```

c++17后，`then`对应的对象会在`then`的参数之前被求值，使得`do_something(obj)`会保证先被执行。所以C++17中可以不采用上面的改正方法。

## 链式continuation

TODO: We already saw chaining example in slow() above. talk about the return from then, and returning a future and chaining more thens.

# 异常处理



## exceptions vs. exceptional futures



# 管理生命周期



## 把所属权传给continuation



## Keeping ownership at the caller



## Sharing ownership (reference counting)



## Saving objects on the stack



# 高级future

## fututre和中断

TODO: A future, e.g., sleep(10s) cannot be interrupted. So if we need to, the promise needs to have a mechanism to interrupt it. Mention pipe's close feature, semaphore stop feature, etc.

## future只能被用一次

TODO: Talk about if we have a `future` variable, as soon as we `get()` or `then()` it, it becomes invalid - we need to store the value somewhere else. Think if there's an alternative we can suggest

# Filbers

Seastar continuation大多很短，但是往往成链式，所以一个continuation做一些工作后就会调度之后的其他continuation。这样的链可能很长，并且常包含循环（见下一个章节“Loops”）。我们称这样的链为执行的一个"fiber"。

这些fiber并不是线程——每个都只是一串continuation——但是他们也有和传统线程一样的一些要求。例如，我们会避免一个fiber被starve，而另一个一直在运行。又如，fiber之间可能会进行通信——一个fiber生成另一个fiber使用的数据，我们希望能保证2个fiber都能运行，并且如果1个过早地停止了，另一个不要一直hang。

TODO: Mention fiber-related sections like loops, semaphores, gates, pipes, etc.

# Loops

大多数很消耗时间的计算都需要循环。Seastar提供了很多用future/promise的方式表示循环的primitives。有关Seastar的循环primitives一个非常重要的点是，每个迭代步的后面都会有一个抢占点_(preemption point)_，因而允许其他任务在迭代步之间运行。

## repeat

一个用`repeat`创建的循环会执行其body，然后收到一个`stop_iteration`对象。这个对象会告诉循环是该继续运行（`stop_iteration::no`）还是该停止（`stop_iteration::yes`），下一个迭代步会在前一个执行完再执行。被传给`repeat`的body需要返回`future<stop_iteration>`。

```c++
seastar::future<int> recompute_number(int number);

seastar::future<> push_until_100(seastar::lw_shared_ptr<std::vector<int>> queue, int element) {
    return seastar::repeat([queue, element] {
        if (queue->size() == 100) {
            return make_ready_future<stop_iteration>(stop_iteration::yes);
        }
        return recompute_number(element).then([queue] (int new_element) {
            queue->push_back(element);
            return stop_iteration::no;
        });
    });
}
```

## do_until

`do_until`和`repeat`非常接近，只是其会显示传入一个判断条件。下列代码展示了该如何使用`do_until`：

```c++
seastar::future<int> recompute_number(int number);

seastar::future<> push_until_100(seastar::lw_shared_ptr<std::vector<int>> queue, int element) {
    return seastar::do_until([queue] { return queue->size() == 100; }, [queue, element] {
        return recompute_number(element).then([queue] (int new_element) {
            queue->push_back(new_element);
        });
    });
}
```

注意循环body需要返回`future<>`，从而使我们能够早循环中创建复杂的continuation。

## do_for_each



## parallel_for_each



## max_concurrent_for_each



# when_all: 等待多个future

上面我们见到了`parallel_for_each()`，其会开启若干异步操作，并等待所有的都完成。Seastar有另一个函数`when_all()`，用于等待若干已经存在的future完成。

`when_all()`的第一种变体是一个变长函数，也就是future可以作为分隔的参数传入，有多少个future是在编译期就确定下来的。每个future可以类型不同。例如：

```c++
#include <seastar/core/sleep.hh>

future<> f() {
    using namespace std::chrono_literals;
    future<int> slow_two = sleep(2s).then([] { return 2; });
    return when_all(sleep(1s), std::move(slow_two), 
                    make_ready_future<double>(3.5)
           ).discard_result();
}
```





# Semaphores



## 用semaphores限制并行性



## 限制资源利用量



## 限制循环的并行



# Pipes



# Shutting down a service with a gate



# shared-nothing programming简介

TODO: Explain in more detail Seastar's shared-nothing approach where the entire memory is divided up-front to cores, malloc/free and pointers only work on one core.

TODO: Introduce our shared_ptr (and lw_shared_ptr) and sstring and say the standard ones use locked instructions which are unnecessary when we assume these objects (like all others) are for a single thread. Our futures and continuations do the same.

# 关于Seastar的事件循环的更多内容

TODO: Mention the event loop (scheduler). remind that continuations on the same thread do not run in parallel, so do not need locks, atomic variables, etc (different threads shouldn't access the same data - more on that below). continuations obviously must not use blocking operations, or they block the whole thread.

TODO: Talk about polling that we currently do, and how today even sleep() or waiting for incoming connections or whatever, takes 100% of all CPUs.

# Seastar网络栈简介

TODO: Mention the two modes of operation: Posix and native (i.e., take a L2 (Ethernet) interface (vhost or dpdk) and on top of it we built (in Seastar itself) an L3 interface (TCP/IP)).

为了最优的性能，Seastar的网络栈和Seastar的普通应用一样被分片了：每个分片（线程）会承担一部分连接。每个连接会被转给某一个线程，当一个连接被创建了，它就一直会由这个线程处理。

在我们之前例子里我们看到，`main()`只会在第一个线程运行`f()`一次。除非server是用`"-c1"`（只开启1个线程）运行的，这意味着抵达其他线程的连接没有被处理。所以在下面的所有例子中，我们需要在所有核上运行相同的service。我们可以用`smp::submit_to`函数轻松实现这一需求：

```c++
seastar::future<> service_loop();

seastar::future<> f() {
    return seastar::parallel_for_each(boost::irange<unsigned>(0, seastar::smp::count),
            [] (unsigned c) {
        return seastar::smp::submit_to(c, service_loop);
    });
}
```

这里我们要求每个Seastar核（从0到`smp::count` - 1）都运行`service_loop()`。每个调用都会返回一个future，`f()`则会在这些future都返回了之后再返回（在下面的例子里，他们都不会返回——我们会在之后的章节介绍该如何停止这些service）。

我们有一个简单的TCP例子开始。这个server会循环地accept 1234端口上的连接，并返回空回复。

```c++
#include <seastar/core/seastar.hh>
#include <seastar/core/reactor.hh>
#include <seastar/core/future-util.hh>
#include <seastar/net/api.hh>

seastar::future<> service_loop() {
    return seastar::do_with(seastar::listen(seastar::make_ipv4_address({1234})),
            [] (auto& listener) {
        return seastar::keep_doing([&listener] () {
            return listener.accept().then(
                [] (seastar::accept_result res) {
                    std::cout << "Accepted connection from " << res.remote_address << "\n";
            });
        });
    });
}
```

这段代码会按如下顺序运行：

1. `listen()`会创建一个`server_socket`对象`listener`，其会监听1234端口（on any network interface）。
2. 我们用`do_with()`来保证listener socket在整个循环的声明周期中都活着。
3. 为了处理一个连接，我们调用`listener`的`accept`方法。这个方法返回一个`future<accept_result>`，也就是最终会解析为client的TCP connection（`accept_result.connection`）和client的IP地址与端口（`accept_result.remote_address`）。
4. 为了循环地接受新的连接，我们使用了`keep_doing()`。`keep_doing()`会一次又一次地运行它的lambda参数，在前一个future的值可用之后马上开始一个新的迭代步。只有在出现异常的时候循环才会停止。只有在`keep_doing()`的循环结束了之后，其返回的future才会完成（也就是只有在出现异常的时候）。

这个server的输出样例为：

```bash
$ ./a.out
Accepted connection from 127.0.0.1:47578
Accepted connection from 127.0.0.1:47582
...
```

如果你在杀掉上面的server之后马上重新运行，往往会失败，报如下的错：

```bash
$ ./a.out
program failed with uncaught exception: bind: Address already in use
```

这是因为在默认情况下，日过有任何就旧连接使用过的痕迹，Seastar就会拒绝重用这个本地端口。在我们的傻傻的server中，因为server这边先关掉的链接，每个连接会会在关闭后"`TIME_WAIT`"的时间内保持linger，从而阻止`listen()`同一个端口。幸运的是，我们可以通过下列配置项来让`listen()`在`TIME_WAIT`中仍可以运行。这个选项类似于`socket(7)`的`SO_REUSADDR`：

```c++
    seastar::listen_options lo;
    lo.reuse_address = true;
    return seastar::do_with(seastar::listen(seastar::make_ipv4_address({1234}), lo),
```

大多数server都会打开`reuse_address`选项。Stevens' book "Unix Network Programming"甚至说“所有TCP server都应该通过设置这一选项来让server可以重启”。因此之后Seastar应该会把这个设置默认打开——即使因为历史原因这不是Linux socket的默认设置。

让我们增改我们的server使其可以返回一些固定的回复，而不是直接关闭连接：

```c++
#include <seastar/core/seastar.hh>
#include <seastar/core/reactor.hh>
#include <seastar/core/future-util.hh>
#include <seastar/net/api.hh>

const char* canned_response = "Seastar is the future!\n";

seastar::future<> service_loop() {
    seastar::listen_options lo;
    lo.reuse_address = true;
    return seastar::do_with(seastar::listen(seastar::make_ipv4_address({1234}), lo),
            [] (auto& listener) {
        return seastar::keep_doing([&listener] () {
            return listener.accept().then(
                    [] (seastar::accept_result res) {
                auto s = std::move(res.connection);
                auto out = s.output();
                return seastar::do_with(std::move(s), std::move(out),
                        [] (auto& s, auto& out) {
                    return out.write(canned_response).then([&out] {
                        return out.close();
                    });
                });
            });
        });
    });
}
```

新增的部分会先取`connected_socket`的`output()`，其为一个`output_stream<char>`对象。我们用`write()`向输出流`out`写入我们的回复。看似简单的`write()`实际上底层是一个复杂的异步操作，可能会在需要时发送多个包，重发，等等。`write()`返回的future表示什么时候可以再次向这个输出流`write()`，但是不保证远端已经接收到我们发送的所有数据了，但是保证输出流的buffer有足够的空间（或者对于TCP来说，在TCP的congestion window中有足够的空间）以允许一个新的写操作了。

在`write()`之后，这个例子调用了`out.close()`，并等待其完成。这是必要的，因为`write()`会尝试批量发送，所以在`close()`结束的时候，我们才能保证我们写入的所有的数据的确抵达了TCP stack——只有在这个时候我们才能释放`out`和`s`。

server也的确会进行预期的操作：

```
$ telnet localhost 1234
...
Seastar is the future!
Connection closed by foreign host.
```

在上面的例子中，我们只看到写入socket。真实世界的server也会需要读操作。`connected_socket`的`input()`方法返回一个`input_stream<char>`对象，可以用来读。最简单的方法就是用`read()`方法来读。这个方法会返回一个`temporary_buffer<char>`的future，其中保留了一些从socket中读到的数据——或者如果远端关闭了连接，返回一个空的buffer。

使用`temporary_buffer<char>`是一个方便且安全的传递临时需要的buffer的方式（例如处理请求时）。一旦对象离开作用域（通过普通return或者异常），它承载的内存会自动被释放。buffer的所有权可以用通过`std::move()`转移。我们会在后续的章节中更详细地讨论`temporary_buffer`。

我们来看一个设计读写的简单例子。这是这是一个简单的echo server，如RFC 862所叙述的：server会接听client发送的请求，一旦连接搭建成功，任何从client发过来的数据会直接被发送回去，直到client中断连接。

```c++
#include <seastar/core/seastar.hh>
#include <seastar/core/reactor.hh>
#include <seastar/core/future-util.hh>
#include <seastar/net/api.hh>

seastar::future<> handle_connection(seastar::connected_socket s,
                                    seastar::socket_address a) {
    auto out = s.output();
    auto in = s.input();
    return do_with(std::move(s), std::move(out), std::move(in),
            [] (auto& s, auto& out, auto& in) {
        return seastar::repeat([&out, &in] {
            return in.read().then([&out] (auto buf) {
                if (buf) {
                    return out.write(std::move(buf)).then([&out] {
                        return out.flush();
                    }).then([] {
                        return seastar::stop_iteration::no;
                    });
                } else {
                    return seastar::make_ready_future<seastar::stop_iteration>(
                            seastar::stop_iteration::yes);
                }
            });
        }).then([&out] {
            return out.close();
        });
    });
}

seastar::future<> service_loop_3() {
    seastar::listen_options lo;
    lo.reuse_address = true;
    return seastar::do_with(seastar::listen(seastar::make_ipv4_address({1234}), lo),
            [] (auto& listener) {
        return seastar::keep_doing([&listener] () {
            return listener.accept().then(
                    [] (seastar::accept_result res) {
                // Note we ignore, not return, the future returned by
                // handle_connection(), so we do not wait for one
                // connection to be handled before accepting the next one.
                (void)handle_connection(std::move(res.connection),
                                        std::move(res.remote_address)).handle_exception(
                        [] (std::exception_ptr ep) {
                    fmt::print(stderr, "Could not handle connection: {}\n", ep);
                });
            });
        });
    });
}
```

主要的`service_loop`函数会接受新的连接，并对每个连接调用`handle_connection()`。`handle_connection()`会返回一个描述何时处理完连接的future。重要的是，我们**不**等待这个future：记住`keep_doing`只会在前一个future准备好之后才会进行下一个迭代步。因为偶尔们像要允许并行运行的连接，我们不希望`accept()`要等到前一个连接关闭才能运行，所以我们调用`handle_connection()`以开始处理connection，但是不返回任何future，从而使得这个返回的空future立即准备好，从而`keep_doing`可以继续运行下一个`accept()`。

这展示了在Seastar中并行运行filber（continuation链）是多么得容易——当一个continuation运行了一个异步函数但是忽略了其返回的future的时候，这个异步函数会并行运行，不会等待。

大多时候悄悄地忽略一个异常是错误地，所以如果我们忽略返回的那个future出现异常，我们推荐用`handle_exception()`这样的continuation来处理异常。在我们的情况下，一个失败的connection无关痛痒（如，一个client关闭了一个我们正在发送output的连接），所以我们并没有去处理异常。

`handle_connect()`非常直接——它重复地在input stream上调用`read()`，把收到的`template_buffer`通过`write()`写入output stream。当`read()`最终返回空的时候，我们停掉`repeat`。

# Sharded services

在前文我们看到了Seastar应用往往需要在所有可用的CPU核上运行。我们注意到`seastar::smp::submit_to()`让函数可以运行在所有核上。

然而，通常我们不仅需要在每个核上运行代码，还需要一个对象来保存状态。除此之外，我们可能会需要操作这些对象，以及一个用于停止这些运行在核上的service的机制。

`seastar::sharded<T>`样本提供了一个结构化的创建这样的_sharded service_的模板。它会在每个核上创建一个单独的T类型的对象，并提供了操作这些副本们的方法，以及如何干净地停掉这个服务的方式。

如果要使用`seastar::sharded`，首先创建一个用于保存单个核上服务的状态的类。例如：

```c++
#include <seastar/core/future.hh>
#include <iostream>

class my_service {
public:
    std::string _str;
    my_service(const std::string& str) : _str(str) { }
    seastar::future<> run() {
        std::cerr << "running on " << seastar::engine().cpu_id() <<
            ", _str = " << _str << \n";
        return seastar::make_ready_future<>();
    }
    seastar::future<> stop() {
        return seastar::make_ready_future<>();
    }
};
```

其中必须创建的函数只有`stop()`。它是用来在停止service的时候被调用的。

之后我们来看一下如何使用这个类：

```c++
#include <seastar/core/sharded.hh>

seastar::sharded<my_service> s;

seastar::future<> f() {
    return s.start(std::string("hello")).then([] {
        return s.invoke_on_all([] (my_service& local_service) {
            return local_service.run();
        });
    }).then([] {
        return s.stop();
    });
}
```

`s.start()`通过在每个核上创建一个`my_service`从而开启了service。`s.start()`的参数会被传给`my_service`的constructor。

但是`s.start()`并没有开始运行任何代码（除去构造函数）。为了运行代码，我们调用了`s.invoke_on_all()`，用于在所有核上调用指定lambda——每个lambda可以使用对应核上的`my_service`。在本例中，我们运行了`run()`。

最后，在终止时我们通过调用`s.stop()`来停止service。这会调用每个核上的对象的`stop()`方法。必须要在摧毁`s`前调用`s.stop()`——不然Seastar会发出警告。

除了`invoke_on_all()`这个会在所有shard上运行同样的代码的方法之外，`invoke_on()`可以用于在特定shard上调用函数。例如：

```c++
seastar::sharded<my_service> s;
...
return s.invoke_on(0, [] (my_service& local_service) {
    std::cerr << "invoked on " << seastar::engine().cpu_id() <<
        ", _str = " << local_service._str << "\n";
});
```

这会在shard 0上运行lambda函数，并且使用了该shard上的`my_service`的引用。

# Shutting down cleanly

TODO: Handling interrupt, shutting down services, etc.

Move the seastar::gate section here.

# Command line options

## Standard Seastar command-line options



## User-defined command-line options



# Debugging a Seastar program

## Debugging ignored exceptions



## Finding where an exception was thrown



## Debugging with gdb

```
handle SIGUSR1 pass noprint
handle SIGALRM pass noprint
```

# Promise objects

如我们



CONTINUE HERE. write an example, e.g., something which writes a message every second, and after 10 messages, completes the future.

# Seastar中的内存分配

## Per-thread内存分配

Seastar需要整个应用被sharded，也就是说，在不同线程上运行的代码会操作内存中的对象不同的对象。我们已经在[内存]一节看到Seastar是如何占据内存的（大多时候会占据绝大部分内存）并平均地分配不同的线程。现代多socket机器有non-uniform memory access (NUMA)，也就说部分内存里部分核更近。Seastar会把这点考虑进来。现在，线程间的内存分配是静态地，也是平均地——线程们被认为会需要相仿量的内存。

为了实现这种per-thread分配，Seastar重新定义了C的库函数`malloc()`, `free()`以及众多相关的函数——`calloc()`, `realloc()`, `posix_memalign()`, `memalign()`, `malloc_usable_size() ` 和`malloc_trim()`。Seastar也重新定义了C++的内存分配函数， `operator new`, `operator delete`以及其诸多变形（包括数组相关的版本，C++14的带size的delete，C++17中带对齐的变形）。

重要的是记住Seastar的不同线程可能看到其他线程分配的内存，但是**极不推荐**去真的这么做。因为锁、memory barrier、cache-line bouncing等问题，在现代多核机器上进行线程间数据共享会导致严重的性能损耗。Seastar更推荐应用避免共享对象（通过分片——每个线程掌握对象的一部分），而当线程们需要交互的时候，进行显式的message passing，用我们之后会看到的`submit_to()`。

## Foreign pointers



# Seastar::thread



## Starting and ending a seastar::thread



# Isolation of application components



## Scheduling groups (CPU scheduler)

考虑如下的异步函数`loop()`，其会运行至`stop`变为`true`。它维护了一个用于记录迭代步数的`counter`，并会在停止时返回它。

```c++
seastar::future<long> loop(int parallelism, bool& stop) {
    return seastar::do_with(0L, [parallelism, &stop] (long& counter) {
        return seastar::parallel_for_each(boost::irange<unsigned>(0, parallelism),
            [&stop, &counter]  (unsigned c) {
                return seastar::do_until([&stop] { return stop; }, [&counter] {
                    ++counter;
                    return seastar::make_ready_future<>();
                });
            }).then([&counter] { return counter; });
    });
}
```

`parallelism`决定了这个傻傻的计数器的并行度：`parallelism=10`意味着我们会开启10个循环，每个循环都向相同的计数器做加法。

如果我们启动2个`loop()`并让他们跑10s，会怎么样呢？

```c++
seastar::future<> f() {
    return seastar::do_with(false, [] (bool& stop) {
        seastar::sleep(std::chrono::seconds(10)).then([&stop] {
            stop = true;
        });
        return seastar::when_all_succeed(loop(1, stop), loop(1, stop)).then(
            [] (long n1, long n2) {
                std::cout << "Counters: " << n1 << ", " << n2 << "\n";
            });
    });
}
```

结果是如果2个`loop()`的并行度都是1，它们会被执行相似的次数：

```
Counters: 3'559'635'758, 3'254'521'376
```

但是如果第一个是`loop(1)`，第二个是`loop(10)`，那么`loop(10)`会得到大约10倍的执行次数：

```
Counters: 629'482'397, 6'320'167'297
```

为什么10秒内`loop(1)`被执行的次数取决于其竞争者呢？我们怎样才能解决这个问题呢？

发生上述情况的原因是：当一个future准备好且有continuation绑定在它之上时，continuation就会准备开始运行了。默认来说，Seastar的调度器会（在每个shard上）保持一个准备跑的continuation的链表，并按照这个链表的顺序运行。在上面的例子中，`loop(1)`总是有1个准备运行的continuation，而`loop(10)`有10个，所以`loop(10)`被执行了10倍。

为了解决这个问题，Seastar允许应用定义单独的**scheduling group**。每个scheduling group会有自己的链表维护准备运行的continuation。每个scheduling group可以在指定百分比的CPU时间上运行自己的continuation，每个scheduling group内部的可运行的continuation的数量不影响CPU的分配。我们来看一下shceduling group是怎么用的：

一个scheduling group由一个`scheduling_group`值决定。这个值是opaque的，但是其实内部实现就是一个整数（类似linux的process ID）。我们用`seastar::with_scheduling_group()`来运行每个scheduling group需要执行的代码：

```c++
seastar::future<long>
loop_in_sg(int parallelism, bool& stop, seastar::scheduling_group sg) {
    return seastar::with_scheduling_group(sg, [parallelism, &stop] {
        return loop(parallelism, stop);
    });
}
```

TODO: explain what `with_scheduling_group` group really does, how the group is "inherited" to the continuations started inside it.

下面我们来创建2个scheduling groups，一个运行`loop(1)`一个运行`loop(10)`：

```c++
seastar::future<> f() {
    return seastar::when_all_succeed(
            seastar::create_scheduling_group("loop1", 100),
            seastar::create_scheduling_group("loop2", 100)).then(
        [] (seastar::scheduling_group sg1, seastar::scheduling_group sg2) {
        return seastar::do_with(false, [sg1, sg2] (bool& stop) {
            seastar::sleep(std::chrono::seconds(10)).then([&stop] {
                stop = true;
            });
            return seastar::when_all_succeed(loop_in_sg(1, stop, sg1), loop_in_sg(10, stop, sg2)).then(
                [] (long n1, long n2) {
                    std::cout << "Counters: " << n1 << ", " << n2 << "\n";
                });
        });
    });
}
```

这里我们创建2个scheduling group，`sg1`和`sg2`。每个scheduling group取了一个名字（仅仅用于诊断查错），和他的`share`数，一个一般在1到1000之间的数：如果一个scheduling group的share是另一个的2倍大，那么它就会得到2倍大多的CPU时间。在本例中，我们都设置为了100，所以他们获得的CPU时间相同。

不像大多数Seastar中的对象一样在每个shard上相互独立，Seastar要求所有shard上的scheduling group个数相同，名字_(identity)_相同，因为这会影响到调用remote shard上的人物。出于这个原因，`seastar::create_scheduling_group()`是用来创建scheduling_group的函数，会返回一个`future<scheduling_group>`。

用相同的share number来运行上面的例子，的确得到两个scheduling group的运行次数相仿：

```
Counters: 3'353'900'256, 3'350'871'461
```

如果我们把第二个scheduling group的share改成200，也就是第一个的一倍多，那么我们会看到第二个会得到了2倍的CPU时间：

```
Counters: 2'273'783'385, 4'549'995'716
```

## Latency

TODO: Task quota, preempt, loops with built-in preemption check, etc.

## Disk I/O scheduler

TODO

## Network scheduler

TODO: Say that not yet available. Give example of potential problem - e.g., sharing a slow WAN link.

## Controllers

TODO: Talk about how to dynamically change the number of shares, and why.

## Multi-tenancy

TODO

