# Concurrent

[Asio]()

Boost.Asio 不是“一个 socket 库”，而是 C++ 里做网络和底层 I/O 的异步引擎。你贴的那句官方描述已经点明了它的边界：可移植的网络与其他低层 I/O——socket、定时器、主机名解析、socket 流、串口、文件描述符、Windows HANDLE，全都归它管。

Asio = Asynchronous Input/Output。它把不同操作系统上五花八门的 I/O 机制（Linux 的 epoll、BSD/macOS 的 kqueue、Windows 的 IOCP、现代 Linux 的 io_uring、其他地方退化为 select）统一封装成一套编程模型。

它的设计目标很明确：

可移植：同一份代码跨 Windows / Linux / macOS 行为一致

可扩展：单线程事件循环就能撑住成千上万并发连接

高效率：支持 scatter-gather I/O，尽量少拷贝数据

贴近 BSD socket 语义：老网络程序员上手零成本

工具包而非框架：你拿它当零件用，不被它绑架架构

[Atomic]()

Boost.Atomic​ 提供了一套 C++11 风格的原子类型和操作，让你在不使用互斥锁的情况下，对共享变量进行线程安全的读写。

它的历史地位很特殊：Boost.Atomic 是 C++11 std::atomic的"前身和原型"。后来 C++11 标准库 <atomic>大量借鉴了它的设计。现在大多数项目直接用 std::atomic，但 Boost.Atomic 仍然有价值——比如需要支持 C++98/03 的老编译器，或者需要一些标准里没有的扩展类型。

一句话概括它的作用：

用硬件级别的原子指令，替代锁来做线程间同步，消除上下文切换开销。

[Beast]()

Boost.Beast 是从 Boost 1.66 起纳入的头文件库，用纯 C++11 写，专门在 Boost.Asio 之上实现 HTTP/1 和 WebSocket（RFC 6455）​ 的协议细节：消息模型、解析、序列化、握手、帧封装。

关键定位一句话：

Beast 不是"HTTP 框架"，而是"构建 HTTP/WebSocket 系统的零件箱"。

传输层：交给 Asio（socket、定时器、io_context）

协议层：Beast 负责（HTTP 报文、WebSocket 帧）

TLS：通过 boost::asio::ssl::stream包 OpenSSL

[Context]()

Boost.Context​ 提供用户态上下文切换的能力——让你在单个线程内，手动保存当前执行点（寄存器、指令指针、栈指针），跳转到另一个执行点，以后再跳回来。

它做的事情和操作系统内核的线程切换本质一样，但：

❌ 不涉及内核态切换（syscall）

❌ 不触发 CPU 上下文切换开销

✅ 纯用户态，几条汇编指令完成

✅ 切换耗时通常在 几十纳秒​ 级别

一句话概括：

Boost.Context 让你在 C++ 里实现"用户态线程/协程/纤程"，把调度权从操作系统手里拿回来

[Coroutine2]()

用C++20协程就好了。

Boost.Coroutine2 是 C++11 起的有栈非对称协程库。官方定义很精准：

提供允许在特定位置挂起和恢复执行的子例程模板，保留局部状态，允许多次重入。

关键三点：

有栈（stackful）：每个协程有自己独立的调用栈

非对称（asymmetric）：只有“调用方 ↔ 被调方”双向关系，不是对称互跳

协作式：只有你显式 sink()/ yield才切换，内核不参与调度

底层完全依赖你刚认识的 Boost.Context​ 做上下文切换，Coroutine2 负责把“栈 + 数据流 + 生命周期”打包成好用的 API。

[Fiber]()

Boost.Fiber​ 是一个 C++11 用户态线程（纤程）库。它提供了一套几乎和 std::thread一模一样的 API，但线程的调度不在内核里，而在用户态由库自己管理。

官方定位：

纤程是轻量级的、用户态调度执行上下文，拥有自己的栈和寄存器状态，通过协作式多任务在单个 OS 线程上并发运行。

[Interprocess]()

Boost.Interprocess​ 是 Boost 里专门做进程间通信（IPC）和共享内存管理的库。它把不同 OS 上五花八门的 IPC 机制（POSIX shared memory、System V shm、Windows named shared memory、mmap、文件锁等）统一封装成一套 C++ 接口。

核心能力一句话：

让多个进程像操作普通 C++ 对象一样，通过共享内存通信、同步、甚至共享 STL 容器——零拷贝、内核旁路。

[Lockfree]()

Boost.Lockfree​ 提供一组线程安全、无锁（lock-free）的并发容器，多个线程可以同时 push/pop，不需要 std::mutex。

关键定位：

“无锁”不等于“无同步”，而是“不用操作系统锁”——靠原子操作 + CAS 循环 + 内存屏障来实现并发安全。

它对实时系统特别有价值：互斥锁有优先级反转、调度不确定性、系统调用开销；无锁结构在这些场景下行为更可预测。

[MPI]()

Boost.MPI 把"跨机器进程间通信"包装成了 C++ 风格的 send/recv/reduce——它不解决分布式系统的所有问题（没有容错、没有服务发现），但它是科学计算和高性能并行领域最成熟、最高效的通信层。

选它的判断标准很简单：如果你的程序需要跑在 2 台以上机器上，且数据量/计算量单机扛不住，MPI 是首选；如果只是做个 Web API 或单机服务，它完全是杀鸡用牛刀。

[MQTT5]()

Boost.MQTT5​ 是基于 Boost.Asio​ 构建的 C++17 MQTT 5.0 客户端库（Boost 1.82+ 引入，头文件库）。它完整实现了 MQTT 5.0 协议，支持 QoS 0/1/2 的消息发布与订阅。

一句话定位：

它是 MQTT 客户端，不是 Broker——你用它连到 MQTT 服务器（如 EMQX、HiveMQ、Mosquitto），发消息、收消息，网络琐事它全包了。

最反直觉的设计点：它没有 connect()函数。你配置好 broker 地址，调 async_run()，剩下的连接、TLS 握手、MQTT 握手、断线重连、消息重传全在内部自动跑。

[Thread]()

用标准的thread就行。

Boost.Thread​ 是 Boost 里做可移植多线程编程的库。在 C++11 之前，它是 C++ 世界多线程的事实标准；C++11 的 <thread>、<mutex>、<condition_variable>大量借鉴了它的设计。

一句话定位：

Boost.Thread 是 std::thread的超集——API 几乎一样，但多了线程中断、线程属性、shared_mutex 等扩展功能，且支持 C++98/03 老编译器。