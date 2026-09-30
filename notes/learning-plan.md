# brpc 系统学习计划

> 基于当前仓库版本 `dd43e724` 制定，日期：2026-09-30。默认具备 C++ 基础，每周投入 6–8 小时，主线共 8 周、约 48–64 小时。本文中的实验是待完成的学习任务，并非已经运行的验证结果。

## 1. 学习目标与使用方式

完成这份计划后，应能独立完成以下任务：

1. 构建项目、运行 Echo 服务、执行指定单测，记录可复现的环境和命令。
2. 正确使用 `Server`、`Channel`、`Controller`，解释同步/异步调用和对象生命周期。
3. 从 `stub.Echo()` 追踪到服务端处理，再回到客户端完成；区分函数调用、网络传输和任务调度边界。
4. 结合源码解释 IOBuf、bthread、服务发现、负载均衡、超时、重试和 backup request。
5. 使用内置服务、bvar、rpcz 和压测工具定位一个延迟或失败问题。
6. 完成一个小型综合实验，以及一项有验证证据的代码或文档改进。

每周按 **文档 1 小时 → 源码 2 小时 → 实验与测试 2–3 小时 → 笔记和复盘 1–2 小时** 推进。以验收清单为进度标准；若每周只有 3–4 小时，将每个阶段拆成两周。

先做能力自检：能否解释 RAII/智能指针/回调生命周期、互斥锁/条件变量/原子操作、TCP 字节流/部分读写、Protobuf 的 message/service/stub？不熟悉的内容在对应阶段补齐；如果多数不熟悉，先增加 1–2 周准备期。汇编切栈、所有协议和全部负载均衡算法放入进阶选修。

阅读策略：先读头文件中的接口契约，再读一个正常路径和一个失败路径，最后用测试核对推断。遇到大文件，按符号跳转，不从头逐行读。仓库中的部分文档包含历史路径和旧环境说明；接口细节以当前源码与测试为准。

## 2. 项目地图与主线

从 [README_cn.md](../README_cn.md) 和 [概述](../docs/cn/overview.md) 建立全景，再按下表进入源码。

| 区域 | 学习内容 | 第一批入口 |
| --- | --- | --- |
| `example/` | 最小可运行服务与客户端 | [echo.proto](../example/echo_c++/echo.proto)、[client.cpp](../example/echo_c++/client.cpp#L40)、[server.cpp](../example/echo_c++/server.cpp#L42) |
| `src/brpc/` | RPC API、调用状态、网络收发、扩展组件 | [channel.h](../src/brpc/channel.h)、[controller.h](../src/brpc/controller.h)、[server.h](../src/brpc/server.h) |
| `src/brpc/policy/` | 协议、命名服务与负载均衡的具体实现 | [baidu_rpc_protocol.cpp](../src/brpc/policy/baidu_rpc_protocol.cpp)、[round_robin_load_balancer.cpp](../src/brpc/policy/round_robin_load_balancer.cpp) |
| `src/bthread/` | M:N 任务调度、等待与唤醒 | [bthread.h](../src/bthread/bthread.h)、[task_group.cpp](../src/bthread/task_group.cpp)、[butex.cpp](../src/bthread/butex.cpp) |
| `src/butil/` | 基础设施，主线先学 IOBuf | [iobuf.h](../src/butil/iobuf.h#L68)、[iobuf.cpp](../src/butil/iobuf.cpp) |
| `src/bvar/` | 计数、窗口统计、延迟分布 | [reducer.h](../src/bvar/reducer.h#L394)、[latency_recorder.h](../src/bvar/latency_recorder.h#L75) |
| `src/json2pb/`、`src/mcpack2pb/` | JSON/Protobuf 转换与其他格式适配 | [json2pb 文档](../docs/cn/json2pb.md)、[json_to_pb.h](../src/json2pb/json_to_pb.h)、[mcpack2pb](../src/mcpack2pb/) |
| `test/` | 行为示例与回归验证 | [测试构建定义](../test/CMakeLists.txt#L233)，按阶段选择用例 |
| `tools/`、`docs/cn/` | 压测、调试及机制说明 | [rpc_press](../tools/rpc_press/)、[中文文档目录](../docs/cn/) |

主线固定为 **Echo + baidu_std + TCP**。下面是逻辑关系图，不代表这些步骤都发生在同一个线程或调用栈中：

```mermaid
flowchart LR
    A[Echo Stub] --> B[Channel / Controller]
    B --> C[请求序列化与打包]
    C --> D[Socket / TCP]
    D --> E[服务端输入解析与分发]
    E --> F[EchoService / done]
    F --> G[响应打包与发送]
    G --> H[客户端响应解析]
    H --> I[完成调用 / 唤醒或回调]
```

其中协议回调的连接关系可从 [global.cpp 中 baidu_std 的注册](../src/brpc/global.cpp#L450) 查起；运行时路径在第 3、4 周逐步补全。

## 3. 八周安排

每周的详细计划各自包含 5 次学习单元、源码问题、实验步骤、测试命令和验收标准。按下表进入；本文件保留整体路线和公共操作说明。周计划是任务说明，自己的实验结果另存为 `notes/records/week-01.md` 至 `week-08.md`。

| 周次 | 主题 | 本周交付物 |
| --- | --- | --- |
| 1 | [构建、运行、建立项目地图](01-environment-and-echo.md) | 环境记录、Echo 成功日志、第一份单测结果 |
| 2 | [API 契约与生命周期](02-api-and-lifetime.md) | 同步/异步对照实验、对象所有权表 |
| 3 | [一次 RPC 的完整路径](03-rpc-lifecycle.md) | 带关键符号、完成事件的请求响应时序图 |
| 4 | [I/O、连接与 IOBuf](04-io-and-iobuf.md) | 收发路径图、缓冲区实验、连接模式对照 |
| 5 | [bthread 调度与同步](05-bthread.md) | 创建—运行—等待—唤醒—结束状态图 |
| 6 | [服务发现、负载均衡与故障处理](06-routing-and-failures.md) | 双实例实验、故障矩阵、尾延迟对照 |
| 7 | [bvar、rpcz 与性能观测](07-observability-and-performance.md) | 指标说明、一份可复现的性能诊断报告 |
| 8 | [综合实验与首次改进](08-capstone-and-review.md) | 可演示的实验服务、回归证据、总结 |

### 第 1 周：从零建立可运行基线

**阅读顺序**

1. [构建说明](../docs/cn/getting_started.md)，重点阅读自己的操作系统章节。
2. [echo.proto](../example/echo_c++/echo.proto) → [client.cpp](../example/echo_c++/client.cpp#L40) → [server.cpp](../example/echo_c++/server.cpp#L42)。
3. [根 CMakeLists.txt](../CMakeLists.txt#L21) → [Echo 构建文件](../example/echo_c++/CMakeLists.txt) → [测试构建文件](../test/CMakeLists.txt#L260)，只看开关、依赖、目标和产物。

**实践**

- 按附录 A 构建，运行 Echo 客户端和服务端；发送带 attachment 的请求。
- 打开 `/status`、`/vars`、`/connections`，将页面信息与日志对应。
- 按附录 B 运行 `ChannelTest.success`；先读测试设置，再看结果。
- 记录 Git 版本、OS/CPU 架构、编译器、CMake、Protobuf/OpenSSL 版本以及实际使用的构建参数。

**验收**

- [ ] 能从全新终端按记录复现编译和启动，客户端收到 `hello world` 及 attachment。
- [ ] 能说明 `.proto` 如何生成 Stub/Service，以及请求从哪一行发出、业务在哪一行执行。
- [ ] 已保存一个指定单测的通过结果；如被环境阻塞，记录完整错误、已尝试步骤和下一步。

详细安排：[第 1 周计划](01-environment-and-echo.md)。实验记录：`notes/records/week-01.md`。

### 第 2 周：掌握 API 契约和生命周期

**阅读顺序**

1. [Client](../docs/cn/client.md)：Channel、同步/异步访问、等待完成、Controller 重用、超时设置。
2. [Server](../docs/cn/server.md)：Service 实现、异步 Service、AddService、Start、Stop/Join。
3. [channel.h](../src/brpc/channel.h)、[controller.h](../src/brpc/controller.h)、[closure_guard.h](../src/brpc/closure_guard.h)、[server.h](../src/brpc/server.h)；再定位 [Server::AddService](../src/brpc/server.cpp#L1618)。

**实践**

- 在自己的实验副本中把同步 Echo 调用改为异步调用，记录发起、回调、最终等待完成的顺序。
- 为 Channel、Stub、Controller、request、response、done、Service 列一张表：谁创建、谁持有、何时允许释放、是否可跨并发请求共享。遵循接口契约，不根据一次运行成功推断生命周期安全。
- 服务端增加显式失败分支，比较 `Failed()`、`ErrorCode()`、`ErrorText()`；正常和提前返回分支都检查 `done` 是否恰好执行一次。
- 阅读并运行 [ServerTest.serving_requests](../test/brpc_server_unittest.cpp#L1949)，观察服务启动和停止的完整流程。

**验收**

- [ ] 异步版本能够结束并回收自己的状态，没有在请求完成前销毁 Controller/response。
- [ ] 能解释 `ClosureGuard`、`release()` 及 Service ownership 的作用。
- [ ] 能用代码指出“共享 Channel”和“共享一次调用的 Controller”在使用约束上的差异。

详细安排：[第 2 周计划](02-api-and-lifetime.md)。实验记录：`notes/records/week-02.md`。

### 第 3 周：追踪一次 RPC 的完整生命周期

**按以下顺序精读，不必通读整个 channel.cpp/controller.cpp：**

| 步骤 | 源码定位 | 要回答的问题 |
| --- | --- | --- |
| 1. 发起 | [Channel::CallMethod](../src/brpc/channel.cpp#L482) | 参数、协议函数、调用 ID 和完成方式如何准备？ |
| 2. 编码 | [SerializeRpcRequest](../src/brpc/policy/baidu_rpc_protocol.cpp#L1056)、[PackRpcRequest](../src/brpc/policy/baidu_rpc_protocol.cpp#L1097) | 序列化消息和打包协议分别做什么？ |
| 3. 发送 | [Controller::IssueRPC](../src/brpc/controller.cpp#L1315) | 选址、打包、Socket 写入如何连接？ |
| 4. 解包与分发 | [ParseRpcMessage](../src/brpc/policy/baidu_rpc_protocol.cpp#L105)、[ProcessRpcRequest](../src/brpc/policy/baidu_rpc_protocol.cpp#L599) | 怎样识别完整消息，找到 Service 和 Method？ |
| 5. 业务与响应 | [svc->CallMethod](../src/brpc/policy/baidu_rpc_protocol.cpp#L875)、[Echo](../example/echo_c++/server.cpp#L42)、[SendRpcResponse](../src/brpc/policy/baidu_rpc_protocol.cpp#L288) | 谁构造 done，谁触发回包，谁释放服务端调用状态？ |
| 6. 客户端完成 | [ProcessRpcResponse](../src/brpc/policy/baidu_rpc_protocol.cpp#L944)、[OnResponse](../src/brpc/details/controller_private_accessor.h#L47)、[OnVersionedRPCReturned](../src/brpc/controller.cpp#L840)、[EndRPC](../src/brpc/controller.cpp#L1133) | 响应如何匹配请求，何时唤醒同步调用或执行回调？ |

配套阅读：[baidu_std](../docs/cn/baidu_std.md)、[当前 RpcMeta 定义](../src/brpc/policy/baidu_rpc_meta.proto)、[bthread_id](../docs/cn/bthread_id.md)。网络输入中间层在下一周补齐。

**实践**

- 用 LLDB/GDB 断点或临时日志追踪一个低频请求，记录 `log_id`、`correlation_id`、关键对象地址和完成事件。
- 画成功路径时序图，再叠加一条超时路径；标注哪些操作跨进程、哪些需要调度、哪些仅为直接函数调用。
- 对照 [ChannelTest.success](../test/brpc_channel_unittest.cpp#L2827) 和 [ChannelTest.timeout](../test/brpc_channel_unittest.cpp#L3007) 阅读两种完成路径，并执行过滤后的用例。

**验收**

- [ ] 不看笔记也能从 Stub 讲到业务函数，再讲回客户端完成，并指出不少于 8 个关键符号。
- [ ] 能区分 `log_id` 与协议中的 `correlation_id`，说明同步等待在哪里结束。
- [ ] 能沿源码解释“先超时、后收到响应”的处理，而不是仅断言响应被忽略。

详细安排：[第 3 周计划](03-rpc-lifecycle.md)。实验记录：`notes/records/week-03.md`。

### 第 4 周：I/O、连接和 IOBuf

**阅读顺序**

1. [I/O](../docs/cn/io.md)、[IOBuf](../docs/cn/iobuf.md)、[Client 的连接方式](../docs/cn/client.md#连接方式)。
2. [event_dispatcher.cpp](../src/brpc/event_dispatcher.cpp#L109)：按平台进入 [epoll 实现](../src/brpc/event_dispatcher_epoll.cpp#L203) 或 [kqueue 实现](../src/brpc/event_dispatcher_kqueue.cpp#L193)。
3. [Socket::OnInputEvent](../src/brpc/socket.cpp#L2252) → [transport.h](../src/brpc/transport.h#L32)/[tcp_transport.cpp](../src/brpc/tcp_transport.cpp#L32) → [InputMessenger::OnNewMessages](../src/brpc/input_messenger.cpp#L103) → [input_messenger_processor.cpp](../src/brpc/input_messenger_processor.cpp#L61)。本版本有 transport 和独立 processor 层，阅读时应保留这两个边界。
4. [Socket::Write](../src/brpc/socket.cpp#L1643)；[IOBuf 接口](../src/butil/iobuf.h#L68) → [cutn](../src/butil/iobuf.cpp#L744) → [两类 append](../src/butil/iobuf.cpp#L1075)。

**实践**

- IOBuf 小实验：向 A 追加字符串，将 A 复制到 B，切走 A 的前半段，断言 B 保持原内容。画出逻辑视图与共享数据块的关系。
- 对照 `append(const IOBuf&)` 和 `append(const void*, size_t)`，记录哪些操作共享数据，哪些需要复制；不要把 IOBuf 理解为所有操作均零拷贝。
- 分别使用 `single`、`pooled`、`short` 访问 Echo，记录连接建立/复用现象；低并发未体现连接池差异时，增加受控并发再比较。
- 运行 [IOBufTest.copy_and_assign](../test/iobuf_unittest.cpp#L537)、[IOBufTest.append_and_cut_it_all](../test/iobuf_unittest.cpp#L600)、[IOBufTest.conversion_with_protobuf](../test/iobuf_unittest.cpp#L1048)，再读 [SocketTest.single_threaded_write](../test/brpc_socket_unittest.cpp#L255)。

**验收**

- [ ] 能说明 TCP 一次 read 与一个 RPC 包为何不是一一对应，并找到解析器处理消息不完整的分支。
- [ ] 在第 3 周时序图中补齐事件通知、transport、输入解析和消息处理边界。
- [ ] 用断言验证缓冲区实验，用观测结果解释连接模式差异。

详细安排：[第 4 周计划](04-io-and-iobuf.md)。实验记录：`notes/records/week-04.md`。

### 第 5 周：bthread 调度、等待和唤醒

**阅读顺序**

1. [线程模型概述](../docs/cn/threading_overview.md)、[bthread](../docs/cn/bthread.md)、[bthread or not](../docs/cn/bthread_or_not.md)。
2. [bthread_start_urgent / bthread_start_background](../src/bthread/bthread.cpp#L344) → [TaskGroup::start_foreground / start_background](../src/bthread/task_group.cpp#L474) → [task_runner](../src/bthread/task_group.cpp#L320)。
3. [TaskControl::worker_thread](../src/bthread/task_control.cpp#L96)、[TaskGroup::sched / sched_to](../src/bthread/task_group.cpp#L716)，再看 [butex_wait](../src/bthread/butex.cpp#L753)。
4. 结合 [thread-local](../docs/cn/thread_local.md) 回看第 2 周的对象与线程关系。

**实践**

- 启动多个 bthread，分别记录 bthread ID 和 pthread ID，使用 `bthread_usleep` 后 join；观察逻辑任务与 worker 的关系。一次运行未发生迁移也属于正常观测结果。
- 在固定 worker 配置下，对比任务使用 `bthread_usleep` 与阻塞系统调用时的完成时间和进度；先写预测，再看数据，不用单次耗时下性能结论。
- 执行 [BthreadTest.bthread_join](../test/bthread_unittest.cpp#L257)、[BthreadTest.bthread_usleep](../test/bthread_unittest.cpp#L602)、[BthreadTest.yield_single_thread](../test/bthread_unittest.cpp#L718)。

**验收**

- [ ] 画出任务创建、排队、运行、等待、唤醒和结束的状态变化，标明对应符号。
- [ ] 能解释阻塞一个 bthread 与阻塞 pthread worker 的区别，以及两种情况如何影响其他任务。
- [ ] 能用本周机制重新解释第 3 周同步 RPC 的等待过程。

详细安排：[第 5 周计划](05-bthread.md)。实验记录：`notes/records/week-05.md`。切栈汇编、work stealing 细节和 [ExecutionQueue](../docs/cn/execution_queue.md) 可在主线完成后深入。

### 第 6 周：服务发现、负载均衡和故障处理

**阅读顺序**

1. [负载均衡](../docs/cn/load_balancing.md)、[Client 的重试与超时](../docs/cn/client.md)、[backup request](../docs/cn/backup_request.md)。
2. [NamingService](../src/brpc/naming_service.h#L36) → [NamingServiceThread 的成员更新](../src/brpc/details/naming_service_thread.cpp#L92) → [LoadBalancerWithNaming](../src/brpc/details/load_balancer_with_naming.cpp#L47) → [RoundRobinLoadBalancer::SelectServer](../src/brpc/policy/round_robin_load_balancer.cpp#L96)。
3. 回读 [IssueRPC](../src/brpc/controller.cpp#L1315)、[OnVersionedRPCReturned](../src/brpc/controller.cpp#L840)，再读 [RetryPolicy](../src/brpc/retry_policy.h#L28)、[默认重试实现](../src/brpc/retry_policy.cpp)、[ChannelOptions](../src/brpc/channel.h#L61)。

**实践**

- 在 8000、8001 端口启动两个本地 Echo 实例；使用 `list://127.0.0.1:8000,127.0.0.1:8001` 和 `rr`，按响应的远端地址或服务端计数统计分布。
- 换成 `file://` 命名，修改成员列表，记录成员更新与单次请求选址两条不同路径。
- 在实验服务中为一个实例加入可配置延迟，分别测试正常、慢响应、连接拒绝、运行中停止实例。每次记录 timeout、max_retry、backup 设置、错误码、耗时和服务端实际执行次数。
- 对比关闭/开启 backup request 时的尾延迟与额外请求数。原始 Echo 示例没有 backup 参数，需在实验副本中设置 `ChannelOptions::backup_request_ms`，不能直接传一个不存在的命令行开关。
- 运行 [ChannelTest.timeout](../test/brpc_channel_unittest.cpp#L3007)、[ChannelTest.retry](../test/brpc_channel_unittest.cpp#L3151)、[ChannelTest.retry_other_servers](../test/brpc_channel_unittest.cpp#L3161)、[ChannelTest.backup_request](../test/brpc_channel_unittest.cpp#L3192)；负载均衡先读 [LoadBalancerTest.fairness](../test/brpc_load_balancer_unittest.cpp#L611)。

**验收**

- [ ] 提交至少四种场景的故障矩阵，并能沿源码说明每种结果的原因。
- [ ] 能区分连接超时、整个 RPC 的超时和 backup 触发时间，找到默认重试策略的判断依据。
- [ ] 能解释普通重试和备份请求的差别、请求放大的代价，以及为何业务仍需考虑幂等性。

详细安排：[第 6 周计划](06-routing-and-failures.md)。实验记录：`notes/records/week-06.md`。熔断和自适应限流留到第 8 周选题或进阶学习。

### 第 7 周：可观测性与性能诊断

**阅读顺序**

1. [内置服务](../docs/cn/builtin_service.md)、[bvar](../docs/cn/bvar.md)、[bvar C++](../docs/cn/bvar_c++.md)、[rpcz](../docs/cn/rpcz.md)、[rpc_press](../docs/cn/rpc_press.md)。
2. [Adder / Reducer](../src/bvar/reducer.h#L241) → [AgentCombiner](../src/bvar/detail/combiner.h#L156)；[LatencyRecorder](../src/bvar/latency_recorder.h#L75) → [实现](../src/bvar/latency_recorder.cpp#L215)。
3. 按问题选读 [CPU profiler](../docs/cn/cpu_profiler.md)、[contention profiler](../docs/cn/contention_profiler.md)、[服务端卡顿排查](../docs/cn/server_debugging.md)。

**实践**

- 在实验服务中增加请求计数和 `LatencyRecorder`，为耗时指标注明单位和测量边界；通过 `/vars` 核对累计值和窗口统计。
- 启动服务时启用 `--enable_rpcz=true`，找到一次请求的时间线，与第 3 周时序图对照。
- 使用附录 C 的低压力基线，再逐步增加负载。每组固定请求大小、并发/目标 QPS、连接方式和故障配置，预热后至少重复 3 次。
- 记录实际 QPS、成功率、p50/p99、CPU、连接数；区分服务端业务耗时与客户端端到端耗时。Echo 默认逐请求日志也可能影响结果，应记录日志设置。
- 执行 [ReducerTest.adder](../test/bvar_reducer_unittest.cpp#L43)、[VariableTest.latency_recorder](../test/bvar_variable_unittest.cpp#L349)，用测试帮助解释指标更新和暴露方式。

**验收**

- [ ] 有一份包含环境、命令、原始数据、假设和结论的诊断报告。
- [ ] 根据数据区分“业务变慢”“请求排队/调度等待”“连接或网络失败”中的至少两种情况。
- [ ] 对一个性能改动做前后对照；数据不支持收益时，明确记录“未证明改善”。

详细安排：[第 7 周计划](07-observability-and-performance.md)。实验记录：`notes/records/week-07.md`。macOS 上优先完成机制验证；涉及 Linux 专属工具或生产性能结论时，在 Linux 环境复现并独立记录。

### 第 8 周：综合实验与第一次改进

**综合项目：可观测的双实例 Echo 实验台。** 复用前七周的实验副本，保持范围可控。

完成以下内容：双实例服务、同步或异步客户端、可配置延迟/失败分支、命名与负载均衡、超时/重试/backup 对照、bvar 指标、一次 rpcz 追踪，以及正常/超时/实例退出三组可重复验证。时间有限时，保留这些基础能力，将图形界面和额外协议留作后续工作。

再从下面选择 **一项** 小改进：

- 为已定位的边界行为补一个有意义的单测，说明它验证的契约。
- 为一处 API 或文档补充生命周期/错误处理示例，并实际运行示例。
- 修复一个已经复现的小问题：先写会失败的回归测试，再修改实现。

准备时阅读 [CONTRIBUTING.md](../CONTRIBUTING.md) 与相应模块的既有测试。新增 C++ 代码沿用四空格缩进和仓库现有模式。

**最终验收**

- [ ] 一份 README 能指导他人复现综合实验，包含启动、停止、输入、预期结果和失败定位方式。
- [ ] 能在 15 分钟内讲清 RPC 全链路、bthread/IOBuf 的角色，以及一个真实观测到的问题。
- [ ] 小改进有独立 diff、行为说明和对应验证结果；代码改动先跑相关测试，再按影响范围扩展构建/测试。
- [ ] 记录仍不理解的 3–5 个问题，每个问题带具体文件、符号和下一步验证办法。

详细安排：[第 8 周计划](08-capstone-and-review.md)。实验记录：`notes/records/week-08.md`。完成这些验收即算主线结束；是否提交上游 PR 是之后的独立选择。

## 4. 可复用的操作附录

以下命令以**仓库根目录**为工作目录，示例假设使用单配置 CMake 构建。每步成功后再进行下一步。这里核对的是当前源码中的目标、参数和目录布局，尚未替你执行编译或实验。

### A. 构建与运行 Echo

先按 [getting_started.md](../docs/cn/getting_started.md) 准备编译器、CMake、gflags、Protobuf/protoc、OpenSSL、leveldb 等依赖。优先复用已有可用环境，第一周只选一套构建方式。

Linux 配置示例：

```bash
cmake -S . -B build/learning \
  -DCMAKE_BUILD_TYPE=Debug \
  -DWITH_DEBUG_SYMBOLS=ON \
  -DCMAKE_EXPORT_COMPILE_COMMANDS=ON \
  -DBUILD_UNIT_TESTS=ON \
  -DBUILD_BRPC_TOOLS=ON
```

macOS + Homebrew 配置示例（与上面的 Linux 配置二选一）：

```bash
cmake -S . -B build/learning \
  -DCMAKE_BUILD_TYPE=Debug \
  -DWITH_DEBUG_SYMBOLS=ON \
  -DCMAKE_EXPORT_COMPILE_COMMANDS=ON \
  -DBUILD_UNIT_TESTS=ON \
  -DBUILD_BRPC_TOOLS=ON \
  -DCMAKE_PREFIX_PATH="$(brew --prefix)" \
  -DOPENSSL_ROOT_DIR="$(brew --prefix openssl@3)"
```

本版本 `DOWNLOAD_GTEST` 默认开启，配置单测时会下载 Googletest。离线环境需同时设置 `-DDOWNLOAD_GTEST=OFF` 和 `-DBRPC_SYSTEM_GTEST_SOURCE_DIR=/实际/googletest/源码目录`；该参数要求源码目录，不是仅含已安装头文件/库的目录。依据见 [测试构建逻辑](../test/CMakeLists.txt#L58)。初次构建若被测试依赖阻塞，可暂时关闭单测先完成 Echo，再补齐测试环境。

`Debug` 不应被直接理解为“全部关闭优化”：当前 [公共编译选项](../CMakeLists.txt) 中另有 `-O2`。调试时以 `build/learning/compile_commands.json` 中的实际参数为准。

构建主库和 Echo；显式指定本次生成的库和头文件，避免误连已有其他构建产物：

```bash
cmake --build build/learning --target brpc-static --parallel 6
cmake -S example/echo_c++ -B build/learning-echo \
  -DCMAKE_BUILD_TYPE=Debug \
  -DBRPC_INCLUDE_PATH="$PWD/build/learning/output/include" \
  -DBRPC_LIB="$PWD/build/learning/output/lib/libbrpc.a"
cmake --build build/learning-echo --parallel 4
```

macOS 上为 Echo 的配置命令同样追加上面的 `CMAKE_PREFIX_PATH` 和 `OPENSSL_ROOT_DIR`。依赖版本与架构要和主库一致。定位依据：[公共示例依赖逻辑](../example/cmake/BrpcExample.cmake)、[主库目标](../src/CMakeLists.txt#L32)、[输出目录](../CMakeLists.txt#L525)。

在两个终端分别运行，下列进程持续运行，实验结束后各按 Ctrl-C 退出：

```bash
# 终端 A，仓库根目录
./build/learning-echo/echo_server --listen_addr=127.0.0.1:8000
```

```bash
# 终端 B，仓库根目录
./build/learning-echo/echo_client \
  --server=127.0.0.1:8000 \
  --attachment=study \
  --timeout_ms=1000 \
  --max_retry=0
```

先关闭重试以观察单次调用；第 6 周再逐项改变参数。服务运行时可访问 `http://127.0.0.1:8000/status`、`/vars`、`/connections`。

### B. 按主题运行单测

当前 CMake 将 `brpc_*_unittest.cpp` 和 `bthread*unittest.cpp` 分别构建为同名可执行文件；butil、bvar 测试分别聚合为 `test_butil`、`test_bvar`。依据：[test/CMakeLists.txt](../test/CMakeLists.txt#L233)。不要假设每个 `*_unittest.cpp` 都有独立二进制。

先构建并列出本周使用的测试，再运行精确过滤；测试执行时切到构建的 `test/` 目录，便于使用相对路径数据文件：

```bash
cmake --build build/learning --target brpc_channel_unittest --parallel 6
(
  cd build/learning/test
  ./brpc_channel_unittest --gtest_list_tests
  ./brpc_channel_unittest --gtest_filter='ChannelTest.success'
)
```

| 阶段 | 构建目标 / 二进制 | 建议起始过滤条件 |
| --- | --- | --- |
| 第 2 周 | `brpc_server_unittest` | `ServerTest.serving_requests` |
| 第 3 周 | `brpc_channel_unittest` | `ChannelTest.success:ChannelTest.timeout` |
| 第 4 周 | `test_butil` | `IOBufTest.copy_and_assign:IOBufTest.append_and_cut_it_all:IOBufTest.conversion_with_protobuf` |
| 第 4 周 | `brpc_socket_unittest` | `SocketTest.single_threaded_write` |
| 第 5 周 | `bthread_unittest` | `BthreadTest.bthread_join:BthreadTest.bthread_usleep:BthreadTest.yield_single_thread` |
| 第 6 周 | `brpc_channel_unittest` | `ChannelTest.timeout:ChannelTest.retry:ChannelTest.retry_other_servers:ChannelTest.backup_request` |
| 第 7 周 | `test_bvar` | `ReducerTest.adder:VariableTest.latency_recorder` |

实际输出必须显示运行了预期用例且通过；“运行 0 个测试”不算通过。之后再扩展同模块测试，最后按改动范围决定是否运行完整 CTest。完整测试中有性能用例和外部服务依赖，先阅读前置条件；例如 [NamingServiceTest.sanity](../test/brpc_naming_service_unittest.cpp#L83) 包含 DNS 查询，不能视为纯离线测试。

### C. 一次有限时长的压测

保持 Echo 服务运行，先构建工具，以 100 QPS 运行 30 秒：

```bash
cmake --build build/learning --target rpc_press --parallel 6
./build/learning/output/bin/rpc_press \
  --proto=example/echo_c++/echo.proto \
  --method=example.EchoService.Echo \
  --server=127.0.0.1:8000 \
  --input='{"message":"hello"}' \
  --protocol=baidu_std \
  --qps=100 \
  --duration=30 \
  --timeout_ms=1000 \
  --max_retry=0 \
  --dummy_port=8888
```

参数定义见 [rpc_press.cpp](../tools/rpc_press/rpc_press.cpp#L27)，产物位置见 [tools/CMakeLists.txt](../tools/CMakeLists.txt)。性能实验使用相同构建配置做组间对照；需要性能基线时另建 Release 目录并记录实际编译参数，不将 Debug 环境中的数值直接当作生产容量。

## 5. 笔记模板与复盘

每周在 `notes/records/` 建立一份笔记，使用下面的结构，实验原始日志放在 `build/learning-records/week-XX/`，与摘要分开保存。第 2 周创建的 `example/learning_echo_c++/` 和 `build/learning-lab/` 由后续周持续复用。

```markdown
# 本周主题

## 要回答的问题
最多 3 个，例如：响应如何唤醒同步客户端？

## 阅读证据
Git revision；文件与符号；一张调用图或对象关系图。

## 实验
环境、构建参数、完整命令、输入、预期、实际结果、重复次数。

## 结论
哪些推断有源码和实验支持？哪些仍只是猜测？

## 验收与后续
已完成的检查项；失败/阻塞原因；下一次从哪个符号或用例继续。
```

每两周做一次闭卷复盘：第 2 周写一个 Echo 使用示例；第 4 周画完整收发图；第 6 周解释故障矩阵；第 8 周演示综合项目。解释不清的部分回到最小实验，不靠继续堆阅读量解决。

常见阻碍与处理方式：

| 阻碍 | 处理方式 |
| --- | --- |
| 构建长期失败 | 区分依赖查找、编译和链接阶段，保存第一个实质错误；必要时先关单测完成示例，之后补测 |
| 源码太大，跳转失去主线 | 每次限定一个请求或一个错误码，保留“调用者—当前符号—被调用者”三层笔记 |
| 并发实验不稳定 | 用 join/同步事件明确结束条件，固定参数并重复运行，区分观察现象与机制保证 |
| 测试失败 | 先查用例前置条件、端口与资源，再判断是否为代码问题；保留最小复现 |
| 性能数据互相矛盾 | 固定构建、日志、负载与硬件环境，记录实际 QPS；区分压测端瓶颈与服务端瓶颈 |

## 6. 主线结束后的选修方向

每次只选一个方向，沿用“接口—源码—实验—测试—结论”的流程。

| 方向 | 材料 | 建议产出 |
| --- | --- | --- |
| HTTP / JSON 与多协议 | [HTTP 服务](../docs/cn/http_service.md)、[HTTP 客户端](../docs/cn/http_client.md)、[json2pb](../docs/cn/json2pb.md)、[新协议接入](../docs/cn/new_protocol.md) | 对比 baidu_std 与 HTTP 的解析、路由和错误表达 |
| 组合调用与流式通信 | [组合 Channel](../docs/cn/combo_channel.md)、[Streaming RPC](../docs/cn/streaming_rpc.md) | 一个并行聚合或流式示例，并分析生命周期 |
| 高负载与稳定性 | [熔断](../docs/cn/circuit_breaker.md)、[自适应限流](../docs/cn/auto_concurrency_limiter.md)、[雪崩](../docs/cn/avalanche.md) | 过载下的吞吐、失败率、尾延迟对照实验 |
| 并发与内存深挖 | [内存管理](../docs/cn/memory_management.md)、[Timer](../docs/cn/timer_keeping.md)、[ExecutionQueue](../docs/cn/execution_queue.md)、[Sanitizers](../docs/cn/sanitizers.md) | 选一个等待/回收问题，写可复现用例并解释机制 |
| 专用传输 | [RDMA](../docs/cn/rdma.md)、[URMA](../docs/cn/urma.md)、[UBRing](../docs/cn/ubring.md) | 在满足各自环境条件后，比较 transport 接口和完成通知路径 |

## 7. 进度清单

- [ ] 第 1 周：构建、Echo、指定单测
- [ ] 第 2 周：API、异步调用、生命周期
- [ ] 第 3 周：成功与超时的 RPC 全链路
- [ ] 第 4 周：I/O、连接、IOBuf
- [ ] 第 5 周：bthread 创建、等待、唤醒
- [ ] 第 6 周：服务发现、负载均衡、故障矩阵
- [ ] 第 7 周：指标、追踪、性能诊断
- [ ] 第 8 周：综合项目、小改进、最终复盘
