> 🌐 **[English](../../advanced/ws-topic.md)**

# WebSocket 主题数据通道

主题数据通道扩展了 WebSocket 控制平面，将实时数据流消息从 coordinator 代理到 CLI 客户端。CLI 命令如 `topic echo`、`topic hz` 和 `topic info` 通过现有的 WebSocket 连接以二进制帧接收消息数据，而无需直接 Zenoh 网络访问。

## 动机

| 场景 | 之前（Zenoh 直连） | 之后（WS 代理） |
|------|-------------------|-----------------|
| CLI 与 daemon 同机 | 可用 | 可用 |
| CLI 远程，Zenoh 可达 | 可用 | 可用 |
| CLI 远程，无 Zenoh 访问 | 不可用 | 可用 |
| 基于浏览器的 Web UI | 不可能 | 可能 |
| 嵌入式目标，无本地磁盘 | 无法本地录制 | `--proxy` 流式传输到 CLI |

关键洞察：CLI 和未来的 Web UI 通过 WebSocket 连接到 coordinator。通过让 coordinator 代替它们订阅 Zenoh 并将消息作为二进制帧转发，主题检查在 WebSocket 连接可达的任何地方都能工作。

---

## 架构

```
CLI  ──── WS（二进制帧）────>  Coordinator  ──── Zenoh 订阅 ────>  Daemon
                              (Zenoh 代理)                       (调试发布)
```

coordinator 充当 Zenoh 代理：

1. CLI 通过现有文本帧 WS 协议发送 `TopicSubscribe` 请求
2. Coordinator 验证数据流并打开 Zenoh 订阅者
3. Coordinator 将每个 Zenoh 样本作为二进制 WS 帧转发回 CLI
4. CLI 通过订阅 UUID 将二进制帧分派到相应的消费者

### 关键源文件

| 文件 | 角色 |
|------|------|
| `libraries/message/src/cli_to_coordinator.rs` | `TopicSubscribe`、`TopicUnsubscribe` 请求变体 |
| `libraries/message/src/coordinator_to_cli.rs` | `TopicSubscribed` 回复变体 |
| `binaries/coordinator/src/ws_control.rs` | Zenoh 代理：订阅、转发二进制帧 |
| `binaries/coordinator/src/control.rs` | `ControlEvent::TopicSubscribe` 用于验证 |
| `binaries/cli/src/ws_client.rs` | `WsSession::subscribe_topics()`、二进制帧分发 |
| `binaries/cli/src/command/topic/echo.rs` | 通过 WS 的主题回显 |
| `binaries/cli/src/command/topic/hz.rs` | 通过 WS 的主题频率测量 |
| `binaries/cli/src/command/topic/info.rs` | 通过 WS 的主题元数据/统计 |
| `binaries/cli/src/command/record.rs` | 基于 WS 录制的 `--proxy` 标志 |

---

## 传输协议

### 订阅握手（JSON 文本帧）

订阅使用现有的 UUID 关联请求-应答协议：

**请求**（CLI -> Coordinator）：
```json
{
  "id": "abc-123",
  "method": "control",
  "params": {
    "TopicSubscribe": {
      "dataflow_id": "550e8400-...",
      "topics": [["camera_node", "image"], ["lidar_node", "points"]]
    }
  }
}
```

**响应**（Coordinator -> CLI）：
```json
{
  "id": "abc-123",
  "result": {
    "TopicSubscribed": {
      "subscription_id": "7f1b3a00-..."
    }
  }
}
```

**取消订阅**（CLI -> Coordinator）：
```json
{
  "id": "def-456",
  "method": "control",
  "params": {
    "TopicUnsubscribe": {
      "subscription_id": "7f1b3a00-..."
    }
  }
}
```

### 二进制数据帧

握手后，coordinator 推送二进制 WS 帧。每个帧有一个固定大小的头部：

```
 0                   16                              N
 ├───────────────────┼──────────────────────────────┤
 │  subscription_id  │  Timestamped<InterDaemonEvent>│
 │  (16 字节 UUID)   │  (bincode 序列化)             │
 └───────────────────┴──────────────────────────────┘
```

| 字段 | 大小 | 描述 |
|------|------|------|
| `subscription_id` | 16 字节 | 匹配 `TopicSubscribed` 确认的 UUID，用于多路复用 |
| payload | 可变 | 来自 Zenoh 的原始 `Timestamped<InterDaemonEvent>` bincode 字节 |

16 字节 UUID 前缀允许在单个 WS 连接上多路复用多个订阅，无需额外帧开销。

---

## 数据流

```
CLI                         WsSession                     Coordinator
 │                              │                              │
 │── subscribe_topics() ───────>│                              │
 │                              │── WsRequest{TopicSubscribe} >│
 │                              │                              │ 验证数据流
 │                              │                              │ 打开 Zenoh 会话（延迟）
 │                              │                              │ 生成订阅者任务
 │                              │<── WsResponse{TopicSubscribed}│
 │<── (sub_id, data_rx) ───────│                              │
 │                              │                              │
 │                              │       ┌── Zenoh 样本 ────────│ Daemon 发布
 │                              │<──────│ 二进制帧             │
 │<── data_rx.recv() ──────────│       │ (sub_id + payload)   │
 │                              │       │                      │
 │                              │<──────│ 二进制帧             │
 │<── data_rx.recv() ──────────│       │                      │
 │                              │       └                      │
 │                              │                              │
 │   (丢弃 session) ───────────>│── Close ────────────────────>│ 中止订阅者任务
```

### Coordinator 内部

1. **验证**：`ControlEvent::TopicSubscribe` 发送到 coordinator 事件循环，检查数据流存在且启用了 `publish_all_messages_to_zenoh: true`
2. **延迟 Zenoh**：coordinator 的 Zenoh 会话在第一个 `TopicSubscribe` 请求时打开，对同一 WS 连接上的后续订阅复用
3. **每 topic 任务**：每个 `(node_id, data_id)` 对生成一个 tokio 任务，订阅对应的 Zenoh topic 并将样本转发到二进制帧通道
4. **背压**：二进制帧通道容量为 64。使用 `try_send` -- 如果通道满了（慢消费者），样本被静默丢弃而非阻塞 Zenoh 订阅者
5. **清理**：WS 连接关闭时，所有订阅者任务被中止

### WsSession（CLI 端）

`WsSession::subscribe_topics()` 方法：

1. 序列化 `TopicSubscribe` 请求
2. 通过内部命令通道发送 `SessionCommand::SubscribeTopics`
3. 异步 `session_loop` 将其包装为 `WsRequest` 并发送
4. 收到 `TopicSubscribed` 确认后，在 `topic_subscribers` 中注册 `data_tx` 发送器，以 `subscription_id` 为键
5. 二进制帧通过提取前 16 字节作为 UUID 并将剩余部分发送到匹配的 `data_tx` 来分发

`session_loop` 中维护的状态：
- `pending_topic_subscribes: HashMap<Uuid, (ack_tx, data_tx)>` -- 等待确认
- `topic_subscribers: HashMap<Uuid, Sender>` -- 接收二进制数据的活跃订阅

---

## 前提条件

数据流描述符必须启用调试消息发布：

```yaml
_unstable_debug:
  publish_all_messages_to_zenoh: true
```

不启用时，coordinator 拒绝 `TopicSubscribe` 并返回：
```
dataflow {id} not found or publish_all_messages_to_zenoh not enabled
```

---

## CLI 命令

### `adora topic echo`

实时将主题数据流到终端。

```bash
# 回显单个主题
adora topic echo -d my-dataflow camera_node/image

# 回显多个主题
adora topic echo -d my-dataflow robot1/pose robot2/vel

# JSON 输出用于管道
adora topic echo -d my-dataflow robot1/pose --format json
```

内部：调用 `session.subscribe_topics()`，从 `data_rx` 通道接收 `Timestamped<InterDaemonEvent>`，反序列化 Arrow 数据，并渲染为表格或 JSON。

### `adora topic hz`

显示每 topic 发布频率统计的交互式 TUI。

```bash
# 所有主题
adora topic hz -d my-dataflow --window 10

# 特定主题
adora topic hz -d my-dataflow robot1/pose robot2/vel --window 5
```

使用 ratatui 实现 TUI。后台 `std::thread` 从 `data_rx` 接收事件，并通过 `BTreeMap<(node_id, data_id), index>` 查找分发到每 topic 的 `HzStats` 跟踪器。

### `adora topic info`

一次性主题元数据和统计。

```bash
adora topic info -d my-dataflow camera_node/image --duration 5
```

收集 `--duration` 秒的消息，然后显示类型信息、发布者、订阅者（来自描述符）、消息数量和带宽。

### `adora record --proxy`

通过 WebSocket 流式传输数据流数据用于本地录制。

```bash
# 先启动数据流
adora start dataflow.yml --detach

# 通过代理录制（数据通过 coordinator 流到 CLI）
adora record dataflow.yml --proxy -o capture.adorec

# 录制特定主题
adora record dataflow.yml --proxy --topics sensor/image,lidar/points
```

使用场景：目标机器（运行 daemon）没有本地磁盘或存储有限。`--proxy` 标志通过 coordinator WebSocket 将数据路由到 CLI 机器，在本地写入 `.adorec` 文件。

不使用 `--proxy`（默认）时，录制节点被注入数据流并直接在 daemon 机器上录制。

---

## Zenoh Topic 格式

coordinator 使用 `adora_core::topics::zenoh_output_publish_topic()` 的格式订阅 Zenoh topic：

```
adora/{dataflow_id}/{node_id}/{data_id}
```

每个 topic 携带 `Timestamped<InterDaemonEvent>` 作为有效载荷，用 bincode 序列化。coordinator 原样转发这些字节（前置订阅 UUID）-- 无需重新序列化。

---

## 背压和性能

| 参数 | 值 | 原因 |
|------|-----|------|
| 二进制帧通道容量 | 64 | 延迟和内存之间的平衡 |
| 丢弃策略 | 满时丢弃 | 优先新鲜度而非完整性 |
| 二进制格式 | 原始 bincode（无 base64） | 避免大负载 33% 的开销 |

对于高吞吐量 topic（相机图像、点云），如果 WS 连接慢，二进制帧通道可能满。丢弃的样本是静默的 -- CLI 在 `topic hz` 中会显示降低的频率但不会停滞。

---

## 错误处理

| 错误 | 来源 | 响应 |
|------|------|------|
| 数据流未找到 | Coordinator 验证 | 带错误消息的 WsResponse |
| `publish_all_messages_to_zenoh` 未启用 | Coordinator 验证 | 带错误消息的 WsResponse |
| Zenoh 会话打开失败 | Coordinator | 带错误消息的 WsResponse |
| Zenoh 订阅者失败 | 每 topic 任务 | 警告日志，任务退出 |
| 二进制帧太短（<16 字节） | CLI session_loop | 警告日志，帧被丢弃 |
| 未知订阅 UUID | CLI session_loop | 帧被静默丢弃 |
| WS 连接关闭 | 任一端 | 所有任务中止，待处理确认收到错误 |

---

## 测试覆盖

| 层级 | 位置 | 覆盖内容 |
|------|------|---------|
| 单元（客户端） | `binaries/cli/src/ws_client.rs` | `handle_response_topic_subscribe_ack` -- 验证确认路由和订阅者注册 |
| 单元（所有现有） | `binaries/cli/src/ws_client.rs` | 更新以通过 `handle_response` 传递主题订阅状态 |

`TopicSubscribe` / 二进制帧路径主要通过运行 coordinator 和 Zenoh 会话的集成测试验证。参见[测试指南](testing-guide.md)获取冒烟测试说明。
