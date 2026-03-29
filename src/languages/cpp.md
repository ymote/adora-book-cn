> **[English](../../languages/cpp.md)**

# C++ API 参考

Adora 通过 [CXX](https://cxx.rs/)（Rust-C++ 互操作）为独立节点和进程内算子提供 C++ 绑定。CXX 桥接从 Rust 定义生成类型安全的 C++ 头文件 -- 无需原始 FFI 或手动 `extern "C"` 声明。

| Crate | 库 | 用途 |
|-------|-----|------|
| `adora-node-api-cxx` | `libadora_node_api_cxx.a` | 独立节点可执行文件 |
| `adora-operator-api-cxx` | `libadora_operator_api_cxx.a` | 由运行时加载的共享库算子 |

---

## 节点 API（`adora-node-api-cxx`）

### 初始化

```cpp
#include "adora-node-api.h"

// 从 Adora daemon 设置的环境变量初始化节点。
AdoraNode init_adora_node();
```

### AdoraNode

```cpp
struct AdoraNode {
    rust::Box<Events>        events;       // 事件流（阻塞接收器）
    rust::Box<OutputSender>  send_output;  // 输出发送器
};
```

### 事件类型

```cpp
enum class AdoraEventType : uint8_t {
    Stop,             // 请求优雅关闭
    Input,            // 输入上有新数据到达
    InputClosed,      // 单个输入被关闭
    Error,            // 发生错误
    Unknown,          // 无法识别的事件变体
    AllInputsClosed,  // 所有输入关闭（流结束）
};
```

### 发送输出

```cpp
// 发送原始字节
AdoraResult send_output(rust::Box<OutputSender>& sender, rust::String id, rust::Slice<const uint8_t> data);

// 发送 Arrow 数组
AdoraResult send_arrow_output(rust::Box<OutputSender>& sender, rust::String id, uint8_t* array_ptr, uint8_t* schema_ptr);

// 发送日志
AdoraResult log_message(const rust::Box<OutputSender>& sender, rust::String level, rust::String message);
```

### 元数据

```cpp
rust::Box<Metadata> new_metadata();

// 读取
uint64_t     Metadata::timestamp() const;
rust::String Metadata::get_str(const rust::Str key) const;
int64_t      Metadata::get_int(const rust::Str key) const;

// 写入
void Metadata::set_string(const rust::Str key, rust::String value);
void Metadata::set_int(const rust::Str key, int64_t value);
```

---

## 算子 API（`adora-operator-api-cxx`）

您必须提供一个 `operator.h` 头文件和实现文件：

```cpp
#pragma once
#include <memory>
#include "adora-operator-api.h"

class Operator {
public:
    Operator();
};

std::unique_ptr<Operator> new_operator();

AdoraOnInputResult on_input(
    Operator& op,
    rust::Str id,
    rust::Slice<const uint8_t> data,
    OutputSender& output_sender);
```

---

## 快速入门：节点示例

```cpp
#include "adora-node-api.h"
#include <iostream>
#include <vector>

int main() {
    auto adora_node = init_adora_node();
    unsigned char counter = 0;

    for (;;) {
        auto event = next_event(adora_node.events);
        auto ty = event_type(event);

        if (ty == AdoraEventType::AllInputsClosed || ty == AdoraEventType::Stop)
            break;

        if (ty == AdoraEventType::Input) {
            auto input = event_as_input(std::move(event));
            counter += 1;
            std::vector<unsigned char> out{counter};
            rust::Slice<const uint8_t> slice{out.data(), out.size()};
            send_output(adora_node.send_output, "counter", slice);
        }
    }
    return 0;
}
```

## 构建集成（CMake）

推荐使用 CMake 和 `DoraTargets.cmake` 辅助文件进行构建。

```cmake
cmake_minimum_required(VERSION 3.21)
project(my-dataflow LANGUAGES C CXX)
set(CMAKE_CXX_STANDARD 20)

include(DoraTargets.cmake)
link_directories(${adora_link_dirs})

add_executable(my_node node/main.cc ${node_bridge})
add_dependencies(my_node Adora_cxx)
target_include_directories(my_node PRIVATE ${adora_cxx_include_dir})
target_link_libraries(my_node adora_node_api_cxx)
```

### 要求

- C++20 编译器
- Rust 工具链
- CMake 3.21+
- 如需 Arrow 集成：Apache Arrow C++ 库
