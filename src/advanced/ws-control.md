> 🌐 **[English](../../advanced/ws-control.md)**

# WebSocket 控制平面

Adora 的控制平面使用 WebSocket 连接进行 CLI、coordinator 和 daemon 之间的所有通信。单个 Axum 服务器在一个端口上暴露三个路由，取代了之前的多端口 TCP 设计。JSON 文本帧承载 UUID 关联的请求-应答协议，以及用于日志流的即发即忘事件。

## 功能概览

| 功能 | 详情 |
|------|------|
| 路由 | `/api/control`（CLI）、`/api/daemon`（daemon）、`/health` |
| 传输格式 | JSON 文本帧 + 主题数据的二进制帧 |
| 协议 | UUID 关联的请求-应答 + 即发即忘事件 |
| 消息大小限制 | 1 MiB（`MAX_CONTROL_MESSAGE_BYTES`） |
| 并发限制 | 256 连接（`MAX_WS_CONNECTIONS`） |
| 服务器框架 | Axum + Tower 中间件 |
| 客户端库 | `tokio-tungstenite`（集成测试、daemon）、自定义 `WsSession`（CLI） |
| 安全性 | 重注册防护、daemon ID 验证、machine ID 长度限制 |

---

## 架构

```
                        单一 Axum 服务器（一个端口）
                       ┌────────────────────────────┐
                       │  /api/control   (CLI)       │
  CLI ──── WS ────────>│  /api/daemon    (Daemon)    │
                       │  /health        (HTTP GET)  │
  Daemon ── WS ───────>│                             │
                       └──────────┬─────────────────┘
                                  │ mpsc::Sender<Event>
                                  v
                            Coordinator
                          (事件循环)
```

coordinator 绑定单个 `TcpListener` 并提供 Axum 路由。每个 WebSocket 升级生成一个处理器任务，通过 `mpsc::Sender<Event>` 通道与 coordinator 的主事件循环通信。

### 关键源文件

| 文件 | 角色 |
|------|------|
| `binaries/coordinator/src/ws_server.rs` | 路由、`serve()`、常量、`ShutdownTrigger` |
| `binaries/coordinator/src/ws_control.rs` | `/api/control` 处理器 |
| `binaries/coordinator/src/ws_daemon.rs` | `/api/daemon` 处理器、安全性、事件转换 |
| `binaries/cli/src/ws_client.rs` | `WsSession` 同步客户端包装器 |
| `libraries/message/src/ws_protocol.rs` | `WsRequest`、`WsResponse`、`WsEvent`、`WsMessage` 类型 |

---

## 传输协议

所有消息都是 JSON 文本帧。存在三种消息形式：

### WsRequest（客户端 -> 服务器）

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "method": "control",
  "params": { "List": null }
}
```

| 字段 | 类型 | 描述 |
|------|------|------|
| `id` | UUID | 用于应答关联的唯一请求标识符 |
| `method` | string | CLI 请求为 `"control"`，daemon 为 `"daemon_event"` / `"daemon_command"` |
| `params` | object | 序列化的 `ControlRequest` 或 `Timestamped<CoordinatorRequest>` |

### WsResponse（服务器 -> 客户端）

成功：
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "result": { "DataflowList": [] }
}
```

错误：
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "error": "no running dataflow with id ..."
}
```

| 字段 | 类型 | 描述 |
|------|------|------|
| `id` | UUID | 匹配发起请求的 `id` |
| `result` | object? | 成功时存在（序列化的 `ControlRequestReply`） |
| `error` | string? | 失败时存在 |

### WsEvent（双向）

```json
{
  "event": "log",
  "payload": { "message": "sensor started", "level": "info" }
}
```

在 `LogSubscribe`/`BuildLogSubscribe` 确认后用于日志流。

### 分发

每个处理器使用自己的策略解析传入帧以保持 u128 精度（参见 [u128 序列化](#u128-序列化变通方案)）：

- **CLI（`ws_client.rs`）**：使用扁平的 `IncomingFrame` 结构体，对 `result`/`payload` 字段使用 `serde_json::value::RawValue`，完全避免 `serde_json::Value`。通过 `event`（日志推送）或 `id`（响应）的存在来区分。
- **Coordinator 控制处理器（`ws_control.rs`）**：解析为 `WsRequest`（始终是来自 CLI 的请求）。
- **Coordinator daemon 处理器（`ws_daemon.rs`）**：检查 `"method"` 键来区分请求与响应。对请求使用 `DaemonWsRequestRaw` 辅助结构。
- **Daemon（`coordinator.rs`）**：使用 `CoordinatorCommandRaw` / `RegisterReplyRaw` 辅助结构直接从原始 JSON 文本解析。

`ws_protocol.rs` 中定义了一个 `WsMessage` 无标签枚举用于通用分发，但生产处理器不使用：

```rust
#[serde(untagged)]
pub enum WsMessage {
    Request(WsRequest),
    Response(WsResponse),
    Event(WsEvent),
}
```

---

## CLI 控制平面（`/api/control`）

CLI 连接到 `/api/control` 发送 `ControlRequest` 命令并接收 `ControlRequestReply` 响应。

### 连接生命周期

1. **连接** -- HTTP 升级为 WebSocket
2. **请求-应答** -- CLI 发送 `WsRequest`，coordinator 处理 `ControlRequest`，发送 `WsResponse`
3. **日志订阅**（可选） -- CLI 发送 `LogSubscribe`/`BuildLogSubscribe`，coordinator 用 `WsResponse` 确认，然后推送 `WsEvent{event:"log"}` 帧
4. **关闭** -- CLI 发送 `Close` 帧或断开连接

### 支持的 ControlRequest 变体

| 变体 | 描述 |
|------|------|
| `List` | 列出所有运行中的数据流 |
| `Build` | 触发数据流构建 |
| `WaitForBuild` | 阻塞直到构建完成 |
| `Start` | 启动数据流 |
| `WaitForSpawn` | 阻塞直到节点生成 |
| `Stop` / `StopByName` | 停止运行中的数据流 |
| `Reload` | 热重载节点/算子 |
| `Check` | 检查数据流状态 |
| `Destroy` | 关闭所有 daemon |
| `Logs` | 获取历史日志 |
| `Info` | 获取数据流详情 |
| `DaemonConnected` | 检查是否有 daemon 连接 |
| `ConnectedMachines` | 列出已连接 daemon |
| `LogSubscribe` | 订阅实时数据流日志 |
| `BuildLogSubscribe` | 订阅实时构建日志 |
| `CliAndDefaultDaemonOnSameMachine` | 检查共置 |
| `GetNodeInfo` | 获取节点元数据 |
| `TopicSubscribe` | 通过二进制 WS 帧订阅实时主题数据（[详情](websocket-topic-data-channel.md)） |
| `TopicUnsubscribe` | 取消主题订阅 |

### 日志订阅流程

```
CLI                         Coordinator
 │                              │
 │─── WsRequest{LogSubscribe} ─>│
 │                              │  (检查数据流是否存在)
 │<── WsResponse{subscribed} ───│
 │                              │
 │<── WsEvent{event:"log"} ────│  (重复)
 │<── WsEvent{event:"log"} ────│
 │                              │
 │─── Close ───────────────────>│  (log_subscribers 被丢弃)
```

如果数据流未找到，coordinator 返回带错误的 `WsResponse`，不发送事件。

### WsSession（CLI 客户端）

`WsSession` 是一个同步包装器，将阻塞 CLI 代码桥接到异步 WebSocket 连接。它创建一个内部 `tokio::runtime::Runtime`（当前线程）并生成异步 `session_loop` 任务。

```
CLI 线程（同步）                       session_loop（异步）
     │                                        │
     │── SessionCommand::Request ────────────>│── WsRequest ──> 服务器
     │                                        │<── WsResponse ──
     │<── oneshot 回复 ─────────────────────│
     │                                        │
     │── SessionCommand::SubscribeLogs ──────>│── WsRequest ──> 服务器
     │                                        │<── WsResponse (确认)
     │<── oneshot 确认 ───────────────────────│
     │<── std_mpsc 日志事件 ─────────────────│<── WsEvent ──
```

session loop 维护：
- `pending_requests: HashMap<Uuid, oneshot::Sender>` -- 用于请求-应答关联
- `pending_subscribes: HashMap<Uuid, (ack_tx, log_tx)>` -- 用于订阅确认路由
- `log_subscribers: Vec<std_mpsc::Sender>` -- 用于广播日志事件
- `pending_topic_subscribes: HashMap<Uuid, (ack_tx, data_tx)>` -- 用于主题订阅确认路由
- `topic_subscribers: HashMap<Uuid, std_mpsc::Sender>` -- 用于按订阅 UUID 分发二进制帧

二进制 WS 帧（主题数据）与文本帧分开分发。详见 [WebSocket 主题数据通道](websocket-topic-data-channel.md)。

断开连接时，所有待处理请求通过其 oneshot 通道收到错误。

---

## Daemon 平面（`/api/daemon`）

Daemon 连接到 `/api/daemon` 进行注册、事件报告和接收 coordinator 命令。

### 注册流程

```
Daemon                       Coordinator
  │                              │
  │── WsRequest{Register} ─────>│
  │                              │  (验证、分配 daemon_id)
  │                              │  (跟踪连接 + cmd 通道)
  │                              │
  │── WsRequest{Event{...}} ───>│  (后续事件)
```

1. Daemon 发送包含 `DaemonRegisterRequest`（版本 + machine ID）的 `Register` 请求
2. Coordinator 验证版本兼容性和 machine ID 长度
3. Coordinator 分配 `DaemonId` 并存储 `DaemonConnection`（包括用于向 daemon 发送命令的 `cmd_tx` 通道）
4. 连接通过 `tracked_daemon_id` 跟踪，用于断开时清理

### 事件转换

Daemon 事件被转换为 coordinator 内部 `Event` 变体：

| DaemonEvent | Coordinator Event |
|-------------|-------------------|
| `AllNodesReady` | `Event::Dataflow { ReadyOnDaemon }` |
| `AllNodesFinished` | `Event::Dataflow { DataflowFinishedOnDaemon }` |
| `Heartbeat` | `Event::DaemonHeartbeat` |
| `Log(message)` | `Event::Log(message)` |
| `Exit` | `Event::DaemonExit` |
| `NodeMetrics` | `Event::NodeMetrics` |
| `BuildResult` | `Event::DataflowBuildResult` |
| `SpawnResult` | `Event::DataflowSpawnResult` |

### 双向通信

coordinator 可以通过存储在 `DaemonConnection` 中的 `cmd_tx` 通道向 daemon 发送命令。daemon 处理器维护 `pending_replies: HashMap<Uuid, oneshot::Sender>` 来关联 daemon 对 coordinator 发起请求的响应。

daemon 处理器上的消息路由：
- 帧有 `"method"` 键 -> daemon 请求（注册或事件）
- 帧没有 `"method"` 键 -> daemon 对 coordinator 命令的响应

### u128 序列化变通方案

`uhlc::ID` 包含一个 `NonZeroU128`，超出了 `serde_json::Value::Number` 范围（仅 i64/u64/f64）。使用 `serde_json::to_value()` 会报 "number out of range" 错误，`serde_json::from_slice::<Value>()` 通过存储为 f64 静默丢失精度。

所有生产代码对包含 `uhlc::Timestamp` 的数据绕过 `serde_json::Value`：

| 组件 | 序列化 | 反序列化 |
|------|--------|---------|
| Daemon（`coordinator.rs`） | `to_string` + `format!` | 辅助结构（`RegisterReplyRaw`、`CoordinatorCommandRaw`）+ `from_str` |
| Coordinator 控制（`ws_control.rs`） | 回复用 `to_string` + `format!` | 不适用（CLI 请求不包含 u128） |
| Coordinator daemon（`ws_daemon.rs`） | 不适用 | `DaemonWsRequestRaw` + `from_str` |
| Coordinator 状态（`state.rs`） | `str::from_utf8` + `format!`（原始字节嵌入） | 不适用 |
| CLI（`ws_client.rs`） | 不适用（请求不包含 u128） | `IncomingFrame` 配合 `serde_json::value::RawValue` |

集成测试类似地通过 `format!()` + `serde_json::to_string()`（不是 `to_value()`）手动构建 WsRequest JSON 字符串以匹配真实传输格式。

---

## 安全性

### 重注册防护

每个 daemon WebSocket 连接仅允许一次 `Register` 请求。如果连接尝试第二次注册，coordinator 记录警告并关闭连接：

```
daemon attempted re-register on same connection, rejecting
```

### Daemon ID 验证

注册后，每个 `Event` 消息必须包含与注册期间分配的 daemon_id 匹配的 `daemon_id`。ID 不匹配导致连接终止：

```
daemon sent event with mismatched id: expected `X`, got `Y` -- closing connection
```

### Machine ID 长度验证

`DaemonRegisterRequest` 中的 `machine_id` 字段限制为 256 字节。超大值导致连接终止。

### 连接和消息限制

| 限制 | 值 | 执行者 |
|------|-----|--------|
| 最大消息大小 | 1 MiB | `WebSocketUpgrade::max_message_size` |
| 最大并发连接 | 256 | Tower `ConcurrencyLimitLayer` |

---

## 连接生命周期和保活

### 建立

`/api/control` 和 `/api/daemon` 都使用标准 HTTP/1.1 WebSocket 升级。Axum `WebSocketUpgrade` 提取器处理握手。

### Ping/pong

两个处理器都用包含相同有效载荷的 `Pong` 帧回应 `Ping` 帧：

```rust
Ok(Message::Ping(data)) => {
    let _ = ws_tx.send(Message::Pong(data)).await;
    continue;
}
```

### 优雅关闭

收到 `Close` 帧时：
- **控制处理器**：中断处理器循环，丢弃日志订阅者通道
- **Daemon 处理器**：中断循环，然后发出 `Event::DaemonExit { daemon_id }` 用于立即清理

### 断开时的清理

**控制连接**：
- `log_tx` 通道被丢弃，停止向该客户端转发日志
- 无 coordinator 状态需要清理（控制连接是无状态的）

**Daemon 连接**：
- 如果跟踪了 `daemon_id`，发出 `DaemonExit` 事件
- `cmd_tx` 和 `pending_replies` 被丢弃
- Coordinator 从其连接映射中移除 daemon

**WsSession（CLI 客户端）**：
- `pending_requests` 中的所有条目收到 `Err("WS connection closed")`
- `pending_subscribes` 中的所有条目收到 `Err("WS connection closed")`

---

## 消息流示例

### CLI 列出数据流

```
CLI                          WsSession                    Coordinator
 │                              │                              │
 │── request(&List) ───────────>│                              │
 │                              │── WsRequest ────────────────>│
 │                              │   id: "abc-123"              │
 │                              │   method: "control"          │
 │                              │   params: "List"             │
 │                              │                              │
 │                              │                    ControlEvent::IncomingRequest
 │                              │                    通过 oneshot 回复
 │                              │                              │
 │                              │<── WsResponse ──────────────│
 │                              │   id: "abc-123"              │
 │                              │   result: {DataflowList:[]}  │
 │                              │                              │
 │<── ControlRequestReply ─────│                              │
```

### Daemon 注册

```
Daemon                                    Coordinator
  │                                           │
  │── WsRequest ─────────────────────────────>│
  │   method: "daemon_event"                  │
  │   params: {inner: Register{...},          │
  │            timestamp: ...}                │
  │                                           │  验证版本
  │                                           │  验证 machine_id
  │                                           │  分配 daemon_id
  │                                           │  存储 DaemonConnection
  │                                           │
  │── WsRequest{Event{Heartbeat}} ──────────>│
  │                                           │  Event::DaemonHeartbeat
  │                                           │
  │                        (WS 关闭时) ──────>│  Event::DaemonExit
```

### 日志订阅生命周期

```
CLI                    WsSession              Coordinator
 │                        │                        │
 │── subscribe_logs() ───>│                        │
 │                        │── WsRequest ──────────>│
 │                        │   params: LogSubscribe │
 │                        │                        │  查找数据流
 │                        │<── WsResponse ────────│  {subscribed: true}
 │<── ack (Ok) ──────────│                        │
 │                        │                        │
 │                        │<── WsEvent{log} ──────│  (节点产生日志)
 │<── log_rx.recv() ─────│                        │
 │                        │<── WsEvent{log} ──────│
 │<── log_rx.recv() ─────│                        │
 │                        │                        │
 │   (丢弃 session) ─────>│── Close ─────────────>│  (log_subscribers 被丢弃)
```

---

## 测试覆盖

### 测试层级

| 层级 | 位置 | 测试数 | 覆盖内容 |
|------|------|--------|---------|
| 单元（协议） | `libraries/message/src/ws_protocol.rs` | 10 | 往返序列化、无标签分发、错误情况 |
| 单元（客户端） | `binaries/cli/src/ws_client.rs` | 6 | 响应路由、订阅确认、主题订阅确认、孤立处理、断开 |
| 集成（控制） | `binaries/coordinator/tests/ws_control_tests.rs` | 11 | 健康检查、List、无效 JSON/参数、Destroy、DaemonConnected、ping/pong、并发请求、连接关闭、日志订阅 |
| 集成（daemon） | `binaries/coordinator/tests/ws_daemon_tests.rs` | 4 | 注册、注册后状态、断开清理、ping/pong |
| 端到端（WsSession） | `tests/ws-cli-e2e.rs` | 4 | WsSession + coordinator：list、status、stop、多请求 |
| **总计** | | **35** | |

### 关键测试模式

**带超时轮询**：集成测试以 2 秒截止和 20ms 睡眠间隔轮询 coordinator 状态（如 `DaemonConnected`），避免不稳定的时序假设。

**无嵌套运行时**：端到端测试在后台 `std::thread` 上使用自己的 tokio 运行时运行 coordinator，而 `WsSession`（创建自己的当前线程运行时）在测试主线程上运行。这避免了"cannot start a runtime from within a runtime"的 panic。

**测试中的 u128 变通方案**：Daemon 测试辅助函数通过 `format!()` + `serde_json::to_string()`（不是 `serde_json::to_value()`）手动构建 WsRequest JSON 字符串，以在传输中保留 `uhlc::ID` u128 值。

**测试 coordinator 设置**：集成和端到端测试都使用 `adora_coordinator::start_testing()`，它绑定到端口 0（OS 分配）并接受空的外部事件流。

---

## 配置参考

### 常量

| 常量 | 值 | 文件 | 用途 |
|------|-----|------|------|
| `MAX_CONTROL_MESSAGE_BYTES` | 1 MiB (1,048,576) | `ws_server.rs` | 最大 WebSocket 帧大小 |
| `MAX_WS_CONNECTIONS` | 256 | `ws_server.rs` | Tower 并发限制 |

### 服务器设置

```rust
// 生产：由 coordinator 的主启动调用
let (port, shutdown, future) = ws_server::serve(bind_addr, event_tx, clock).await?;
tokio::spawn(future);
// ...
shutdown.shutdown(); // 优雅停止
```

### 测试设置

```rust
// 绑定到端口 0，返回 (port, future)
let (port, future) = adora_coordinator::start_testing(
    "127.0.0.1:0".parse().unwrap(),
    futures::stream::empty(),
).await?;
```

### 关闭

`ShutdownTrigger` 包装一个 `oneshot::Sender<()>`。调用 `.shutdown()` 发送信号，Axum 服务器通过 `with_graceful_shutdown` 接收。正在处理的请求完成；新连接被拒绝。
