> **[English](../../languages/rust.md)**

# Rust API 参考

本文档涵盖用于构建 Adora 数据流组件的两个主要 Rust crate：

- **`adora-node-api`** -- 用于独立节点可执行文件
- **`adora-operator-api`** -- 用于由 Adora 运行时管理的进程内算子

---

## 节点 API（`adora-node-api`）

添加到您的 `Cargo.toml`：

```toml
[dependencies]
adora-node-api = { workspace = true }
```

### AdoraNode

发送输出和检索节点信息的主要结构体。通过下面的初始化函数之一获取。

#### 初始化

```rust
// 推荐：自动检测环境（daemon、测试或交互模式）。
pub fn init_from_env() -> NodeResult<(Self, EventStream)>

// 用于动态节点：通过节点 ID 连接到 daemon。
pub fn init_from_node_id(node_id: NodeId) -> NodeResult<(Self, EventStream)>

// 独立交互模式（在终端提示输入）。
pub fn init_interactive() -> NodeResult<(Self, EventStream)>

// 集成测试模式。
pub fn init_testing(
    input: TestingInput,
    output: TestingOutput,
    options: TestingOptions,
) -> NodeResult<(Self, EventStream)>
```

`init_from_env` 是推荐的入口点。它按顺序检查：

1. 由 `setup_integration_testing` 设置的线程本地测试状态
2. `ADORA_NODE_CONFIG` 环境变量（由 daemon 设置）
3. `ADORA_TEST_WITH_INPUTS` 环境变量（基于文件的集成测试）
4. 交互式终端回退（仅当 stdin 是 TTY 时）

#### 发送输出

所有发送方法对未在数据流 YAML 中声明的输出 ID 静默忽略。

```rust
// 发送 Arrow 数组。当有益时将数据复制到共享内存。
pub fn send_output(
    &mut self,
    output_id: DataId,
    parameters: MetadataParameters,
    data: impl Array,
) -> NodeResult<()>

// 发送原始字节。
pub fn send_output_bytes(
    &mut self,
    output_id: DataId,
    parameters: MetadataParameters,
    data_len: usize,
    data: &[u8],
) -> NodeResult<()>
```

#### Service、Action 和 Streaming 辅助函数

用于[通信模式](../concepts/patterns.md)的高级方法。

```rust
// 发送 service 请求。注入 request_id 到参数并返回。
pub fn send_service_request(
    &mut self,
    output_id: DataId,
    parameters: MetadataParameters,
    data: impl Array,
) -> NodeResult<String>

// 发送 service 响应。
pub fn send_service_response(
    &mut self,
    output_id: DataId,
    parameters: MetadataParameters,
    data: impl Array,
) -> NodeResult<()>
```

#### 日志

```rust
// 结构化日志。级别："error"、"warn"、"info"、"debug"、"trace"。
pub fn log(&self, level: &str, message: &str, target: Option<&str>)

// 便捷方法。
pub fn log_error(&self, message: &str)
pub fn log_warn(&self, message: &str)
pub fn log_info(&self, message: &str)
pub fn log_debug(&self, message: &str)
pub fn log_trace(&self, message: &str)
```

---

### EventStream

传入事件的异步迭代器。实现 `futures::Stream` trait。

事件流在收到 `Stop` 事件后自行关闭。节点应在流结束后退出。

```rust
// 阻塞直到下一个事件到达。
pub fn recv(&mut self) -> Option<Event>

// 带超时阻塞。
pub fn recv_timeout(&mut self, dur: Duration) -> Option<Event>

// 异步接收。
pub async fn recv_async(&mut self) -> Option<Event>

// 非阻塞接收。
pub fn try_recv(&mut self) -> Result<Event, TryRecvError>
```

---

### Event

表示传入事件。此枚举是 `#[non_exhaustive]` 的。

```rust
#[non_exhaustive]
pub enum Event {
    Input { id: DataId, metadata: Metadata, data: ArrowData },
    InputClosed { id: DataId },
    InputRecovered { id: DataId },
    NodeRestarted { id: NodeId },
    Stop(StopCause),
    Reload { operator_id: Option<OperatorId> },
    Error(String),
}
```

---

## 算子 API（`adora-operator-api`）

算子是由 Adora 运行时管理的进程内组件。编译为共享库（`.so`/`.dylib`/`.dll`）。

```toml
[dependencies]
adora-operator-api = { workspace = true }

[lib]
crate-type = ["cdylib"]
```

### AdoraOperator Trait

```rust
pub trait AdoraOperator: Default {
    fn on_event(
        &mut self,
        event: &Event,
        output_sender: &mut AdoraOutputSender,
    ) -> Result<AdoraStatus, String>;
}
```

### AdoraStatus

```rust
pub enum AdoraStatus {
    Continue,  // 继续运行
    Stop,      // 停止此算子
    StopAll,   // 停止整个数据流
}
```

### register_operator! 宏

生成 Adora 运行时加载和调用算子所需的 FFI 入口点。

```rust
use adora_operator_api::register_operator;
register_operator!(MyOperator);
```

---

## 快速入门示例：节点

一个接收 `tick` 输入并发送随机数作为输出的最小节点。

```rust
use adora_node_api::{AdoraNode, Event, IntoArrow, adora_core::config::DataId};

fn main() -> eyre::Result<()> {
    let (mut node, mut events) = AdoraNode::init_from_env()?;
    let output = DataId::from("random".to_owned());

    while let Some(event) = events.recv() {
        match event {
            Event::Input { id, metadata, data } => {
                if id.as_str() == "tick" {
                    let value: u64 = fastrand::u64(..);
                    node.send_output(output.clone(), metadata.parameters, value.into_arrow())?;
                }
            }
            Event::Stop(_) => {}
            _ => {}
        }
    }
    Ok(())
}
```
