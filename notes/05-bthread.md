# 第 5 周：bthread 调度、等待与唤醒

> 导航：[总计划](learning-plan.md) · [上一周：I/O 与 IOBuf](04-io-and-iobuf.md) · [下一周：路由与故障处理](06-routing-and-failures.md)

本周预算 **6–8 小时**。学习顺序固定为“创建 → 运行 → join → 协作等待 → butex”，先建立可观察的任务生命周期，再读调度器内部。不要从 butex 细节倒推所有 bthread 行为，也不要把阻塞系统调用与 bthread 的协作等待混为一谈。

本文是执行计划，不是实验记录。执行时把结论写入 `notes/records/week-05.md`，原始日志放入 `build/learning-records/week-05/`。

## 前置条件

- 已完成第 4 周输入路径图，知道网络输入处理可能由 `TcpTransport` 创建 bthread。
- 能区分逻辑任务 ID、操作系统 pthread 与进程；了解 mutex、condition variable 和原子变量的基本用途。
- 主测试构建在 `build/learning/`，实验副本在 `example/learning_echo_c++/`，构建目录为 `build/learning-lab/`。
- 运行前创建记录目录：`mkdir -p build/learning-records/week-05 notes/records`。

## 本周完成标准

完成后应能解释 `bthread_start_*` 怎样把任务交给 `TaskGroup`，用户函数和 `bthread_join` 怎样衔接，以及结束状态怎样对 joiner 可见；还要知道当前 `bthread_join` 总把输出参数置空，任务结果应通过创建时传入的共享参数传递。最后要区分 `bthread_usleep`/butex 的协作等待与阻塞 pthread/system call，并说明一次实验未观察到迁移为何不能证明 bthread 永不迁移。

本周交付物：任务状态图、bthread/pthread ID 观测表、协作等待与系统阻塞对照数据、同步 RPC 等待机制补图。

| 单元 | 主线时间 | 实验是否计入 |
| --- | ---: | --- |
| 1. 公共 API | 60 分钟 | 无 |
| 2. 创建、运行与 join | 90 分钟 | 实验 1 计入 |
| 3. sched 与等待对照 | 120 分钟 | 实验 2 计入 |
| 4. butex | 60 分钟 | 实验 3 为选修，不计主线 |
| 5. 测试与 RPC 回看 | 60 分钟 | 无 |

主线合计 **390 分钟（6.5 小时）**；选修 butex 实验约 45 分钟，可在本周尚有余量时完成。

## 源码阅读地图

| 阶段 | 文件与符号 | 阅读问题 |
| --- | --- | --- |
| 公共 API | [bthread.h](../src/bthread/bthread.h) 的 `bthread_start_*`、`join`、`usleep`、`yield` | 返回码与生命周期契约是什么？哪些 API 可从 pthread 调用？ |
| API 实现 | [bthread.cpp](../src/bthread/bthread.cpp) 的启动、join、concurrency 实现 | API 怎样取得或创建 `TaskGroup`？ |
| 创建任务 | [TaskGroup::start_foreground / start_background](../src/bthread/task_group.cpp) | urgent/background 怎样入队或切换？属性何时写入 TaskMeta？ |
| 执行任务 | [TaskGroup::task_runner](../src/bthread/task_group.cpp) | 用户函数从哪里调用？为何当前实现丢弃函数返回值？结束状态如何发布？ |
| worker | [TaskControl::worker_thread](../src/bthread/task_control.cpp) | pthread worker 怎样取得 TaskGroup 并进入运行循环？ |
| 调度 | [TaskGroup::sched / sched_to](../src/bthread/task_group.cpp) | 当前任务怎样让出 worker？下一个任务从哪里来？ |
| 等待原语 | [butex.h](../src/bthread/butex.h)、[butex_wait](../src/bthread/butex.cpp) | 值不匹配、bthread 等待、普通 pthread 等待走哪些不同路径？ |
| 线程局部状态 | [thread_local.md](../docs/cn/thread_local.md) | bthread 迁移后，缓存 pthread-local 地址为何危险？ |

先用这张状态骨架指导阅读：

```text
create
  -> TaskMeta initialized -> foreground handoff OR background runqueue
  -> task_runner calls user function
  -> running -> cooperative wait -> sched -> wake -> running
             -> blocking syscall -> current pthread worker blocked
  -> user function returns (return pointer discarded)
  -> completion signalled -> join returns (output pointer set to null)
  -> TaskMeta eventually reused
```

图中的“eventually reused”提醒你：保存一个旧 `bthread_t` 并不等于永久拥有某个 TaskMeta。

## 学习单元 1（60 分钟）：先掌握公共 API 契约

**阅读（30 分钟）**

1. 读 [threading_overview.md](../docs/cn/threading_overview.md)，建立 pthread、bthread、worker 的词汇表。
2. 读 [bthread.md](../docs/cn/bthread.md) 的 M:N、阻塞行为和常见问答。
3. 读 [bthread_or_not.md](../docs/cn/bthread_or_not.md)，记录不适合无条件替换 pthread 的情况。
4. 精读 [bthread.h](../src/bthread/bthread.h) 中启动、停止/中断、join、yield、usleep 和 concurrency 声明附近注释。

**带问题的操作（20 分钟）**

- 为 `start_urgent`、`start_background`、`join`、`usleep` 各写一行“调用者看到的契约”。
- 区分 task 的“已创建”“可运行”“正在运行”“等待”“已结束”；不要把函数返回后立刻等同于所有内部资源已销毁。
- 回答普通 pthread 调用 `bthread_usleep` 时是否必然具有和 bthread 相同的让出效果，以源码/文档为依据。
- 列出 `BTHREAD_ATTR_NORMAL` 与 `BTHREAD_ATTR_PTHREAD` 至少一个行为差异，暂不扩展到所有属性组合。

**观察与记录（10 分钟）**

- 写下三个 ID：`bthread_self()`、`pthread_self()`、进程 ID；说明它们各标识什么。
- 明确记录：“一个 bthread 可在等待恢复后运行于不同 pthread，但单次运行未迁移也正常”。
- 把仍不理解的切栈实现列为进阶问题，本周不读汇编。

## 学习单元 2（90 分钟）：创建、运行和 join

**阅读（30 分钟）**

1. 从 [bthread_start_urgent / bthread_start_background](../src/bthread/bthread.cpp) 进入 TaskGroup。
2. 对照 [TaskGroup::start_foreground / start_background](../src/bthread/task_group.cpp)，记录任务何时进入 runqueue，何时可能立即切换。
3. 在同一文件定位 `TaskGroup::task_runner`，确认用户函数的返回指针当前被显式丢弃。
4. 回到 `bthread_join` 实现，确认输出参数总被置为 `nullptr`，再找完成信号与可见性保证，并读 [BthreadTest.bthread_join](../test/bthread_unittest.cpp) 的正常与非法 ID 场景。

**带问题的操作（20 分钟）**

- 画 urgent 和 background 两条创建路径，标出“API 返回前是否倾向运行新任务”的差异，但不要写成绝对实时保证。
- 追踪用户函数的 `void*` 返回值在 `task_runner` 中被丢弃的位置，以及 `bthread_join` 将输出参数设为空的位置。
- 设计正确结果通道：创建任务前由调用方分配结果结构，通过 `arg` 传入；任务写入，join 后调用方读取。
- 解释 join 已结束任务时为什么仍需要正确的内存可见性，而不能只检查“任务不存在”。
- 找出 join 自身、无效 tid 的测试期望，并记录错误码。

**观察与记录（10 分钟）**

- 状态图至少标注 `bthread_start_*`、`start_*`、`task_runner`、`bthread_join` 四个符号。
- 将“调度顺序倾向”与“接口正确性保证”分成两列，不根据日志打印先后创造契约。

## 实验 1（计入单元 2，30 分钟）：任务 ID、worker ID 与 join

此实验需要先新增程序；计划中没有预先存在的 `bthread_lab` 可执行文件。

1. 在 `example/learning_echo_c++/` 新增 `bthread_lab.cpp`。
2. 在该目录 `CMakeLists.txt` 新增 `bthread_lab` target，沿用现有示例 target 的 brpc 配置和链接方式。
3. 主线程先分配 8 个生命周期覆盖所有任务的结果结构，通过各自 `arg` 传给 background bthread；每个任务记录开始时的 `bthread_self()`、`pthread_self()`，调用 `bthread_usleep(10 * 1000)` 后再记录一次，写入完成标志并 `return nullptr`。
4. 主线程逐个调用 `bthread_join(tid, nullptr)`，成功后再核对共享结果结构中的完成标志和 ID；不要尝试从 join 输出参数取得用户函数结果。
5. 配置、构建、运行：

```bash
set -o pipefail
cmake -S example/learning_echo_c++ -B build/learning-lab
cmake --build build/learning-lab --target bthread_lab -j6
./build/learning-lab/bthread_lab \
  2>&1 | tee build/learning-records/week-05/bthread-id-and-join.log
```

**成功预期**：8 个任务均获得非零 bthread ID，`bthread_join(tid, nullptr)` 成功，join 后共享结果均显示完成；同一任务前后的 pthread ID 可能相同，也可能不同。
**失败预期**：若共享结果结构生命周期短于任务，可能崩溃或产生错误值；若尝试用 join 输出参数接收函数返回值，当前实现只会得到空指针。
**边界**：日志只展示一次调度样本。没有观察到迁移，不得写成“bthread 绑定 pthread”；观察到迁移，也不能据此估算迁移概率。

## 学习单元 3（120 分钟）：sched、worker 与协作等待

**阅读（30 分钟）**

1. 读 [TaskControl::worker_thread](../src/bthread/task_control.cpp)，确认它运行在 pthread worker 上。
2. 读 [TaskGroup::sched / sched_to](../src/bthread/task_group.cpp)，记录保存当前任务和选择下一任务的边界。
3. 在 `task_group.cpp` 定位 `bthread_usleep` 相关实现，追踪定时器、waiter 与重新入队。
4. 回看第 4 周 `TcpTransport::ProcessEvent` 与 `QueueMessage`，解释网络处理为什么能与其他 bthread 共享 worker。

**带问题的操作（20 分钟）**

- 回答“bthread 阻塞”至少有哪两种完全不同的含义：通过 bthread API 等待，或在用户函数中调用阻塞系统/pthread API。
- 标出协作等待在哪个调用点进入 `sched`；醒来后如何重新成为可运行任务。
- 思考当一个 worker 被系统 `usleep` 占住时，其他 worker 能做什么；所有 worker 都被占住时又怎样。
- 把 work stealing 只画成“可能从其他队列取得任务”，不把一次日志现象当完整算法证明。

**观察与记录（10 分钟）**
建立对照表：

| 情况 | 当前 bthread | 当前 pthread worker | 其他可运行 bthread | 主要证据 |
| --- | --- | --- | --- | --- |
| `bthread_usleep` | 等待 | 可运行别的任务 | 可继续推进 |  |
| system `usleep` | 函数内阻塞 | 被占住 | 依赖其他 worker |  |
| `butex_wait` in bthread | 等待 | 可运行别的任务 | 可继续推进 |  |
| `butex_wait` from pthread | pthread 等待 | 当前 pthread 被等待占用 | 视其他 worker 而定 |  |

## 实验 2（计入单元 3，60 分钟）：协作等待与阻塞系统调用对照

在实验 1 的 `bthread_lab.cpp` 中新增独立模式参数，例如自行实现 `--study_wait_mode=bthread|system` 和 `--study_task_count`。这些不是仓库现成开关，必须先编码并重新构建后使用。

1. 在创建任何 bthread 前调用 `bthread_setconcurrency(8)`，检查返回值并打印实际 `bthread_getconcurrency()`；若设置失败，记录返回码并停止本对照，不在未知 worker 数下继续声称配置固定。
2. 创建明显多于 worker 数量的任务，例如 40 个；每个任务等待 100ms，然后递增完成计数。
3. `bthread` 模式调用 `bthread_usleep(100000)`；`system` 模式调用系统 `usleep(100000)`。
4. 完成计数使用原子变量；由未参与 bthread 任务的主 pthread 每 50ms 读取一次，然后再 join 全部任务，避免数据竞争或让采样器自身占用 bthread worker。
5. 每组至少运行 3 次：

```bash
set -o pipefail
for run in 1 2 3; do
  ./build/learning-lab/bthread_lab --study_wait_mode=bthread --study_task_count=40 \
    2>&1 | tee "build/learning-records/week-05/cooperative-wait-${run}.log"
  ./build/learning-lab/bthread_lab --study_wait_mode=system --study_task_count=40 \
    2>&1 | tee "build/learning-records/week-05/blocking-syscall-${run}.log"
done
```

**成功预期**：协作等待组能让 worker 继续调度其他任务；系统等待组会占住调用它的 worker，任务推进更接近按 worker 数分批。结果以完成进度和重复数据为准。
**失败预期**：若 `bthread_setconcurrency(8)` 失败，必须记录返回码和实际 concurrency，不能仍宣称“固定 8 worker”；机器负载会造成时间波动，应保留原始三次结果。
**边界**：这不是通用性能基准。系统阻塞不会神奇地阻塞所有 worker，但足够多的阻塞任务可耗尽 worker；反过来，`bthread_usleep` 的优势也不代表所有 bthread API 都无成本。

## 学习单元 4（60 分钟）：butex 等待与唤醒

**阅读（30 分钟）**

1. 先读 [butex.h](../src/bthread/butex.h) 的公开注释，注意 timeout 使用绝对时间。
2. 读 [butex_wait](../src/bthread/butex.cpp)，先看 expected value 不匹配的快速失败。
3. 分开追踪普通 pthread / pthread-task 路径 `butex_wait_from_pthread` 与 bthread 路径。
4. 在 bthread 路径找 waiter 入队、`set_remained`、`TaskGroup::sched`、超时/中断/值变化后的返回分类。
5. 浏览 `butex_wake`/`butex_wake_all`，只理解“选择 waiter 并使其可运行”，暂不研究所有竞争优化。

**带问题的操作（20 分钟）**

- 为什么 wait 前必须再次比较 expected value？值已不匹配时为何返回 `EWOULDBLOCK`？
- bthread waiter 为什么能通过 `sched` 让出 worker，而 pthread waiter 需要不同等待实现？
- timeout、interrupt、unmatched value 分别对应哪些 errno？在图中保持三条不同出口。
- `bthread_join`、condition variable、countdown event 如何可能复用 butex 概念？只找一处调用证据，不展开所有封装。

**观察与记录（10 分钟）**

- 给状态图增加 waiting queue、wake、timeout、interrupt 四个节点。
- 写一句边界结论：butex 是 bthread/pthread 可共同使用的等待原语，但调用环境决定等待路径。

## 选修实验 3（45 分钟，不计主线）：值检查与一次唤醒

在 `bthread_lab.cpp` 新增 `--study_mode=butex` 分支，并明确这是自行实现的学习代码。

1. 用 `bthread::butex_create_checked<butil::atomic<int>>()` 创建 butex，显式 `store(0)` 初始化；waiter 已经 join 后再配对 destroy。
2. 先在值为 1 时以 expected=0 调用 wait，断言立即返回 -1 且 `errno == EWOULDBLOCK`。
3. 再置 0，启动 waiter；waiter 使用正确条件循环：值仍为 0 时调用 `butex_wait(expected=0)`，接受被 wake 后返回 0，或主流程先改值导致的 `EWOULDBLOCK`，随后重新检查条件。
4. 主流程不尝试证明 waiter 已经入队；它只把值原子地改为 1 并调用 `butex_wake_all`。join waiter，断言其最终观察到值 1 并退出。
5. 构建后运行：

```bash
set -o pipefail
cmake --build build/learning-lab --target bthread_lab -j6
./build/learning-lab/bthread_lab --study_mode=butex \
  2>&1 | tee build/learning-records/week-05/butex-lab.log
```

**成功预期**：首个不匹配分支立即以 `EWOULDBLOCK` 返回；竞争分支无论“先入队后 wake”还是“先改值后 wait”都能通过条件循环结束。
**失败预期**：只 wake 而不修改条件会使循环继续等待；不处理 `EWOULDBLOCK` 会把合法竞争误判成错误。不要用固定 sleep 或额外标志声称 waiter 已真正进入内部等待队列。
**边界**：实验用于理解条件与唤醒协议，不鼓励业务直接用 butex 替代已有 mutex/condition variable；不要访问 butex 内部 waiter 容器。

## 学习单元 5（60 分钟）：测试、thread-local 与 RPC 回看

**针对性测试（25 分钟）**
`bthread_unittest.cpp` 在 CMake 中生成独立二进制 `bthread_unittest`：

```bash
cmake --build build/learning --target bthread_unittest -j6
set -o pipefail
(
  cd build/learning/test
  ./bthread_unittest \
    --gtest_filter='BthreadTest.bthread_join:BthreadTest.bthread_usleep:BthreadTest.yield_single_thread' \
    2>&1 | tee ../../learning-records/week-05/bthread-tests.log
)
```

运行前先读三个测试。注意 `BthreadTest.bthread_usleep` 自带关于前序测试残留 stealing 的注释；若偶发失败，应保存完整日志、单独重跑并分析，不能直接删除用例或宣称机制失效。

**thread-local 阅读（15 分钟）**

- 读 [thread_local.md](../docs/cn/thread_local.md) 的示例。
- 解释为何在可能切换 worker 的调用前缓存 `thread_local` 地址存在风险。
- 审视实验代码：`pthread_self()` 仅用于观测，不作为业务状态索引。

**回看同步 RPC（20 分钟）**

- 回到第 3 周 `Controller::IssueRPC`、`Join`/完成路径和 `bthread_id` 相关代码。
- 在时序图中标出同步调用者等待、响应处理任务运行、完成信号唤醒调用者三个事件。
- 用“调用者是 bthread”和“调用者是普通 pthread”两条路径解释等待差异；不要声称同步 RPC 总会阻塞一个 worker。
- 将第 4 周网络输入 bthread 与本周 worker 状态图连接起来，说明系统阻塞耗尽 worker 时为何可能拖慢 RPC 收发处理。

## 常见误区

- 把 bthread 叫成单线程 event loop：它是 M:N 调度模型，可运行在多个 pthread worker 上。
- 认为每个 bthread 固定绑定一个 pthread：等待恢复后可能迁移，一次未迁移不构成保证。
- 把 `bthread_usleep` 与系统 `usleep` 等同：前者在 bthread 环境中可让出 worker，后者占住当前 pthread。
- 认为一个阻塞系统调用会直接冻结全部 bthread：它先占一个 worker；足够多阻塞任务才可能耗尽 worker。
- 认为调用 `bthread_start_urgent` 就获得严格优先级或实时调度保证。
- 期待 `bthread_join` 返回用户函数指针：当前实现总置空；应通过创建时传入且生命周期足够长的共享参数传递结果。
- 将 butex 的 wake 当成条件本身；正确协议仍需修改条件并处理竞态与伪时序假设。
- 在可能迁移的 bthread 中长期保存 pthread-local 地址或用 `pthread_self()` 绑定业务身份。
- 看到测试 PASS 就声称验证了所有 work stealing 与公平性行为。

## 验收清单

- [ ] 状态图覆盖创建、入队、运行、协作等待、唤醒、结束和 join。
- [ ] 能沿源码讲出 `bthread_start_* -> TaskGroup::start_* -> task_runner`。
- [ ] 能说明 urgent 与 background 的可观察倾向，同时不把倾向写成实时保证。
- [ ] 实验 1 的 8 个任务全部通过 `bthread_join(tid, nullptr)` 等待，并在 join 后验证共享结果结构。
- [ ] ID 表同时记录 bthread ID 和 pthread ID，并正确解释“未迁移”样本。
- [ ] 对照实验固定并核实 worker 数，至少重复三次并保留进度数据。
- [ ] 能准确区分协作等待、阻塞系统调用以及从普通 pthread 调用 butex wait。
- [ ] 能口述 butex 值检查协议；若做选修实验，覆盖 `EWOULDBLOCK` 快速失败及“先改值/先入队”两种竞争结果。
- [ ] 三个过滤测试已有实际结果，或记录了完整且可复现的阻塞原因。
- [ ] 已用本周机制重新解释同步 RPC 等待，不再写成“同步调用必占一个 pthread”。
- [ ] 结论写入 `notes/records/week-05.md`，日志保存在指定构建记录目录。

## 复盘问题

1. background bthread 从创建到用户函数执行，至少经过哪些对象与函数？
2. join 一个已经结束的任务时，为什么仍要考虑被 join 任务写入数据的可见性？
3. 八个 worker 中一个被系统调用阻塞时，其余任务怎样继续？八个都阻塞时网络输入任务会怎样？
4. butex 值在 wait 前已经改变，为何应快速失败而不是先入队再等待一次 wake？
5. bthread 从等待中恢复到另一 pthread 后，哪些 thread-local 使用方式仍安全，哪些缓存指针方式危险？
6. 同步 RPC 的调用者是 bthread 与普通 pthread 时，等待对调度资源的影响有何不同？

## 与第 6 周衔接

第 6 周研究命名、负载均衡、timeout、retry 和 backup request。届时要把“请求何时完成”拆成网络失败、定时器触发、响应任务运行和调用者被唤醒，避免仅按墙钟时间猜测内部顺序。若后续为实验副本增加可控行为，统一使用自行实现的服务端 `--study_delay_ms`（默认 0）、`--study_fail`（默认 false）和客户端 `--study_backup_request_ms`（默认 -1）；这些参数在当前原始 Echo 中并不存在。
