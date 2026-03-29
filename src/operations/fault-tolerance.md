> 🌐 **[English](../../operations/fault-tolerance.md)**

# 容错机制

Adora 为机器人和 AI 数据流提供内置容错功能。节点可以在故障时自动重启，检测上游连接超时，在输入不可用时优雅降级，coordinator 可以将状态持久化到磁盘以在崩溃和重启后恢复。

## 功能概览

| 功能 | 范围 | 配置 |
|------|------|------|
| 重启策略 | 每节点 | `restart_policy`、`max_restarts`、`restart_delay` 等 |
| 健康监控 | 每节点 | `health_check_timeout`、`health_check_interval`（数据流级） |
| 输入超时 | 每输入 | `input_timeout` |
| 断路器 | 自动 | 由 `input_timeout` 触发，自动恢复 |
| NodeRestarted 事件 | 下游节点 | 上游重启时自动发送 |
| InputTracker API | Rust 节点 | `adora_node_api::InputTracker` |
| 可观测性 | daemon 范围 | 周期性记录的原子计数器 |
| 分布式健康 | 多 daemon | Coordinator 心跳监控 |
| Coordinator 状态持久化 | Coordinator | `--store redb`（需要 `redb-backend` 特性） |

---

## 重启策略

控制节点退出或崩溃时的行为。

### 配置

```yaml
nodes:
  - id: my-node
    path: ./target/debug/my-node
    restart_policy: on-failure  # never | on-failure | always
    max_restarts: 5             # 0 = 无限（默认：0）
    restart_delay: 1.0          # 初始延迟（秒）
    max_restart_delay: 30.0     # 指数退避上限
    restart_window: 300.0       # 此秒数后重置计数器
```

### 策略类型

**`never`**（默认）-- 节点不会重启。故障正常传播。

**`on-failure`** -- 仅在节点以非零退出码退出时重启。正常退出（代码 0）不会重启。

**`always`** -- 在任何退出时重启，除非：
- 数据流被用户停止（`adora stop` 或 Ctrl-C）
- 所有输入关闭且节点以非零代码退出

### 重启内部工作原理

当节点进程退出时，daemon 按以下顺序评估重启决策：

1. **策略检查**：重启策略是否允许？
   - `Never` -> 不重启
   - `OnFailure` -> 仅在退出码 != 0 时重启
   - `Always` -> 重启
2. **禁用检查**：是否设置了 `disable_restart`？（所有输入关闭或通过 `stop_all` 手动停止时设置）
3. **窗口检查**：如果设置了 `restart_window` 且自第一次重启以来窗口已过期，将计数器重置为 0
4. **限制检查**：如果 `max_restarts > 0` 且窗口计数器超过限制，永久放弃
5. **退避**：如果设置了 `restart_delay`，睡眠计算出的延迟时间（醒来后重新检查 `disable_restart`）
6. **重新生成**：使用相同配置重新生成节点进程

daemon 在 `spawn/prepared.rs` 生命周期循环中跟踪每个节点实例的重启状态。每个节点在自己的 tokio 任务中运行，因此重启不会阻塞其他节点。

### 退避

设置 `restart_delay` 后，daemon 在重启前等待。延迟每次尝试翻倍（指数退避），受 `max_restart_delay` 限制。

退避指数内部限制为 16，以防止溢出（`2^16 = 65536x 倍数`）。

示例，`restart_delay: 1.0` 和 `max_restart_delay: 10.0`：
```
尝试 1：等待 1s    (1.0 * 2^0)
尝试 2：等待 2s    (1.0 * 2^1)
尝试 3：等待 4s    (1.0 * 2^2)
尝试 4：等待 8s    (1.0 * 2^3)
尝试 5：等待 10s   (受 max_restart_delay 限制)
尝试 6：等待 10s   (受限)
```

在退避睡眠期间，daemon 持续监控 `disable_restart` 标志。如果在节点等待重启时所有输入关闭，重启将被取消并记录："restart cancelled: inputs closed during backoff wait"。

### 重启窗口

设置 `restart_window` 后，重启计数器在窗口过期后重置（从当前窗口的第一次重启开始计算）。这实现了"每 M 秒 N 次重启"的语义。

示例：`max_restarts: 5`、`restart_window: 300.0` 表示"每 5 分钟最多 5 次重启"。如果窗口过期而未达到限制，计数器重置，节点获得另外 5 次机会。

### 关闭期间的重启禁用

当 daemon 停止数据流（通过 `stop_all`）时，它在发送 Stop 事件**之前**对每个节点调用 `disable_restart()`。这防止重启机制与关闭过程冲突。`disable_restart` 标志是 daemon 事件循环和节点生成生命周期任务之间共享的 `Arc<AtomicBool>`。

### NodeRestarted 事件

当节点重启时，daemon 向所有消费其输出的下游节点发送 `NodeRestarted` 事件。这允许下游节点：

- 重置内部状态或缓存
- 记录上游恢复
- 重新初始化连接或会话

事件携带重启节点的 `NodeId`。下游节点通过事件流自动接收：

```rust
match event {
    Event::NodeRestarted { id } => {
        println!("upstream node {id} restarted, resetting state");
        // 清除旧节点实例的任何缓存状态
    }
    _ => {}
}
```

daemon 通过 `dataflow.mappings` 查找下游节点，该映射将每个节点的输出映射到所有订阅的 (receiver_node, input_id) 对。每个唯一接收者每次重启收到一个 `NodeRestarted` 事件。

---

## 健康监控

被动监控检测停止与 daemon 通信的挂起节点。

```yaml
health_check_interval: 2.0  # 秒（默认：5.0，数据流级）
nodes:
  - id: my-node
    path: ./target/debug/my-node
    health_check_timeout: 30.0  # 秒（每节点）
    restart_policy: on-failure
```

### 可配置的健康检查间隔

`health_check_interval` 是**数据流级**设置，控制 daemon 检查节点健康的频率。默认为 5.0 秒。较低值更快检测挂起节点但增加开销。在数据流 YAML 的顶层设置，而非每节点。

### 内部工作原理

daemon 以配置的 `health_check_interval` 运行健康检查扫描（通过发出 `Event::NodeHealthCheckInterval` 的 tokio 间隔流）。

每个 `RunningNode` 有一个 `last_activity: Arc<AtomicU64>` 字段，存储最后一次通信的时间戳（自纪元以来的毫秒）。每次节点向 daemon 发送请求（事件订阅、输出发送等）时，由节点通信处理器（`node_communication/mod.rs`）原子更新。

健康检查函数（`check_node_health`）遍历所有运行节点：

1. 跳过未设置 `health_check_timeout` 的节点
2. 跳过 `last_activity == 0` 的节点（尚未连接）
3. 计算 `elapsed_ms = now - last_activity`
4. 如果 `elapsed_ms > timeout_ms`，记录警告并**终止**节点进程

终止后，正常的退出处理运行，评估重启策略。这意味着 `health_check_timeout` 结合 `restart_policy: on-failure` 可以自动恢复挂起节点。

### 什么算作"活动"

从节点到 daemon 的任何消息都算：
- 事件订阅请求
- 输出数据发送（通过共享内存或 TCP）
- 定时器 tick 确认

从其他节点接收的正常输入数据**不会**重置计时器 -- 节点必须主动与 daemon 通信。

---

## 输入超时与断路器

每输入超时检测上游节点何时停止产生数据。

### 配置

```yaml
nodes:
  - id: downstream-node
    path: ./target/debug/downstream
    inputs:
      sensor_data:
        source: camera-node/frames
        input_timeout: 5.0  # 秒
```

`input_timeout` 按每个输入设置，而非每个节点。不同输入可以有不同的超时。

### 内部工作原理

daemon 为每个带超时的输入维护一个 `InputDeadline`：

```
struct InputDeadline {
    timeout: Duration,        // 配置的超时
    last_received: Instant,   // 最后一次接收数据的时间
}
```

这些存储在 `RunningDataflow.input_deadlines` 中，以 `(NodeId, DataId)` 为键。

**超时检测**在同一 5 秒健康检查间隔中运行。`check_input_timeouts` 函数：

1. 扫描所有 `input_deadlines` 条目
2. 如果 `last_received.elapsed() > timeout`，输入为"断开"
3. `(node_id, input_id)` 对从 `input_deadlines` 移到 `broken_inputs`
4. daemon 调用 `break_input()`，向下游节点发送 `InputClosed { id }`
5. 如果节点的所有输入现在都关闭（且没有断开/可恢复的），发送 `AllInputsClosed` 并禁用节点的重启

**截止时间重置**：每次数据到达输入时，其 `last_received` 重置为 `Instant::now()`。

### 断路器：自动恢复

断路器在 `RunningDataflow.broken_inputs` 中跟踪断开的输入。当新数据到达断开的输入时：

1. 数据正常传递给节点
2. `broken_inputs` 条目被移除
3. 输入重新添加到 `open_inputs`
4. 创建新的 `InputDeadline`（重启超时）
5. 向节点发送 `InputRecovered { id }` 事件
6. `circuit_breaker_recoveries` 计数器递增

这意味着恢复是完全自动的。如果上游节点（通过重启策略）重启并开始再次产生数据，下游节点无缝恢复接收。

### 节点端处理

在 Rust 节点中，在事件循环中处理这些事件：

```rust
use adora_node_api::{AdoraNode, Event};

let (mut node, mut events) = AdoraNode::init_from_env()?;
while let Some(event) = events.recv() {
    match event {
        Event::Input { id, data, .. } => {
            // 正常处理
        }
        Event::InputClosed { id } => {
            // 上游在此输入上停止产生数据。
            // 你可以：使用缓存数据、跳过处理、告警操作员等。
        }
        Event::InputRecovered { id } => {
            // 上游在此输入上恢复在线。
            // 恢复正常处理。
        }
        Event::Stop(_) => break,
        _ => {}
    }
}
```

---

## InputTracker API（Rust）

`InputTracker` 辅助工具跟踪输入健康状况并缓存每个输入最后接收的值，使优雅降级变得容易。

```rust
use adora_node_api::{AdoraNode, Event, InputTracker, InputState};

let (mut node, mut events) = AdoraNode::init_from_env()?;
let mut tracker = InputTracker::new();

while let Some(event) = events.recv() {
    tracker.process_event(&event);

    match event {
        Event::Input { id, data, .. } => {
            // 新鲜数据可用
        }
        Event::InputClosed { id } => {
            // 输入超时 -- 回退到缓存数据
            if let Some(stale_data) = tracker.last_value(&id) {
                // 使用过期数据作为回退
            }
        }
        Event::Stop(_) => break,
        _ => {}
    }

    // 检查整体健康
    if tracker.any_closed() {
        let closed: Vec<_> = tracker.closed_inputs();
        // 记录或调整行为
    }
}
```

### 内部设计

`InputTracker` 维护两个 `HashMap`：

- `states: HashMap<DataId, InputState>` -- 每个输入的当前状态（Healthy 或 Closed）
- `cache: HashMap<DataId, ArrowData>` -- 每个输入最后接收的值

在 `Event::Input` 时，两个映射都更新（state = Healthy，cache = data clone）。在 `Event::InputClosed` 时，仅状态改变（缓存保留）。在 `Event::InputRecovered` 时，状态恢复为 Healthy。缓存永远不会清除，所以 `last_value()` 即使在输入关闭后也总是返回最近的数据。

注意：`ArrowData` 包装 `Arc<dyn arrow::array::Array>`，所以缓存克隆是引用计数的（开销小）。

### API 参考

| 方法 | 返回 | 描述 |
|------|------|------|
| `new()` | `InputTracker` | 创建空 tracker |
| `process_event(&Event)` | `bool` | 更新状态。如果事件相关则返回 true |
| `state(&DataId)` | `Option<InputState>` | 当前状态（Healthy 或 Closed） |
| `is_closed(&DataId)` | `bool` | 检查输入是否关闭 |
| `last_value(&DataId)` | `Option<&ArrowData>` | 最后接收的值（关闭时也可用） |
| `closed_inputs()` | `Vec<&DataId>` | 所有当前关闭的输入 |
| `any_closed()` | `bool` | 如果任何被跟踪的输入关闭则为 true |

---

## 可观测性

daemon 使用原子计数器（`FaultToleranceStats`）跟踪容错事件，并在健康检查间隔期间每 5 秒记录一次摘要。

### 计数器

| 计数器 | 类型 | 递增时机 |
|--------|------|---------|
| `restarts` | `AtomicU64` | 节点重启发起时（在生成生命周期中） |
| `health_check_kills` | `AtomicU64` | 节点被健康检查终止时（无响应） |
| `input_timeouts` | `AtomicU64` | 输入超时触发时（断路器跳闸） |
| `circuit_breaker_recoveries` | `AtomicU64` | 数据到达断开输入时（自动恢复） |

所有计数器使用 `Ordering::Relaxed`，因为它们是信息性的，不需要严格的排序保证。

### 日志输出

当任何计数器非零时，daemon 发出结构化日志行：

```
INFO fault tolerance stats restarts=3 health_kills=0 input_timeouts=1 cb_recoveries=1
```

这些计数器在 daemon 进程生命周期内是累积的。它们不会在数据流之间重置。

---

## 分布式健康

在多 daemon 部署中，coordinator 监控 daemon 心跳。

### 协议

- **心跳间隔**：3 秒（coordinator 向每个 daemon 发送心跳）
- **断开阈值**：30 秒无响应
- **检测**：每次心跳扫描时，coordinator 移除在阈值内未响应的 daemon
- **通知**：coordinator 向所有剩余 daemon 广播 `PeerDaemonDisconnected { daemon_id }`

### DaemonInfo

`ConnectedMachines` CLI 查询返回 `Vec<DaemonInfo>`：

```rust
pub struct DaemonInfo {
    pub daemon_id: DaemonId,
    pub last_heartbeat_ago_ms: u64,  // 自上次心跳以来的毫秒数
}
```

这允许监控工具检测存活但响应缓慢的 daemon。

### Daemon 端处理

当 daemon 收到 `PeerDaemonDisconnected` 时，它记录结构化警告：

```
WARN peer daemon disconnected daemon_id=machine-B
```

目前这是信息性的。未来可能包含从断开 daemon 自动迁移节点的功能。

---

## Coordinator 状态持久化

默认情况下 coordinator 将所有状态保存在内存中。如果 coordinator 进程崩溃或重启，所有关于运行中数据流的知识丢失 -- daemon 继续运行但成为孤儿，用户必须手动重新运行数据流。

**redb 存储后端**通过使用 [redb](https://docs.rs/redb) 将 coordinator 状态持久化到磁盘上的单个文件来解决这个问题。redb 是一个纯 Rust 嵌入式键值存储，具有设计上崩溃安全的写时复制 B 树。

### 设计：无状态 Coordinator 配有状态后端

coordinator 本身在 K8s 意义上保持无状态 -- 它可以随时停止和重启。所有持久状态存在于 `CoordinatorStore` trait 后面的存储后端中：

```
Coordinator（无状态进程）
    |
    v
CoordinatorStore trait
    |
    +-- InMemoryStore（默认，无持久化）
    +-- RedbStore    （持久化到 ~/.adora/coordinator.redb）
```

这种分离意味着：
- coordinator 事件循环在正常运行期间从不读取文件系统（仅在启动恢复时）
- 所有状态变更在定义良好的持久化点写入存储
- 存储可以在不更改 coordinator 逻辑的情况下替换

### 启用持久化

```bash
# 使用默认路径（~/.adora/coordinator.redb）
adora coordinator --store redb

# 使用自定义路径
adora coordinator --store redb:/path/to/coordinator.redb

# 默认：仅内存（无持久化）
adora coordinator --store memory
```

`redb` 后端需要 `redb-backend` Cargo 特性，该特性在默认 CLI 构建中已启用。

### 持久化内容

存储跟踪三种记录类型：

| 记录 | 键 | 持久化字段 |
|------|-----|-----------|
| `DataflowRecord` | UUID（16 字节） | uuid、name、descriptor（JSON）、status、daemon ID、generation 计数器、创建/更新时间戳 |
| `BuildRecord` | UUID（16 字节） | build ID、status、errors、创建/更新时间戳 |
| `DaemonInfo` | DaemonId（bincode） | daemon ID、machine ID |

记录使用 [bincode](https://docs.rs/bincode/2) 序列化，提供紧凑、快速的编码。

### 数据流状态生命周期

coordinator 在每次状态转换时持久化数据流状态：

```
Start 命令     -->  Pending
所有 daemon 就绪 -->  Running
Stop 命令      -->  Stopping
所有节点完成   -->  Succeeded  或  Failed { error }
生成失败       -->  Failed { error: "spawn failed: ..." }
```

每次持久化调用递增记录的 `generation` 计数器，为冲突检测提供单调版本。

### 持久化时间点

coordinator 在事件循环的以下时刻写入存储：

1. **数据流启动**（`ControlRequest::Start`）-- 创建状态为 `Pending` 的记录
2. **数据流生成**（所有 daemon 的 `DataflowSpawnResult` 成功）-- 更新为 `Running`
3. **生成失败**（`DataflowSpawnResult` 错误）-- 更新为 `Failed` 并附带实际错误消息
4. **停止请求**（`ControlRequest::Stop` 或 `StopByName`）-- 更新为 `Stopping`
5. **所有节点完成**（`DataflowFinishedOnDaemon`）-- 更新为 `Succeeded` 或 `Failed` 并附带每节点错误详情
6. **优雅关闭**（Ctrl-C 或 `Destroy` 命令）-- 在发送停止消息前将所有运行中数据流标记为 `Stopping`

如果存储写入失败，coordinator 记录警告并继续以内存状态运行。这防止存储故障阻塞数据流生命周期。

### 启动恢复

当 coordinator 以包含前一次运行数据的 redb 存储启动时，执行恢复：

1. 通过 `store.list_dataflows()` 读取所有持久化数据流记录
2. 对于任何非终态状态的记录（`Pending`、`Running`、`Stopping`）：
   - 标记为 `Failed { error: "coordinator restarted" }`
   - 递增 generation 计数器
   - 将更新后的记录写回存储
3. 终态记录（`Succeeded`、`Failed`）保持不变

这确保崩溃 coordinator 中的过期数据流不会与实际运行的混淆。运行这些数据流的 daemon 会独立检测到 coordinator 断开。

### 错误详情保留

当数据流失败时，`Failed` 状态包含实际的每节点错误消息而非通用字符串：

```
Failed { error: "node-1: exited with code 137; node-2: failed to spawn node: binary not found" }
```

错误从所有 daemon 的 `DataflowDaemonResult.node_results` 收集，格式化为 `node_id: error_message`，用 `; ` 连接。

### Schema 版本控制

redb 数据库包含一个带 `schema_version` 键的 `meta` 表。打开时：

- 如果不存在版本（新数据库），写入当前版本
- 如果存储的版本与二进制版本匹配，数据库正常打开
- 如果不匹配，数据库被拒绝并报告错误

这防止在 Adora 版本之间存储记录序列化格式更改时的静默数据损坏。当前 schema 版本为 `1`。

### 文件安全

在 Unix 系统上：
- 数据库文件创建后设置为 `0600`（仅所有者读写）
- 默认目录（`~/.adora/`）设置为 `0700`（仅所有者）
- 通过 `redb:/path` 提供的自定义路径会验证以拒绝 `..` 组件

### 内部架构

```rust
// 存储 trait (libraries/coordinator-store/src/lib.rs)
pub trait CoordinatorStore: Send + Sync {
    fn put_dataflow(&self, record: &DataflowRecord) -> Result<()>;
    fn get_dataflow(&self, uuid: &Uuid) -> Result<Option<DataflowRecord>>;
    fn list_dataflows(&self) -> Result<Vec<DataflowRecord>>;
    fn delete_dataflow(&self, uuid: &Uuid) -> Result<()>;
    // ... daemon 和 build 方法
}
```

`RedbStore` 实现使用三个 redb 表（`daemons`、`dataflows`、`builds`），基于 UUID 的二进制键和 bincode 序列化的值。所有操作是同步的（redb 是同步库）；coordinator 直接从异步事件循环调用它们，因为它们是快速的进程内操作。

bincode 反序列化限制为 64 MiB，以防止损坏数据在长度前缀中编码巨大分配大小。

---

## 完整 YAML 参考

```yaml
# 数据流级设置
health_check_interval: 2.0    # 健康检查扫描间隔（默认：5.0 秒）

nodes:
  - id: sensor-node
    path: ./target/debug/sensor
    inputs:
      tick: adora/timer/millis/100
    outputs:
      - frames

  - id: processor
    path: ./target/debug/processor

    # 重启策略
    restart_policy: on-failure    # never | on-failure | always
    max_restarts: 5               # 0 = 无限
    restart_delay: 1.0            # 初始退避延迟（秒）
    max_restart_delay: 30.0       # 最大退避上限（秒）
    restart_window: 300.0         # N 秒后重置计数器

    # 健康监控
    health_check_timeout: 30.0    # N 秒无活动则终止

    inputs:
      frames:
        source: sensor-node/frames
        input_timeout: 5.0        # 断路器超时（秒）
        queue_size: 10            # 输入缓冲区大小（默认：10）
    outputs:
      - result
```

---

## 使用场景

### 1. 间歇性硬件故障的相机管道

相机驱动节点偶尔因 USB 断开而崩溃。处理管道应在这些中断中存活并在相机重连时恢复。

```yaml
nodes:
  - id: camera-driver
    path: ./target/debug/camera-driver
    restart_policy: on-failure
    max_restarts: 0               # 无限 -- 硬件故障是预期的
    restart_delay: 2.0            # 等待 USB 重新枚举
    max_restart_delay: 30.0
    inputs:
      tick: adora/timer/millis/33  # 约 30 FPS
    outputs:
      - frames

  - id: object-detector
    path: ./target/debug/detector
    inputs:
      frames:
        source: camera-driver/frames
        input_timeout: 5.0        # 容忍 5 秒相机中断
    outputs:
      - detections

  - id: planner
    path: ./target/debug/planner
    inputs:
      detections:
        source: object-detector/detections
        input_timeout: 10.0       # 更长容忍 -- 可以使用过期数据规划
      lidar:
        source: lidar-driver/points
        input_timeout: 3.0
```

**相机崩溃时发生什么：**

1. `camera-driver` 以非零代码退出
2. daemon 评估 `on-failure` 策略 -> 2 秒退避后重启
3. 中断期间，5 秒后 `object-detector` 收到 `InputClosed { id: "frames" }`
4. 10 秒后 `planner` 收到 `InputClosed { id: "detections" }`
5. 相机重启，开始产生帧
6. `object-detector` 收到新帧数据 + `InputRecovered { id: "frames" }`（断路器恢复）
7. `planner` 收到检测结果 + `InputRecovered { id: "detections" }`

**planner 中的节点端处理：**

```rust
use adora_node_api::{AdoraNode, Event, InputTracker};

let (mut node, mut events) = AdoraNode::init_from_env()?;
let mut tracker = InputTracker::new();

while let Some(event) = events.recv() {
    tracker.process_event(&event);

    match event {
        Event::Input { id, data, .. } => match id.as_ref() {
            "detections" => plan_with_detections(&data),
            "lidar" => update_lidar_map(&data),
            _ => {}
        },
        Event::InputClosed { id } => match id.as_ref() {
            "detections" => {
                // 相机管道停机 -- 仅用激光雷达规划
                plan_lidar_only();
            }
            "lidar" => {
                // 激光雷达停机 -- 使用最后已知的检测数据
                if let Some(stale) = tracker.last_value(&"detections".into()) {
                    plan_with_stale_detections(stale);
                }
            }
            _ => {}
        },
        Event::Stop(_) => break,
        _ => {}
    }
}
```

### 2. 带 OOM 崩溃的 ML 推理节点

ML 推理节点偶尔在大输入上内存不足。它应该快速重启但在反复失败后放弃（表明系统性问题）。

```yaml
nodes:
  - id: ml-inference
    path: ./target/debug/ml-inference
    restart_policy: on-failure
    max_restarts: 3
    restart_delay: 0.5
    restart_window: 60.0          # 每分钟 3 次重启
    health_check_timeout: 60.0    # ML 推理可能很慢
    inputs:
      images:
        source: preprocessor/images
    outputs:
      - predictions
```

### 3. 带优雅降级的多传感器融合

机器人融合多个传感器的数据。单个传感器可能失败，但系统应以降低的能力继续运行。

```yaml
nodes:
  - id: sensor-fusion
    path: ./target/debug/sensor-fusion
    inputs:
      camera:
        source: camera-node/frames
        input_timeout: 3.0
      lidar:
        source: lidar-node/points
        input_timeout: 3.0
      imu:
        source: imu-node/readings
        input_timeout: 1.0        # IMU 是关键的，短超时
      gps:
        source: gps-node/fix
        input_timeout: 10.0       # GPS 可能间歇性
    outputs:
      - fused-state
```

**使用 InputTracker 的节点端：**

```rust
use adora_node_api::{AdoraNode, Event, InputTracker};

let (mut node, mut events) = AdoraNode::init_from_env()?;
let mut tracker = InputTracker::new();

while let Some(event) = events.recv() {
    tracker.process_event(&event);

    match event {
        Event::Input { id, data, .. } => {
            // 处理来自任何传感器的新鲜数据
            update_sensor(&id, &data);
            compute_and_send_fusion(&mut node, &tracker);
        }
        Event::InputClosed { id } => {
            // 传感器离线 -- 调整融合权重
            eprintln!("sensor {id} offline, degrading");
            compute_and_send_fusion(&mut node, &tracker);
        }
        Event::InputRecovered { id } => {
            // 传感器恢复在线
            eprintln!("sensor {id} recovered");
        }
        Event::Stop(_) => break,
        _ => {}
    }
}

fn compute_and_send_fusion(node: &mut AdoraNode, tracker: &InputTracker) {
    // 使用可用的新鲜数据，对降级传感器使用过期缓存
    let camera = tracker.last_value(&"camera".into());
    let lidar = tracker.last_value(&"lidar".into());
    let imu = tracker.last_value(&"imu".into());

    if tracker.is_closed(&"imu".into()) {
        // IMU 是关键的 -- 切换到紧急模式
        emergency_stop(node);
        return;
    }

    // 融合可用传感器，对活跃的赋予更高权重
    let closed = tracker.closed_inputs();
    let active_count = 4 - closed.len();
    // ... 使用 active_count 进行置信度加权的融合逻辑
}
```

### 4. 长期运行的数据处理管道

批处理管道持续运行。处理节点偶尔因第三方库 bug 而挂起。健康监控检测并恢复这些挂起。

```yaml
nodes:
  - id: data-ingest
    path: ./target/debug/ingest
    restart_policy: always        # 始终重启 -- 这是长期运行服务
    max_restarts: 0               # 无限
    restart_delay: 1.0
    inputs:
      tick: adora/timer/millis/1000
    outputs:
      - records

  - id: processor
    path: ./target/debug/processor
    restart_policy: on-failure
    max_restarts: 10
    restart_delay: 0.5
    restart_window: 600.0         # 每 10 分钟 10 次重启
    health_check_timeout: 30.0    # 挂起 30 秒则终止
    inputs:
      records: data-ingest/records
    outputs:
      - results

  - id: writer
    path: ./target/debug/writer
    restart_policy: on-failure
    max_restarts: 5
    restart_delay: 2.0            # 给数据库时间恢复
    max_restart_delay: 60.0
    inputs:
      results:
        source: processor/results
        input_timeout: 60.0       # 处理器可能很慢
```

### 5. 带 redb 持久化的 Coordinator 崩溃恢复

```bash
# 使用持久化存储启动 coordinator
adora coordinator --store redb

# 在另一个终端，启动数据流
adora start examples/rust-dataflow/dataflow.yml --name my-pipeline --detach

# coordinator 崩溃或被终止（如 OOM、硬件故障）
# ... 时间过去 ...

# 使用同一存储重启 coordinator
adora coordinator --store redb
```

**重启时发生什么：**

1. Coordinator 打开 `~/.adora/coordinator.redb` 并读取持久化数据流记录
2. 找到状态为 `Running` 的 `my-pipeline`
3. 标记为 `Failed { error: "coordinator restarted" }`，递增 generation
4. 记录：`INFO recovering stale dataflow <uuid> ("my-pipeline") -> marking as Failed`
5. `adora list` 现在显示 `my-pipeline` 及其最终状态和时间戳
6. Daemon 独立检测到 coordinator 断开并停止其节点
7. 用户可以启动新的数据流 -- coordinator 完全运行

关键好处：coordinator 在跨重启中保留完整的数据流生命周期事件历史。不使用 `--store redb` 时，所有状态在退出时丢失，操作员不会有崩溃前运行内容的记录。

### 6. 定期批处理作业使用 Always-Restart

处理批次后退出的节点。它应该重启以处理下一个批次。

```yaml
nodes:
  - id: batch-processor
    path: ./target/debug/batch-proc
    restart_policy: always        # 即使正常退出也重启
    max_restarts: 0               # 无限
    restart_delay: 10.0           # 批次之间等待 10 秒
    max_restart_delay: 10.0       # 无指数增长
    inputs:
      trigger: adora/timer/millis/1  # 立即第一次触发
    outputs:
      - batch-result
```

节点处理一个批次，以代码 0 退出，等待 10 秒，然后重启处理下一个。`always` 策略确保即使成功也重启。设置 `restart_delay == max_restart_delay` 提供恒定延迟。

---

## 最佳实践

**从 `on-failure` 开始**。仅对预期退出并重启的节点（如定期批处理作业）使用 `always`。

**设置 `max_restarts`**。无限重启可能掩盖 bug。从 3-5 开始，根据需要增加。仅对预期且不可避免崩溃的节点（硬件驱动、外部 API 客户端）使用 `max_restarts: 0`。

**使用 `restart_window`**。防止永久重启循环。60-300 秒的窗口是典型的。没有窗口时，在启动时崩溃的节点会立即耗尽重启预算。

**调优 `restart_delay`**。从 0.5-1.0 秒开始。太短导致抖动；太长延迟恢复。将延迟与节点的典型启动时间和故障根因匹配：
- USB/硬件重连：2-5 秒
- 网络服务重连：1-3 秒
- OOM/瞬态 bug：0.5-1.0 秒

**`health_check_timeout` 设置要宽裕**。应至少是节点最长预期处理时间的 2-3 倍。ML 推理节点可能需要 60 秒以上。设置太短会在正常处理期间终止健康节点。

**按每个输入设置 `input_timeout`**。不是所有输入都需要相同的超时。对高频输入（IMU、相机）使用较短超时，对慢/突发源（GPS、批处理结果）使用较长超时。好的起点是预期发布间隔的 3-5 倍。

**对关键路径使用 `InputTracker`**。当节点必须在降级输入下继续运行时，使用 `InputTracker` 回退到缓存数据。这对传感器融合、规划和控制节点至关重要。

**生产部署使用 `--store redb`**。redb 后端确保 coordinator 在崩溃和重启后保留数据流历史。内存默认适合开发但在退出时丢失所有状态。redb 文件很小（与数据流记录数成正比）且开销可忽略。

**组合功能实现纵深防御**：
- `restart_policy` + `restart_delay` -> 从节点崩溃恢复
- `health_check_timeout` -> 从挂起节点恢复
- `input_timeout` -> 检测过期上游数据
- `InputTracker` -> 节点代码中的优雅降级
- `--store redb` -> 在 coordinator 崩溃后存活
