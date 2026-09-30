# 第 4 周：I/O、连接与 IOBuf

> 导航：[总计划](learning-plan.md) · [上一周：RPC 生命周期](03-rpc-lifecycle.md) · [下一周：bthread](05-bthread.md)

本周预算 **6–8 小时**，目标是把第 3 周停在“Socket/TCP”处的时序图继续向下展开：事件如何到达、字节如何进入缓冲区、协议怎样判断完整消息、消息怎样交给处理函数，以及 IOBuf 在哪些操作中共享数据、哪些操作中复制数据。

本文是执行计划，不是实验记录。执行时把结论、命令输出和失败原因写入 `notes/records/week-04.md`，原始日志放入 `build/learning-records/week-04/`。

## 前置条件

- 已完成第 1 周构建基线，主构建目录为 `build/learning/`。
- 已完成第 2 周实验副本 `example/learning_echo_c++/`，其构建目录为 `build/learning-lab/`。
- 能从 `Channel::CallMethod` 讲到 `Socket::Write`，并能从 `ParseRpcMessage` 讲到 `ProcessRpcRequest`。
- 知道 TCP 提供字节流，不保留应用层消息边界；若这一点不清楚，先补读 [I/O 文档](../docs/cn/io.md)。
- 运行实验前创建记录目录：`mkdir -p build/learning-records/week-04 notes/records`。

## 本周完成标准

完成本周后，应能用源码回答四个问题：

1. 可读事件如何从平台事件机制传到 `InputMessengerProcessor`？
2. 为什么一次 `read` 可能只有半个 RPC，也可能包含多个 RPC？
3. `IOBuf` 的复制、`append(IOBuf)`、`append(void*, size_t)` 和 `cutn` 分别操作什么？
4. `single`、`pooled`、`short` 改变的是连接管理，还是协议与业务语义？

本周交付物：一张输入路径图、一张 IOBuf 数据共享图、一份连接模式观测表，以及带断言的两个小实验。

## 源码阅读地图

| 层次 | 文件与符号 | 阅读时必须回答的问题 |
| --- | --- | --- |
| 平台事件 | [event_dispatcher.cpp](../src/brpc/event_dispatcher.cpp)、[epoll 实现](../src/brpc/event_dispatcher_epoll.cpp)、[kqueue 实现](../src/brpc/event_dispatcher_kqueue.cpp) | 当前系统编译哪一个实现？事件标志怎样归一化后进入 Socket？ |
| Socket 入口 | [Socket::OnInputEvent / DoRead](../src/brpc/socket.cpp)、[MoreReadEvents](../src/brpc/socket_inl.h) | `_nevent` 怎样合并同一 fd 的重复事件？真正读取字节发生在哪里？ |
| transport | [Transport](../src/brpc/transport.h)、[TcpTransport::ProcessEvent](../src/brpc/tcp_transport.cpp) | transport 决定了什么？为什么不能从图中省略？ |
| messenger | [InputMessenger::OnNewMessages](../src/brpc/input_messenger.cpp) | `EAGAIN`、EOF、一次多消息分别如何处理？ |
| processor | [InputMessengerProcessor::CutInputMessage / ProcessNewMessage](../src/brpc/input_messenger_processor.cpp) | 协议探测、读缓冲和消息调度的边界在哪里？ |
| baidu_std 解析 | [ParseRpcMessage](../src/brpc/policy/baidu_rpc_protocol.cpp) | 哪些长度检查返回 `PARSE_ERROR_NOT_ENOUGH_DATA`？完整消息何时从 source 切走？ |
| 消息执行 | [ProcessInputMessage](../src/brpc/input_messenger.cpp)、[InputMessageBase](../src/brpc/input_message_base.h) | `_process` 指针何时调用？对象由谁销毁？ |
| 输出路径 | [Socket::Write](../src/brpc/socket.cpp)、[TcpTransport::CutFromIOBuf](../src/brpc/tcp_transport.cpp) | 写入立即完成吗？待写数据与 fd 可写事件怎样衔接？ |
| 缓冲结构 | [IOBuf 接口](../src/butil/iobuf.h)、[IOBuf 实现](../src/butil/iobuf.cpp) | Block、BlockRef 与逻辑字节序列是什么关系？ |

先画下面这条骨架，阅读后再补充条件分支：

```text
epoll/kqueue event
  -> Socket::OnInputEvent
  -> TcpTransport::ProcessEvent
  -> InputMessenger::OnNewMessages
  -> Socket::DoRead(IOPortal)
  -> InputMessengerProcessor::ProcessNewMessage
  -> InputMessengerProcessor::CutInputMessage
  -> protocol parse / MakeMessage
  -> TcpTransport::QueueMessage
  -> ProcessInputMessage
  -> protocol process callback
```

注意：这是控制流骨架，不等于单一 pthread 调用栈；`ProcessEvent` 和 `QueueMessage` 都可能创建 bthread，第 5 周再解释调度细节。

## 学习单元 1（75 分钟）：事件通知到读取入口

**阅读（35 分钟）**

1. 读 [docs/cn/io.md](../docs/cn/io.md)，只摘录 blocking、non-blocking、edge-trigger 三个概念。
2. 在 `event_dispatcher_epoll.cpp` 定位 `EventDispatcher::AddConsumer`、`RemoveConsumer`、`Run`。
3. 在 `event_dispatcher_kqueue.cpp` 找对应函数，比较注册读事件、删除事件、等待事件的系统调用。
4. 在 `socket.cpp` 定位 `Socket::OnInputEvent`，在 `socket_inl.h` 定位 `MoreReadEvents`。

**带问题的操作（25 分钟）**

- 用 `uname -s` 记录当前平台；Linux 只精读 epoll，macOS/BSD 只精读 kqueue，但必须浏览另一实现的同名方法。
- 搜索构建条件，记录当前平台为何只编译其中一种实现。
- 沿事件参数写下“可读、错误、远端关闭”的平台标志如何影响 `OnInputEvent`。
- 找出 `OnInputEvent` 中 `_nevent.fetch_add` 的分支，再用 `MoreReadEvents` 解释处理期间到达的新事件如何被消费；不凭旧文档补一个不存在的中间函数。

**观察与记录（15 分钟）**

- 在周记录中建立“epoll / kqueue 对照表”：注册、删除、等待、远端关闭提示各一行。
- 明确写下：两种实现的 API 和事件标志不同，但上层输入处理目标相同。
- 不把 Linux 的 `EPOLLRDHUP` 当成 macOS 可直接观察的标志，也不在 macOS 计划里要求运行 epoll 工具。

## 学习单元 2（90 分钟）：transport、messenger 与 processor

**阅读（45 分钟）**

1. 读 [transport.h](../src/brpc/transport.h) 的抽象接口，再读 [TcpTransport::Init / ProcessEvent / QueueMessage](../src/brpc/tcp_transport.cpp)。
2. 读 [InputMessenger::OnNewMessages](../src/brpc/input_messenger.cpp)，逐个标记 `nr > 0`、EOF、`EINTR`、`EAGAIN` 和其他错误。
3. 读 [InputMessengerProcessor::OnceReadSize / CutInputMessage / ProcessNewMessage](../src/brpc/input_messenger_processor.cpp)。
4. 回到 [ParseRpcMessage](../src/brpc/policy/baidu_rpc_protocol.cpp)，找出头部不足与 body 不足两个“不完整”出口。

**带问题的操作（30 分钟）**

- 在第 3 周图中补上 transport 和 processor 两层，不直接画成 `Socket -> ParseRpcMessage`。
- 回答：`IOPortal` 中已有半包时，下次读取从哪里继续？
- 回答：一次读取含三个完整请求时，哪几个消息被排入独立 bthread，最后一个消息在哪里处理？
- 回答：服务端未知协议时为什么可能尝试多个 handler，而已连接客户端的响应协议通常是固定的？
- 回答：`PARSE_ERROR_NOT_ENOUGH_DATA` 为什么不是连接错误？读取 EOF 后又有什么不同？

**观察与记录（15 分钟）**

- 在路径图上用实线表示直接调用，用虚线表示可能排队到 bthread。
- 为每个跨层对象标注类型：fd、`IOPortal`、`ParseResult`、`InputMessageBase`。
- 记录尚未解释的调度问题，留到第 5 周，不在本周推断 worker 行为。

## 学习单元 3（150 分钟）：IOBuf 的共享与复制语义

**阅读（40 分钟）**

1. 读 [IOBuf 文档](../docs/cn/iobuf.md)，特别关注“管理结构复制”和“数据块共享”。
2. 在 [iobuf.h](../src/butil/iobuf.h) 定位复制构造、赋值、`append` 重载、`cutn`、`copy_to`、`IOPortal`。
3. 在 [iobuf.cpp](../src/butil/iobuf.cpp) 对照 `IOBuf::cutn`、`append(const IOBuf&)`、`append(const void*, size_t)`。
4. 读 [IOBufTest.copy_and_assign](../test/iobuf_unittest.cpp)、`append_and_cut_it_all`，先预测每个断言，再看实现。

**带问题的操作（30 分钟）**

- 画 A、B 两个 IOBuf 的逻辑 BlockRef 序列，标注它们可能共同引用的 Block。
- 区分“复制 IOBuf 管理结构”“复制外部字节进入 Block”“从逻辑序列切走引用”三种动作。
- 查明 `cutn(&out, n)` 对源和目标的长度影响；不要把“修改拷贝不影响原 IOBuf”误读为底层块永不共享。
- 解释为何 IOBuf 适合短生命周期的网络缓冲，却不宜无界期持有大量小片段。

**观察与记录（20 分钟）**

建立如下表格并填入源码依据：

| 操作 | 目标长度变化 | 源长度变化 | 数据块通常共享还是复制 | 边界情况 |
| --- | ---: | ---: | --- | --- |
| `IOBuf b = a` | 变为 `a.size()` | 不变 | 待核对 | 空 IOBuf |
| `b.append(a)` | 增加 | 不变 | 待核对 | 自追加是否允许 |
| `b.append(ptr, n)` | 增加 | 不适用 | 待核对 | `n == 0` |
| `a.cutn(&b, n)` | `b` 增加 | `a` 减少 | 待核对 | `n > a.size()` |

### 实验 1（计入单元 3，60 分钟）：带断言的 IOBuf 共享实验

此实验需要先新增程序，不能直接运行一个不存在的可执行文件。

1. 在 `example/learning_echo_c++/` 新增 `iobuf_lab.cpp`。
2. 在该目录 `CMakeLists.txt` 中新增 `iobuf_lab` target，并沿用 `brpc_example_configure_target` 和 `${BRPC_LIB}` 的链接方式。
3. 程序构造 A=`"header-body"`，复制得到 B；从 A `cutn` 前 7 字节到 C；再分别向 A 追加原始字符数组、向 C 追加 B。
4. 每一步用 `CHECK_EQ` 检查 A/B/C 的 `size()` 与 `to_string()`，并打印逻辑结果。
5. 重新配置并只构建新 target：

```bash
set -o pipefail
cmake -S example/learning_echo_c++ -B build/learning-lab
cmake --build build/learning-lab --target iobuf_lab -j6
./build/learning-lab/iobuf_lab 2>&1 | tee build/learning-records/week-04/iobuf-lab.log
```

**成功预期**：B 在 A 被切分和追加后仍保持 `header-body`；C 得到切出的前缀，再能追加 B；所有断言通过。

**失败预期**：若预期字符串或长度写错，`CHECK_EQ` 应明确失败；若 target 未加入 CMake，构建阶段应报告未知 target，而不是把它误判为运行时问题。

**边界**：本实验验证公开行为，不直接证明“绝对零拷贝”；要判断共享方式，仍需以复制构造和 `append` 实现为依据。不要依赖私有 Block 地址编写脆弱断言。

## 学习单元 4（90 分钟）：写路径与连接模式实验

**阅读（30 分钟）**

1. 读 [Socket::Write](../src/brpc/socket.cpp) 的两个主要重载，记录写队列、错误回调与 fd 可写等待的关系。
2. 读 [TcpTransport::CutFromIOBuf / WaitEpollOut](../src/brpc/tcp_transport.cpp)。
3. 读 [client.md 的连接方式](../docs/cn/client.md)，只研究 `single`、`pooled`、`short`。
4. 回看实验副本 `client.cpp` 中 `--connection_type` 到 `ChannelOptions::connection_type` 的赋值。

**实验 2：三种连接模式（45 分钟）**

1. 构建既有实验副本：`cmake --build build/learning-lab --target echo_server echo_client -j6`。
2. 启动服务并保存日志：

```bash
set -o pipefail
./build/learning-lab/echo_server --listen_addr=127.0.0.1:8000 \
  2>&1 | tee build/learning-records/week-04/echo-server.log
```

3. 分别运行下列客户端；每种至少发出 20 次请求后终止，记录成功数和 `/connections` 页面观察：

```bash
set -o pipefail
./build/learning-lab/echo_client --server=127.0.0.1:8000 --connection_type=single \
  --study_request_count=20 --interval_ms=50 --timeout_ms=1000 --max_retry=0 \
  2>&1 | tee build/learning-records/week-04/client-single.log
./build/learning-lab/echo_client --server=127.0.0.1:8000 --connection_type=pooled \
  --study_request_count=20 --interval_ms=50 --timeout_ms=1000 --max_retry=0 \
  2>&1 | tee build/learning-records/week-04/client-pooled.log
./build/learning-lab/echo_client --server=127.0.0.1:8000 --connection_type=short \
  --study_request_count=20 --interval_ms=50 --timeout_ms=1000 --max_retry=0 \
  2>&1 | tee build/learning-records/week-04/client-short.log
```

4. 若当前客户端固定循环次数或并发度不能形成对照，先在 `example/learning_echo_c++/client.cpp` 增加仅供学习的请求次数/并发参数，再重新构建；不要假装原示例已有这些开关。
5. 每种模式结束后保存连接页面或结构化摘录，避免只写“看起来不同”。

**成功预期**：三种模式都保持 RPC 语义正确；`single` 倾向复用单连接，`short` 倾向请求后关闭，`pooled` 的连接数量取决于受控并发与池状态。

**失败预期**：低并发下 `pooled` 与 `single` 可能观察不到明显差异，这不是失败；增加受控并发后再观察。连接失败、端口占用或服务未启动应单独记录。

**边界**：一次本机实验不能推出性能优劣；`max_connection_pool_size` 也不等于进程最大连接数。先将第 2/3 周已经实现的 `--study_fail=false`、`--study_delay_ms=0` 复位，避免前一周故障注入污染结果；尚未实现的 `--study_backup_request_ms` 不需要传入。

**观察与记录（15 分钟）**

| 模式 | 请求数/并发 | 观察到的连接数 | 是否复用 | 关闭时机证据 | 限制 |
| --- | ---: | ---: | --- | --- | --- |
| single |  |  |  |  |  |
| pooled |  |  |  |  |  |
| short |  |  |  |  |  |

## 学习单元 5（75 分钟）：针对性测试与整合

先确认测试文件存在，再执行过滤用例。`IOBufTest` 位于聚合二进制 `test_butil`，不是 `iobuf_unittest` 独立程序；Socket 测试是独立二进制。

```bash
cmake --build build/learning --target test_butil brpc_socket_unittest -j6
set -o pipefail
(
  cd build/learning/test
  ./test_butil \
    --gtest_filter='IOBufTest.copy_and_assign:IOBufTest.append_and_cut_it_all:IOBufTest.conversion_with_protobuf' \
    2>&1 | tee ../../learning-records/week-04/iobuf-tests.log
  ./brpc_socket_unittest \
    --gtest_filter='SocketTest.single_threaded_write' \
    2>&1 | tee ../../learning-records/week-04/socket-test.log
)
```

若本机构建布局不同，先用 `find build/learning -type f -perm -111` 定位产物，并在记录中说明实际路径；不要修改计划中的源码结论来迁就路径差异。

测试后完成三件事：

- 将第 3 周 RPC 时序图补全到协议 processor 输入层，标注半包与多包循环。
- 用 150–250 字解释 `Socket::Write` 为什么不能简单等同于一次成功的内核 `write`。
- 对每个测试写“它证明什么 / 它没有证明什么”，避免只粘贴 PASS。

## 常见误区

- 把一次 TCP `read` 当成一个 RPC：TCP 没有应用消息边界，解析器必须处理半包和多包。
- 把 `PARSE_ERROR_NOT_ENOUGH_DATA` 当错误响应：它通常表示保留缓冲并继续读取。
- 从图中跳过 `Transport` 或 `InputMessengerProcessor`：当前版本明确存在这两个边界。
- 把 IOBuf 的所有操作都称为零拷贝：从原始指针追加字节与共享另一个 IOBuf 的语义不同。
- 认为复制后的两个 IOBuf 不能共享 Block：公开语义是逻辑修改互不影响，底层可通过引用计数共享数据。
- 在 macOS 上寻找 epoll 运行证据，或在 Linux 上按 kqueue 标志解释事件。
- 低并发只看到一条 pooled 连接就断言连接池无效。
- 从本机 20 次请求的耗时得出生产性能结论。

## 验收清单

- [ ] 输入路径图包含 event dispatcher、Socket、transport、messenger、processor、protocol process 七个边界。
- [ ] 图中标出了 `EAGAIN`、EOF、半包和一次多消息四条分支。
- [ ] 能指出当前平台使用 epoll 还是 kqueue，并列出另一平台至少两个 API 差异。
- [ ] 能解释 `IOPortal` 为什么适合作为持续读取缓冲。
- [ ] IOBuf 实验包含长度和内容断言，B 在 A 修改后保持逻辑内容不变。
- [ ] 能区分 `append(IOBuf)` 与 `append(void*, size_t)` 的数据处理方式。
- [ ] 三种连接模式均有命令、请求条件、观测证据和限制说明。
- [ ] 三个 IOBuf 过滤测试与一个 Socket 测试已有实际结果，或记录了可复现阻塞原因。
- [ ] 所有结论写入 `notes/records/week-04.md`，原始日志未混入本计划文件。

## 复盘问题

1. 如果 baidu_std 头部已经收到但 body 尚缺 12 字节，数据保存在哪里，下一次从哪个函数继续？
2. 为什么一次事件处理可能解析多个消息，却不一定为每个消息立即创建并唤醒一个 bthread？
3. “IOBuf 复制便宜”成立需要哪些前提？什么使用方式可能长期锁住多个 Block？
4. `short` 连接减少复用后，可能增加哪些系统成本？为什么业务响应内容不应改变？
5. 如果写入遇到 fd 暂时不可写，transport、Socket 和事件机制各负责哪一部分？

## 与第 5 周衔接

本周只确认 `ProcessEvent`、`QueueMessage` 和 `ProcessInputMessage` 可能进入 bthread，不解释它们何时占用或让出 pthread worker。第 5 周从 `bthread_start_*`、`bthread_join` 和 `bthread_usleep` 开始，再进入 `TaskGroup::sched` 与 butex，把图中的虚线调度边界变成状态变化。
