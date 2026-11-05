# Asynchronous

[Cobalt]()


Boost.Cobalt 是 Boost 里“站在 Asio 肩膀上的 C++20 协程库”：它把 C++20 无栈协程（co_await/ co_yield/ co_return）和 Boost.Asio 的异步模型缝在一起，让你用“看起来像同步、跑起来是异步”的写法写高并发 I/O 程序。它不是又一个手写栈式协程库，而是面向现代 C++ 的协程原语 + 基础算法 + 类型工具箱。