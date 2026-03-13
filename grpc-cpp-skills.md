# gRPC C++ 核心源码技能整理

> 基于 [grpc/grpc](https://github.com/grpc/grpc) 仓库 C++ core 源码分析，涵盖架构、构建系统、异步编程模型及开发规范。

---

## 一、仓库结构概览

| 目录 | 内容 |
|------|------|
| `src/core/` | 共享 C++ 核心库（所有语言绑定的基础） |
| `src/cpp/` | C++ 公共 API 实现 |
| `include/grpc/` | C 公共头文件（稳定 ABI） |
| `include/grpc++/` `include/grpcpp/` | C++ 公共头文件 |
| `src/{ruby,python,php,csharp,objective-c}/` | 各语言绑定（基于 core 库） |
| `test/core/` `test/cpp/` | 对应 src 的测试 |
| `examples/` | 各语言示例代码 |
| `tools/buildgen/` | 构建文件生成器（模板 → 生成文件） |
| `tools/distrib/` | 格式化、检查、发布脚本 |
| `tools/run_tests/` | 跨语言测试运行器 |
| `third_party/` | 依赖库（Bazel 模式下不需要手动管理） |

**Python 实现的特殊约束：** 不能依赖 protobuf 库，任何共享库必须暴露 C 风格 API，不使用 C++ protobuf 类型。

---

## 二、构建系统

### 1. Bazel（主要构建系统，推荐）

Bazel 是 gRPC 首选构建系统，不需要手动初始化 submodule（Bazel 自动管理依赖）。

```sh
# 构建全部
bazel build :all

# 调试模式运行所有 C/C++ 测试
bazel test --config=dbg //test/...

# 构建 core 子树
bazel build //src/core/...

# 运行单个测试
bazel test //test/core/end2end:end2end_http2_test
bazel test //test/cpp/end2end:end2end_test

# Bazel 7+ 必须加此 flag（gRPC 尚未完全兼容 bzlmod）
bazel test --enable_bzlmod=false //test/...
```

### 2. CMake

```sh
# 需要先初始化 submodule
git submodule update --init

mkdir -p cmake/build && cd cmake/build
cmake -DCMAKE_CXX_STANDARD=17 ../..
make -j$(nproc)
```

CMake 依赖管理：通过 `gRPC_<depname>_PROVIDER` 变量控制，`module` 表示从 submodule 构建，`package` 表示使用系统已安装版本。

### 3. 跨语言测试运行器

```sh
# 构建并测试指定语言
python tools/run_tests/run_tests.py -l c++
python tools/run_tests/run_tests.py -l python -c dbg

# 仅构建不运行测试
python tools/run_tests/run_tests.py -l c++ --build_only

# 使用 Docker（包含所有依赖，推荐 CI 环境）
python tools/run_tests/run_tests.py -l c++ --use_docker
```

---

## 三、Bazel BUILD 规范

### 3.1 核心宏

gRPC 使用自己的 Bazel 宏，不直接使用 `cc_library` / `cc_test`：

```python
# 每个目录必须有 grpc_package 声明
grpc_package(name = "//src/core/lib/foo")

# 库
grpc_cc_library(
    name = "my_lib",
    srcs = ["my_lib.cc"],
    hdrs = ["my_lib.h"],
    external_deps = [
        "absl/log:log",   # 非 gRPC 依赖放在 external_deps
        "gtest",          # gtest 外部依赖同时包含 gmock
    ],
    deps = [":other_grpc_lib"],
)

# 测试
grpc_cc_test(
    name = "my_test",
    srcs = ["my_test.cc"],
    external_deps = ["gtest"],
    deps = [":my_lib"],
)

# Fuzz 测试（用 fuzztest_main 而非 gtest_main）
grpc_cc_test(
    name = "my_fuzz_test",
    srcs = ["my_fuzz_test.cc"],
    external_deps = ["fuzztest", "fuzztest_main"],
)
```

### 3.2 关键规则

- **BUILD 文件位置**：实现代码的 BUILD 定义在 `src/core/BUILD` 和根 `BUILD`，不在 `src/` 子目录下新建 BUILD 文件。
- **测试 BUILD 文件**：在 `test/core/` 和 `test/cpp/` 中有各自的 BUILD 文件。
- **`grpc_proto_library` 命名**：依赖时直接用 `name` 属性值，不是 `[name]_cc_proto`（与标准 Bazel `cc_proto_library` 不同）。
- **`upb` 相关规则**：`grpc_upb_proto_library` 和 `grpc_upb_proto_reflection_library` 必须定义在**根 `BUILD` 文件**，不能放在 proto 所在子目录的 BUILD 文件中。
- **`:grpc` target**：不允许直接或间接依赖 C++ protobuf 库。
- **未使用的具名参数**：编译报错，必须用 `(void)param_name;` 或省略参数名。

### 3.3 核心端到端测试宏

```python
# test/core/end2end/BUILD
grpc_core_end2end_test_suite(
    name = "end2end_http2",           # → 生成 target: end2end_http2_test
    config = "end2end_http2_config.cc",
    # 测试实现文件在 test/core/end2end/tests/ 中
)
```

---

## 四、Core 库架构

### 4.1 整体分层

```
C++ API (include/grpcpp/, src/cpp/)
       ↓
C Surface (include/grpc/, src/core/lib/surface/)
       ↓
Channel / Call / Server (src/core/call/, src/core/server/, src/core/client_channel/)
       ↓
Filter Chain (src/core/filter/, src/core/ext/filters/)
       ↓
Transport (src/core/transport/, src/core/ext/transport/)
       ↓
Event Engine (src/core/lib/event_engine/)  ← OS I/O 抽象层
```

### 4.2 核心模块说明

| 模块 | 路径 | 职责 |
|------|------|------|
| **Call** | `src/core/call/` | RPC 生命周期的核心数据结构 |
| **Server** | `src/core/server/` | 服务端实现 |
| **ClientChannel** | `src/core/client_channel/` | 客户端 Channel：名称解析、负载均衡、连接管理 |
| **Resolver** | `src/core/resolver/` | 可插拔名称解析（DNS、xDS 等） |
| **LoadBalancing** | `src/core/load_balancing/` | 负载均衡策略框架（pick_first、round_robin、xDS） |
| **Filter** | `src/core/filter/` | Channel 过滤器机制基础设施 |
| **ext/filters** | `src/core/ext/filters/` | 具体过滤器实现（认证、压缩、重试等） |
| **Transport** | `src/core/transport/` | 传输层抽象接口 |
| **ext/transport** | `src/core/ext/transport/` | 具体传输实现（chttp2 为主，chaotic_good 为实验性） |
| **Promise** | `src/core/lib/promise/` | 异步编程框架 |
| **EventEngine** | `src/core/lib/event_engine/` | OS I/O 和线程原语抽象 |
| **Handshaker** | `src/core/handshaker/` | 安全连接建立框架 |
| **Credentials/TSI** | `src/core/credentials/`, `src/core/tsi/` | 安全凭证和 TLS 抽象 |
| **xDS** | `src/core/xds/` | Envoy xDS API 实现 |
| **Channelz** | `src/core/channelz/` | Channel 状态检查/内省系统 |
| **ServiceConfig** | `src/core/service_config/` | 每服务/每方法 Channel 配置 |
| **Telemetry** | `src/core/telemetry/` | 指标收集与上报 |

---

## 五、Promise 异步编程框架

### 5.1 核心概念

Promise 是 gRPC Core 异步编程的核心，定义在 `src/core/lib/promise/`。

**Promise 本质**：一个返回 `Poll<T>` 的可调用对象（functor）。

```cpp
// Poll<T> 有两种状态
Poll<int> result = my_promise();
if (result.pending()) {
    // 尚未完成，等待被重新唤醒
} else {
    int value = std::move(result.value());
    // 完成，拿到结果
}
```

**Activity**：Promise 的执行上下文，负责驱动 Promise 运行到完成，并在 Promise 就绪时重新唤醒它。

**Party**：允许多个 Promise 并发（非并行）执行的工具。

### 5.2 常用组合子（Combinators）

```cpp
// Seq：顺序执行（前一个完成后执行下一个）
auto seq = Seq(
    ReadFromNetwork(),         // 先读取
    [](Buffer buf) {
        return ProcessBuffer(std::move(buf));  // 处理结果
    }
);

// TrySeq：顺序执行，遇错停止
auto try_seq = TrySeq(
    Authenticate(),
    [](AuthToken token) { return SendRequest(token); },
    [](Response resp) { return HandleResponse(resp); }
);

// Join：并发等待所有完成
auto join = Join(ReadMetadata(), ReadBody());
// 返回 tuple<MetadataResult, BodyResult>

// TryJoin：任一失败则整体失败
auto try_join = TryJoin(CheckAuth(), CheckQuota());

// Race：取最先完成的
auto race = Race(Timeout(deadline), WaitForResponse());

// Loop：循环执行直到返回非 Continue 值
auto loop = Loop([i = 0]() mutable -> LoopIteration<int> {
    if (i++ < 10) return Continue{};
    return i;
});

// Map：同步转换结果
auto mapped = Map(ReadInt(), [](int x) { return x * 2; });
```

### 5.3 错误类型选择

gRPC Core 不使用 C++ 异常，通过返回值传递错误：

| 错误类型 | 适用场景 |
|----------|---------|
| `bool` | 简单成功/失败 |
| `absl::Status` / `absl::StatusOr<T>` | 跨层通用错误（fallback 首选） |
| `StatusFlag` / `ValueOrError<T>` | Promise 库内部（可被 Promise 库识别） |
| 自定义错误类型 | 特定场景；建议同时提供转换到 `absl::Status` 的方法 |

### 5.4 同步原语

```
inter_activity_latch.h   — 跨 Activity 同步（Latch）
inter_activity_mutex.h   — 跨 Activity 互斥锁
inter_activity_pipe.h    — 跨 Activity 管道通信
latch.h                  — 单 Activity 内 Latch
promise_mutex.h          — 单 Activity 内互斥锁
pipe.h                   — 单 Activity 内管道
mpsc.h                   — 多生产者单消费者队列
```

**注意**：跨 Activity 和单 Activity 同步原语不可混用，否则可能死锁。

---

## 六、Channel Filter 机制

### 6.1 Filter 架构

Filter 是拦截和修改 RPC 的机制，排列成栈（stack），每个 Filter 可将 RPC 传递给下一个或终止 RPC。

```
Client Request →
  [AuthFilter] → [CompressionFilter] → [RetryFilter] → Transport
                                                              ↓
Client Response ←                                      Server
```

### 6.2 Blackboard 共享状态

`Blackboard` 是 Filter 间共享状态的键值存储，避免紧耦合但应谨慎使用：

```cpp
// 在 FilterArgs 中访问 Blackboard
auto* blackboard = filter_args.GetBlackboard();
blackboard->Set<MyKey>(my_value);
auto value = blackboard->Get<MyKey>();
```

### 6.3 新旧架构对比

| 特性 | 旧（Callback-based Filter）| 新（Promise-based Interceptor）|
|------|---------------------------|-------------------------------|
| 代表 | `retry_filter.h` | `retry_interceptor.h` |
| 编程模型 | 回调链 | Promise 组合 |
| 状态 | 活跃迁移中（新功能用 Interceptor）|

---

## 七、ClientChannel 架构

### 7.1 核心组件职责

```
ClientChannel
 ├─ Resolver：将 URI → 后端地址列表（支持 dns:///, xds:///, 等）
 ├─ LoadBalancingPolicy：从 Subchannel 列表中为每个 RPC 选择一个
 │    (pick_first / round_robin / grpclb / xds_cluster_manager 等)
 ├─ Subchannel：到单个后端地址的连接管理
 │    └─ Connector：建立传输层连接
 └─ ConfigSelector：为 RPC 选择合适的 ServiceConfig
```

### 7.2 线程安全

`ClientChannel` 使用 `WorkSerializer` 确保内部状态的线程安全访问，所有内部操作都在 WorkSerializer 的串行执行上下文中进行。

### 7.3 Subchannel 复用

全局 SubchannelPool（`global_subchannel_pool.h`）允许多个 Channel 复用到同一后端的连接，通过地址和 ChannelArgs 的组合作为缓存 key。

---

## 八、代码规范

### 8.1 类型偏好

```cpp
// ✅ 优先使用 std 类型
std::optional<int> maybe_val;
std::string_view view;

// ✅ std 没有时用 absl
absl::flat_hash_map<K, V> map;

// ✅ 用 std::optional，不用 absl::optional
std::optional<T> opt;

// ❌ 不用 C++20+ 特性
// std::format, std::ranges, std::jthread 等
```

### 8.2 日志

```cpp
// ✅ 正确
#include "absl/log/log.h"
LOG(ERROR) << "Something went wrong: " << status;
LOG(INFO) << "Connected to " << address;

// ❌ 错误
std::cerr << "error";
gpr_log(GPR_ERROR, "error");
```

### 8.3 头文件包含顺序

```cpp
// 1. 对应头文件
#include "src/core/foo/foo.h"

// 2. 系统头文件
#include <string>
#include <vector>

// 3. Abseil 头文件（字母序）
#include "absl/log/log.h"
#include "absl/status/status.h"

// 4. gRPC 头文件（字母序）
#include "src/core/bar/bar.h"
```

### 8.4 C 公共 API 命名规则

```c
// 非静态函数必须加 grpc_ 前缀
grpc_channel_create(...)

// 静态函数不加前缀
static void helper_func(...)

// 头文件中的 struct/union/enum 类型名加 grpc_ 前缀
typedef struct grpc_channel grpc_channel;

// 头文件中的枚举值和宏定义大写，加 GRPC_ 前缀
#define GRPC_MAX_MESSAGE_SIZE (4 * 1024 * 1024)
typedef enum { GRPC_STATUS_OK = 0, GRPC_STATUS_CANCELLED = 1 } grpc_status_code;

// 多词标识符用下划线，不用驼峰
grpc_channel_create_custom
```

### 8.5 返回类型偏好

```cpp
// ✅ 优先显式结构体
struct ConnectionResult {
    bool connected;
    absl::Status status;
};

// ❌ 避免 std::pair / std::tuple 作为返回类型
std::pair<bool, absl::Status> result;
```

---

## 九、代码格式化与一致性维护

### 9.1 关键命令

```sh
# 一键再生成 + 全量格式化（提交前必须运行）
tools/distrib/sanitize.sh

# 仅格式化 C/C++ 代码（基于 clang-format）
tools/distrib/clang_format_code.sh

# 仅格式化 Python 代码
tools/distrib/black_code.sh     # 格式
tools/distrib/isort_code.sh     # import 排序
tools/distrib/ruff_code.sh --fix # lint + fix

# 仅格式化 Bazel BUILD 文件（buildifier）
tools/distrib/buildifier_format_code.sh

# 重新生成实验性功能代码
tools/distrib/gen_experiments_and_format.sh
```

### 9.2 生成文件注意事项

`src/`、`include/`、`CMakeLists.txt` 等大量文件由 `templates/` 中的模板通过 `tools/buildgen/generate_projects.sh` 生成，**不要直接编辑生成文件**。

如需修改，步骤为：
1. 修改 `templates/` 中对应的模板文件
2. 运行 `tools/buildgen/generate_projects.sh`
3. 或直接运行 `tools/distrib/sanitize.sh`（包含上一步）

---

## 十、测试体系

### 10.1 目录结构

```
test/
├── core/              # Core 库测试（对应 src/core/）
│   ├── end2end/       # 端到端测试（使用 grpc_core_end2end_test_suite 宏）
│   ├── promise/       # Promise 库测试
│   ├── transport/     # 传输层测试
│   └── ...
└── cpp/               # C++ API 测试（对应 src/cpp/）
    ├── end2end/
    ├── security/
    └── ...
```

### 10.2 单个测试运行

```sh
# Bazel（推荐）
bazel test //test/core/promise:... --config=dbg
bazel test //test/core/end2end:end2end_http2_test

# 跨语言测试脚本
python tools/run_tests/run_tests.py -l c++ -t "test_name_pattern"
```

### 10.3 端到端测试宏

`grpc_core_end2end_test_suite` 通过组合配置文件（如 `end2end_http2_config.cc`）与 `test/core/end2end/tests/` 中的独立测试实现文件生成多个 `grpc_cc_test` target。宏的 `name` 属性加 `_test` 后缀为最终 target 名。

---

## 十一、公共头文件结构

```
include/
├── grpc/              # C 公共 API（稳定 ABI，须 C89 兼容，须支持 C++ extern "C" 包装）
│   ├── grpc.h         # 核心 API
│   ├── grpc_security.h
│   └── event_engine/  # Event Engine 公共接口
├── grpc++/            # C++ API（旧前缀，保留兼容性）
└── grpcpp/            # C++ API（新前缀，推荐使用）
    ├── channel.h
    ├── server.h
    └── security/
```

**公共头文件规则：**
- `include/grpc/` 下的头文件须以 pedantic C89 编译通过
- 须可被 C++ 程序包含（用 `extern "C"` 包裹）
- 头文件须自包含并有 `#define` 防重复包含保护（格式：路径转大写 + 下划线）

---

## 十二、关键路径速查

| 功能 | 路径 |
|------|------|
| Core 架构总览 | `src/core/GEMINI.md` |
| Promise 库 | `src/core/lib/promise/` |
| Event Engine | `src/core/lib/event_engine/` |
| 客户端 Channel | `src/core/client_channel/client_channel.h` |
| Subchannel | `src/core/client_channel/subchannel.h` |
| 服务端核心 | `src/core/server/server.h` |
| HTTP/2 传输 | `src/core/ext/transport/chttp2/` |
| TLS 握手 | `src/core/tsi/` |
| xDS 实现 | `src/core/xds/` |
| 负载均衡接口 | `src/core/load_balancing/lb_policy.h` |
| 解析器接口 | `src/core/resolver/resolver.h` |
| C 公共 API 入口 | `include/grpc/grpc.h` |
| C++ API 入口 | `include/grpcpp/grpcpp.h` |
| Bazel 宏定义 | `bazel/grpc_build_system.bzl` |
| 格式化脚本 | `tools/distrib/sanitize.sh` |
| 测试运行器 | `tools/run_tests/run_tests.py` |
