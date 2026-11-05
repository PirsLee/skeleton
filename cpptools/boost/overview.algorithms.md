# Algorithms

[Algorithm]()

Boost.Algorithm 是 Boost 里“STL 算法工具箱的补充包”：它把 C++ 标准库里一直没有、或者很晚才加进去的通用算法——字符串处理、范围钳制、序列重排、高速排序、十六进制编解码等——打包成一个头文件库。接口风格刻意模仿 STL，泛型、非侵入、零依赖。很多算法后来被 C++11/14/17 标准化了，但 Boost.Algorithm 里至今仍有大量 STL 没有的实用工具。

[Foreach]()

❌

Boost.Foreach 是 Boost 里“把遍历写得像脚本语言一样简洁”的库：它用一个宏 BOOST_FOREACH，让你在 C++03 里就能写出接近 for (auto& x : container)的遍历代码，完全隐藏迭代器样板。在 C++11 范围 for 普及之前，它是 Boost 项目里出镜率最高的工具之一；今天它更多作为兼容老编译器的备选，但理解它有助于读懂大量历史代码。

[Graph]()

Boost.Graph（BGL）是 Boost 里“图论算法的 STL”：它把 STL 的泛型哲学推到了极致——图的数据结构（邻接表、邻接矩阵）和图算法（BFS、DFS、最短路径、最小生成树）完全解耦，算法只依赖图概念（Graph Concept）而不是某个具体类。就像 std::sort能排序任何提供迭代器的容器一样，BGL 的 dijkstra_shortest_paths能跑在任何满足“图”接口的自定义结构上。

[GraphParallel]()

Boost.GraphParallel（也就是 Parallel BGL / PBGL）是 Boost.Graph 的并行分布式扩展：它把 BGL 的泛型图接口从“单机单地址空间”抬到“多进程多节点”，让你用和顺序 BGL 几乎一样的代码，跑在 MPI 集群上处理大图。核心卖点是——接口不变，只换数据结构和后端。

[Range]()

Boost.Range 是 Boost 里“把迭代器对抽象成一等公民”的库：它把“一对 begin/end迭代器”封装成 Range​ 概念，让你写算法时不再反复写 v.begin(), v.end()，而是直接把容器、数组、C 字符串、迭代器 pair、甚至流都当成“一段可遍历的东西”来处理。它也是后来 C++20 std::ranges的思想前身之一——但 Boost.Range 更早、更贴近迭代器层，且不要求 C++20。