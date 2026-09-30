# 第 1 周：构建环境、运行 Echo、建立项目地图

[返回总计划](learning-plan.md) · [下一周：API 与生命周期](02-api-and-lifetime.md)

本周安排 **7 小时，分 5 次完成**。目标是得到一套可以重复执行的构建、启动和测试步骤，并用自己的话解释一个最小 RPC 服务。以下命令和实验供学习时执行，本文没有预先运行它们。

## 1. 开始前的准备

需要知道 C++ 的头文件/实现文件、编译/链接、类与继承、指针与 RAII；能在终端切换目录和查看进程输出。如果 Protobuf 是新概念，本周先掌握 message、service、Stub 三者的关系。

准备三个终端：服务端、客户端、观测与测试。下文默认从仓库根目录执行命令。使用以下统一目录，后七周也沿用它们：

| 用途 | 目录或文件 |
| --- | --- |
| 框架构建 | `build/learning/` |
| 原始 Echo 构建 | `build/learning-echo/` |
| 本周命令输出 | `build/learning-records/week-01/` |
| 自己填写的学习记录 | `notes/records/week-01.md` |
| 第 2 周开始使用的实验副本 | `example/learning_echo_c++/`，构建到 `build/learning-lab/` |

周计划保留为任务说明；实验结果另写到 `notes/records/`。先运行：

```bash
mkdir -p notes/records build/learning-records/week-01
git rev-parse HEAD
git status --short
uname -s
uname -m
cmake --version
c++ --version
protoc --version
```

把版本信息与实际命令保存下来。某个命令不存在时，先记录缺失项，再对照 [构建说明](../docs/cn/getting_started.md) 补齐环境。

## 2. 本周学习单元

| 单元 | 时间 | 核心问题 | 交付物 |
| --- | --- | --- | --- |
| 1 | 60 分钟 | brpc 解决什么问题，代码在哪里？ | 一页项目地图 |
| 2 | 120 分钟 | 如何从源码得到自己的库和示例？ | 构建记录与产物清单 |
| 3 | 90 分钟 | Echo 请求如何发起、执行和返回？ | 两端日志与接口草图 |
| 4 | 90 分钟 | 如何用已有测试验证基本行为？ | 单测证据与失败场景记录 |
| 5 | 60 分钟 | 能否独立复现并讲清流程？ | 本周总结和验收清单 |

### 单元 1：认识项目边界

按顺序阅读 [README_cn.md](../README_cn.md)、[概述](../docs/cn/overview.md)，再打开下面的入口。限制在接口和目录职责，不深入内部实现。

| 阅读入口 | 要找的内容 | 写进笔记的问题 |
| --- | --- | --- |
| [echo.proto](../example/echo_c++/echo.proto) | package、message、service、`cc_generic_services` | 一次 Echo 的输入/输出是什么？ |
| [client.cpp](../example/echo_c++/client.cpp) | `ChannelOptions`、`Channel::Init`、`stub.Echo` | 服务器地址、协议和超时在哪里指定？ |
| [server.cpp](../example/echo_c++/server.cpp) | `EchoServiceImpl::Echo`、`AddService`、`Start` | 框架如何得到业务实现？ |
| [channel.h](../src/brpc/channel.h) | `Channel`、`ChannelOptions` | Channel 和某一次请求是什么关系？ |
| [controller.h](../src/brpc/controller.h) | 错误、附件、远端地址、耗时 | 为什么请求之外还需要 Controller？ |
| [server.h](../src/brpc/server.h) | Service 注册和服务器生命周期 | 业务服务与监听服务器是同一个对象吗？ |

画一张只含五个方框的草图：业务客户端、Stub、Channel/Controller、Server、业务 Service。为每条连线写一句职责。将 bthread、IOBuf、bvar 标为后续要补充的机制，不急于画内部细节。

### 单元 2：建立独立构建目录

阅读 [根 CMakeLists.txt](../CMakeLists.txt)、[库目标定义](../src/CMakeLists.txt)、[Echo CMakeLists.txt](../example/echo_c++/CMakeLists.txt)、[公共示例依赖查找](../example/cmake/BrpcExample.cmake)。只回答：依赖如何找到、目标叫什么、产物输出在哪里。

依赖至少涉及 gflags、Protobuf/protoc、OpenSSL、leveldb；具体包名按仓库的操作系统说明选择。不要同时切换 Make、CMake 和 Bazel 排查同一个错误。本计划先使用 CMake。

Linux 配置：

```bash
cmake -S . -B build/learning \
  -DCMAKE_BUILD_TYPE=Debug \
  -DWITH_DEBUG_SYMBOLS=ON \
  -DCMAKE_EXPORT_COMPILE_COMMANDS=ON \
  -DBUILD_UNIT_TESTS=ON \
  -DBUILD_BRPC_TOOLS=ON
```

macOS + Homebrew 配置，与上面的 Linux 命令二选一：

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

配置成功后构建静态库：

```bash
cmake --build build/learning --target brpc-static --parallel 6
```

**两个容易忽略的前提：** [测试配置](../test/CMakeLists.txt) 默认使用下载的 Googletest；离线时需设置 `-DDOWNLOAD_GTEST=OFF` 并提供 `-DBRPC_SYSTEM_GTEST_SOURCE_DIR=/实际/googletest/源码目录`。另外，当前公共编译选项含 `-O2`，`Debug` 不等于所有源文件均无优化，实际参数以编译数据库为准。

Linux 下配置 Echo：

```bash
cmake -S example/echo_c++ -B build/learning-echo \
  -DCMAKE_BUILD_TYPE=Debug \
  -DBRPC_INCLUDE_PATH="$PWD/build/learning/output/include" \
  -DBRPC_LIB="$PWD/build/learning/output/lib/libbrpc.a"
```

macOS 下配置 Echo：

```bash
cmake -S example/echo_c++ -B build/learning-echo \
  -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_PREFIX_PATH="$(brew --prefix)" \
  -DOPENSSL_ROOT_DIR="$(brew --prefix openssl@3)" \
  -DBRPC_INCLUDE_PATH="$PWD/build/learning/output/include" \
  -DBRPC_LIB="$PWD/build/learning/output/lib/libbrpc.a"
```

配置成功后：

```bash
cmake --build build/learning-echo --parallel 4
rg -n 'EchoService_Stub|CallMethod|class EchoService' \
  build/learning-echo/echo.pb.h build/learning-echo/echo.pb.cc
```

检查 `libbrpc.a`、导出的头文件、`echo.pb.h/.cc`、`echo_server` 和 `echo_client`。解释哪些来自手写源码，哪些来自 protoc。生成文件用于阅读，接口调整应回到 `.proto`。

### 单元 3：运行并观察一次正常调用

终端 A 启动服务：

```bash
./build/learning-echo/echo_server --listen_addr=127.0.0.1:8000
```

终端 B 启动客户端：

```bash
./build/learning-echo/echo_client \
  --server=127.0.0.1:8000 \
  --attachment=week01 \
  --timeout_ms=1000 \
  --max_retry=0 \
  --interval_ms=1000
```

至少观察五次请求，记录 `message`、attachment、`log_id`、本地/远端地址和 latency。先用 `max_retry=0` 保持行为容易解释，重试留到第 6 周。

终端 C 观测：

```bash
curl -fsS http://127.0.0.1:8000/status
curl -fsS http://127.0.0.1:8000/vars
curl -fsS http://127.0.0.1:8000/connections
```

这些页面也可在浏览器打开。对照 [内置服务](../docs/cn/builtin_service.md)，分别找出已注册服务、一个统计量和活动连接。把页面上的一个现象与客户端/服务端日志联系起来；页面可访问本身不能代替 RPC 成功验证。

### 单元 4：做两个边界实验，运行第一个单测

**实验 A：区分 message 与 attachment。** 保持客户端的 `--attachment=week01`。停止自己启动的服务，在终端 A 以 `--echo_attachment=false` 重启，再观察客户端。预期 message 仍回显，响应 attachment 为空；解释它们由 [Echo 实现](../example/echo_c++/server.cpp) 中哪两段代码分别处理。实验后恢复原设置。

**实验 B：服务暂不可用。** 保持客户端 `max_retry=0`，对终端 A 的服务按 Ctrl-C，观察至少一次失败后重新启动。记录失败文本与恢复时间。说明“连接失败”与“服务端收到请求后处理失败”不是同一个位置的错误；此处不把固定错误码或毫秒级恢复时间当成跨环境保证。

两个程序都是循环运行，结束时分别 Ctrl-C 退出。不要只关闭窗口而遗漏后台实验进程。

先阅读 [ChannelTest.success](../test/brpc_channel_unittest.cpp) 的设置和断言，再执行：

```bash
cmake --build build/learning --target brpc_channel_unittest --parallel 6
(
  cd build/learning/test
  ./brpc_channel_unittest --gtest_list_tests
  ./brpc_channel_unittest --gtest_filter='ChannelTest.success'
)
```

确认输出确实运行 `ChannelTest.success`，结果为通过；过滤到零个用例不算完成。记录该用例的准备、触发、断言、清理四部分，并说明它比手动 Echo 多验证了什么。

### 单元 5：复现与闭卷复盘

打开一个新终端，仅依赖自己的记录完成启动、请求、观测和停止。如果某一步需要凭记忆补参数，就补进记录。

闭卷回答：

1. 为什么 `.proto` 不是 TCP 包本身？Stub 在哪一步调用 Channel？
2. Channel、Controller、Service 分别负责什么？一个请求的结果从哪里读取？
3. 客户端把 `done` 设为 `nullptr` 表达什么？这个问题在第 2 周还需要补哪些细节？
4. 为什么静态库已构建成功，Echo 仍可能链接失败？该检查哪份配置？
5. 单测二进制在哪里，为什么先切到构建的 `test/` 目录执行？

## 3. 记录模板与完成标准

在 `notes/records/week-01.md` 填写以下信息：

| 项目 | 需要填写的证据 |
| --- | --- |
| 环境 | Git revision、OS/架构、编译器、CMake、protoc、所用依赖路径/版本 |
| 构建 | 完整配置/构建命令、成功产物、遇到的第一个实质错误及修复办法 |
| 正常调用 | 一对可关联的请求/响应日志，message 与 attachment 的观察 |
| 边界实验 | 配置改动、预期、实际结果、对应源码位置 |
| 测试 | 目标、过滤条件、执行用例数、结果 |
| 理解 | 五方框地图、尚未解释的三个问题 |

- [ ] 构建参数和依赖路径可追溯，Echo 确认链接到本次构建的 brpc。
- [ ] 能解释 protoc 生成物与手写业务代码的分工。
- [ ] 正常调用、关闭附件回显、停止后恢复服务三个场景均有记录。
- [ ] `ChannelTest.success` 实际运行并通过；如有阻塞，记录为未完成而非通过。
- [ ] 能在 5 分钟内讲清最小 Echo 的结构，并独立停止自己的实验进程。

## 4. 遇到问题时如何缩小范围

| 现象 | 优先检查 |
| --- | --- |
| CMake 找不到依赖 | `CMakeCache.txt` 中 include/lib 路径，主库与示例是否使用同一套依赖 |
| Protobuf 相关编译/链接错误 | protoc 与链接库版本、CPU 架构、是否误用其他构建生成的头文件 |
| 测试配置卡在下载 | Googletest 下载条件；可先关单测跑 Echo，但本周测试验收仍需补齐 |
| 服务启动失败 | 8000 端口是否被占用；改用空闲端口时同步修改客户端和页面地址 |
| RPC 成功但调试信息不清楚 | 确认调试符号、实际优化参数和正在运行的二进制路径 |
| 一次错误后反复换构建系统 | 留在 CMake 路线，先区分 configure、compile、link 或 runtime 阶段 |

下一周会复制四个 Echo 源文件到学习副本，在那里开展异步调用、所有权和失败分支实验；本周保留一份可复现的原始 Echo 基线。
