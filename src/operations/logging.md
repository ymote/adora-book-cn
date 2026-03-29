> 🌐 **[English](../../operations/logging.md)**

# 日志

Adora 为实时机器人和 AI 数据流提供结构化日志系统。日志以结构化 JSONL 文件的形式按节点捕获，转发到 coordinator 用于实时流式传输，并可选择作为数据消息通过数据流图路由。

## 应该使用哪种日志方式？

如果不确定哪种方式适合你的场景，从这里开始。

| 我想要... | 方式 | 配置 |
|-----------|------|------|
| **从 Python 记录日志** | 使用 Python 的 `logging` 模块（自动桥接） | 无 -- 只需 `import logging` |
| **从 Rust 记录日志** | 使用 `node.log_info()` / `node.log_error()` 等 | 无 -- 开箱即用 |
| **从 C/C++ 记录日志** | 使用 `adora_log()` / `log_message()` | 无 -- 开箱即用 |
| **过滤嘈杂的节点** | 在 YAML 中设置 `min_log_level` | 每节点 YAML 字段 |
| **在一个地方查看所有日志** | 订阅 `adora/logs` 虚拟输入 | `inputs: logs: adora/logs` |
| **将某个节点的日志作为数据处理** | 在该节点上使用 `send_logs_as` | 每节点 YAML + 连接输出 |
| **轮转日志文件** | 在 YAML 中设置 `max_log_size` | 每节点 YAML 字段 |
| **构建自定义日志接收器** | 使用 `adora-log-utils` crate | Rust 依赖 |
| **过滤 CLI 显示** | 使用 `--log-level` / `--log-filter` 标志 | CLI 标志或环境变量 |

### 各语言快速入门

**Python** -- 最简单的方式是使用 Python 内置的 `logging` 模块：

```python
import logging
from adora import Node

node = Node()  # 自动将 Python logging 桥接到 adora

logging.info("Sensor started")       # 捕获为结构化 "info" 日志
logging.warning("High temp: 42C")    # 捕获为结构化 "warn" 日志
print("raw debug output")            # 捕获为 "stdout" 级别
```

当 `Node()` 创建时，它安装一个处理器，将所有 Python `logging` 调用通过 Rust 的 `tracing` 系统路由。daemon 将这些解析为带有级别、消息、文件和行号的结构化日志条目。无需额外配置。

你也可以使用显式 API 来获取结构化字段：

```python
node.log_info("Reading acquired")
node.log("info", "Reading acquired", fields={"sensor_id": "temp-01"})
```

**Rust** -- 使用节点 API 便捷方法：

```rust
let (node, mut events) = AdoraNode::init_from_env()?;

// 便捷方法（大多数情况下推荐）
node.log_info("Sensor started");
node.log_warn("High temperature");

// 带结构化字段
let mut fields = BTreeMap::new();
fields.insert("sensor_id".into(), "temp-01".into());
node.log_with_fields("info", "Reading acquired", None, Some(&fields));
```

另外，Rust 节点也可以使用 `tracing` crate。当 adora 的 tracing 订阅者通过 `init_tracing()` 初始化后，`tracing::info!()` 等会将结构化 JSON 输出到 stdout，daemon 会自动解析：

```rust
// 同样有效 -- daemon 自动解析为结构化日志
tracing::info!("Sensor started");
tracing::warn!(sensor_id = "temp-01", "High temperature");
```

当你需要对日志格式的显式控制时使用 `node.log_*()`。当你需要生态系统集成（span、instrumentation、OpenTelemetry）时使用 `tracing::*!()`。两者在 daemon 中产生相同的结构化日志条目。

**C** -- 使用 `adora_log()` 函数：

```c
adora_log(ctx, "info", 4, "Sensor started", 14);
```

**C++** -- 使用 `log_message()` 函数：

```cpp
log_message(node.send_output, "info", "Sensor started");
```

---

## 功能概览

| 功能 | 范围 | 配置 |
|------|------|------|
| 日志级别过滤 | CLI 显示 | `--log-level`、`ADORA_LOG_LEVEL` |
| 输出格式 | CLI 显示 | `--log-format`、`ADORA_LOG_FORMAT` |
| 每节点级别覆盖 | CLI 显示 | `--log-filter`、`ADORA_LOG_FILTER` |
| 源级过滤 | 每节点 YAML | `min_log_level` |
| Stdout 数据路由 | 每节点 YAML | `send_stdout_as` |
| 结构化日志路由 | 每节点 YAML | `send_logs_as` |
| 日志文件轮转 | 每节点 YAML | `max_log_size` |
| 轮转文件限制 | 每节点 YAML | `max_rotated_files` |
| 节点日志 API | Rust/Python/C/C++ 节点 | `node.log()`、`adora_log()` 等 |
| 日志工具库 | Rust crate | `adora-log-utils` |
| **日志聚合** | **数据流输入** | **`adora/logs` 虚拟输入** |
| 时间范围过滤 | `adora logs` | `--since`、`--until` |
| 实时日志流 | `adora logs` | `--follow` |
| 文本搜索 | `adora logs` | `--grep` |
| 本地日志读取 | `adora logs` | `--local`、`--all-nodes` |

---

## 日志文件格式

每个节点产生一个 JSONL 文件（每行一个 JSON 对象），位于：

```
<working_dir>/out/<dataflow_uuid>/log_<node_id>.jsonl
```

每行具有以下结构：

```json
{
  "timestamp": "2024-01-15T10:30:00.123Z",
  "level": "info",
  "node_id": "sensor",
  "message": "Starting sensor...",
  "target": "sensor::module",
  "fields": { "key": "value" }
}
```

| 字段 | 类型 | 描述 |
|------|------|------|
| `timestamp` | string | 带毫秒精度的 RFC3339 时间戳 |
| `level` | string | `"error"`、`"warn"`、`"info"`、`"debug"`、`"trace"` 或 `"stdout"` |
| `node_id` | string | 节点 ID |
| `message` | string | 日志消息文本 |
| `target` | string? | Rust 模块目标（如 `"sensor::module"`），不存在时为 null |
| `fields` | object? | 来自日志框架的结构化键值字段。**信任模型：** 字段来源于节点 stdout，未经净化直接传递。在混合信任环境中，日志消费者应在操作前验证字段内容 |

### 节点输出如何变成日志条目

daemon 捕获节点进程 stdout/stderr 的每一行，并尝试将其解析为结构化日志消息（带 `level`、`message`、`timestamp` 和可选 `fields` 的 JSON）。如果解析成功，结构化字段被保留。如果解析失败，原始行成为 `"stdout"` 级别的条目。

这意味着使用 Rust 的 `tracing` 或 `log` crate 并带 JSON 输出的节点会自动获得完整的结构化日志。简单使用 `println!` 的节点产生 `"stdout"` 级别的条目。

---

## 查看日志：`adora run`

使用 `adora run` 运行数据流时，所有节点的日志实时显示在终端上。

### 标志

```
adora run dataflow.yml [OPTIONS]
```

| 标志 | 默认值 | 环境变量 | 描述 |
|------|--------|----------|------|
| `--log-level LEVEL` | `stdout` | `ADORA_LOG_LEVEL` | 最小显示级别 |
| `--log-format FORMAT` | `pretty` | `ADORA_LOG_FORMAT` | 输出格式：`pretty`、`json`、`compact` |
| `--log-filter FILTER` | 无 | `ADORA_LOG_FILTER` | 每节点级别覆盖 |

### 日志级别

从最详细到最不详细：

| 级别 | 描述 |
|------|------|
| `stdout` | 所有内容，包括节点的原始 stdout（默认） |
| `trace` | 细粒度诊断消息 |
| `debug` | 开发者级诊断消息 |
| `info` | 一般信息消息 |
| `warn` | 警告条件 |
| `error` | 仅错误条件 |

设置 `--log-level info` 会隐藏 `stdout`、`trace` 和 `debug` 消息。`stdout` 级别是一个特殊的全通过级别。

### 级别过滤逻辑

级别过滤使用 `LogLevelOrStdout::passes()`：

```
消息级别     过滤级别      是否显示？
─────────   ──────────   ──────────
stdout      stdout       是
stdout      info         否       （stdout 仅通过 stdout 过滤器）
info        stdout       是       （任何日志级别通过 stdout 过滤器）
debug       info         否       （debug 比 info 更详细）
error       info         是       （error 比 info 更不详细）
```

### 每节点覆盖

`--log-filter` 标志允许你为不同节点设置不同级别：

```bash
adora run dataflow.yml --log-level info --log-filter "sensor=debug,planner=warn"
```

这对所有节点显示 `info` 及以上，但 `sensor`（显示 `debug` 及以上）和 `planner`（显示 `warn` 及以上）除外。

格式：`"node1=level,node2=level"`（逗号分隔的 `name=level` 对）。

### 输出格式

**Pretty**（默认）-- 彩色、人类可读：
```
10:30:00 INFO   sensor: Starting sensor...

10:30:01 INFO   [adora]: spawning node processor

10:30:01 stdout sensor: raw output line
```

- 本地时区的时间戳（`HH:MM:SS`）
- 级别颜色：ERROR（红色）、WARN（黄色）、INFO（绿色）、DEBUG（蓝色）、TRACE（暗淡）、stdout（斜体暗蓝）
- 节点名称加粗，带基于名称的唯一颜色
- 系统消息以 `[adora]` 为前缀
- 生命周期消息（`spawning`、`node finished`、`stopping`）用空行进行视觉分隔

**Json** -- 完整的 `LogMessage` 结构体 JSON，每行一条：
```json
{"build_id":null,"dataflow_id":"abc-123","node_id":"sensor","level":"INFO","message":"Starting...","timestamp":"2024-01-15T10:30:00Z",...}
```

适合管道到 `jq` 或接入日志聚合系统。

**Compact** -- 最小化，无颜色：
```
10:30:00 INFO sensor: Starting sensor...
```

适合 CI/CD 环境和日志文件。

---

## 查看日志：`adora logs`

读取历史日志或从运行中的数据流流式传输实时日志。

### 基本用法

```bash
# 读取特定节点的日志（通过 coordinator）
adora logs <dataflow_uuid> <node_name>

# 直接读取本地日志文件
adora logs --local <node_name>
adora logs --local --all-nodes

# 流式传输实时日志
adora logs <dataflow_uuid> <node_name> --follow
adora logs --local <node_name> --follow
```

### 标志

| 标志 | 简写 | 默认值 | 描述 |
|------|------|--------|------|
| `--local` | | false | 从本地 `out/` 目录读取，而非 coordinator |
| `--all-nodes` | | false | 合并所有节点的日志，按时间戳排序 |
| `--tail N` | `-n` | 全部 | 仅显示最后 N 行 |
| `--follow` | `-f` | false | 流式传输新日志条目 |
| `--since DURATION` | | 无 | 仅显示此时长前之后的日志 |
| `--until DURATION` | | 无 | 仅显示此时长前之前的日志 |
| `--level LEVEL` | | `stdout` | 最小日志级别（环境变量：`ADORA_LOG_LEVEL`） |
| `--grep PATTERN` | | 无 | 不区分大小写的文本搜索 |
| `--coordinator-addr IP` | | `127.0.0.1` | Coordinator 地址 |
| `--coordinator-port PORT` | | 默认 | Coordinator 控制端口 |

### 时间过滤

`--since` 和 `--until` 接受相对于当前时间的时长字符串：

```bash
# 最近 5 分钟的日志
adora logs --local sensor --since 5m

# 1 小时前到 30 分钟前的日志
adora logs --local sensor --since 1h --until 30m

# 过去一小时最后 10 条错误
adora logs --local sensor --since 1h --level error --tail 10
```

支持的时长格式：`30`（秒）、`30s`、`5m`、`1h`、`2d`。

### 文本搜索

`--grep` 对以下内容执行不区分大小写的子串匹配：
- 日志消息文本
- 节点 ID
- 模块目标

```bash
# 查找所有超时相关消息
adora logs --local --all-nodes --grep "timeout"

# 从特定模块查找错误
adora logs --local sensor --grep "camera::driver" --level error
```

### 过滤管道

所有过滤器按此顺序应用：

```
读取/解析 -> 时间过滤 -> Grep -> Tail -> 显示
```

在 coordinator 模式下使用 `--since`、`--until` 或 `--grep` 时，CLI 从服务器获取所有日志（服务器端忽略 `--tail`），然后在客户端应用所有过滤器。这确保组合过滤器时结果正确。

### 本地模式与 Coordinator 模式

**本地模式**（`--local`）直接从当前工作目录的 `out/` 目录读取 JSONL 文件。无需运行 coordinator 或 daemon。如果使用 `--all-nodes` 或未给出节点名称，所有日志文件将合并并按时间戳排序。

**Coordinator 模式**（默认）通过 WebSocket 连接到运行中的 coordinator。coordinator 从 daemon 的工作目录读取日志文件并流式传回。适用于本地和分布式部署。

### Follow 模式

**本地 follow**（`--local --follow`）：每 200ms 轮询日志文件的新内容。新行被解析、按 `--grep` 过滤后打印。时间/tail 过滤仅适用于初始历史输出。

**Coordinator follow**（`--follow`）：向 coordinator 打开 WebSocket 订阅。coordinator 实时从 daemon 转发日志消息。级别过滤在服务器端应用以提高效率。`--grep` 和 `--since` 在客户端应用于流。

---

## 环境变量

所有环境变量作为后备 -- CLI 标志始终优先。

| 变量 | 使用者 | 值 | 描述 |
|------|--------|-----|------|
| `ADORA_LOG_LEVEL` | `adora run`、`adora logs` | `error`、`warn`、`info`、`debug`、`trace`、`stdout` | 默认最小日志级别 |
| `ADORA_LOG_FORMAT` | `adora run` | `pretty`、`json`、`compact` | 默认输出格式 |
| `ADORA_LOG_FILTER` | `adora run` | `"node1=level,node2=level"` | 默认每节点覆盖 |
| `ADORA_QUIET` | daemon | 任意值 | 抑制日志转发到显示（文件写入继续） |

示例：

```bash
# 为开发会话设置默认值
export ADORA_LOG_LEVEL=info
export ADORA_LOG_FORMAT=pretty
export ADORA_LOG_FILTER="sensor=debug"

# 这些是等效的：
adora run dataflow.yml
adora run dataflow.yml --log-level info --log-format pretty --log-filter "sensor=debug"

# CLI 标志覆盖环境变量：
adora run dataflow.yml --log-level debug   # 覆盖 ADORA_LOG_LEVEL=info
```

---

## YAML 配置

### `min_log_level`

在源端（daemon 端）过滤日志，在它们到达日志文件、coordinator 或 `send_logs_as` 路由之前。

```yaml
nodes:
  - id: noisy-sensor
    path: ./target/debug/sensor
    min_log_level: info    # 抑制此节点的 debug/trace/stdout
```

有效值：`error`、`warn`、`info`、`debug`、`trace`、`stdout`。

设置后，daemon 在解析后立即丢弃低于此级别的日志消息。这减少了磁盘 I/O、网络流量和日志文件大小。过滤使用与 CLI 显示过滤器相同的 `passes()` 逻辑。

### `send_stdout_as`

将原始 stdout/stderr 行作为数据流输出消息路由。

```yaml
nodes:
  - id: legacy-node
    path: ./legacy-script.py
    send_stdout_as: raw_output
    outputs:
      - raw_output
      - data

  - id: log-consumer
    inputs:
      logs: legacy-node/raw_output
```

每个 stdout/stderr 行作为 Arrow 编码的字符串发送。这对于集成通过 stdout 输出数据的遗留节点（如使用 `print()` 的 Python 脚本）很有用。

`send_stdout_as` 和正常的日志文件写入同时发生 -- stdout 路由不会抑制日志文件。

### `send_logs_as`

将解析的结构化日志条目作为数据流输出消息路由。

```yaml
nodes:
  - id: sensor
    path: ./target/debug/sensor
    send_logs_as: log_entries
    outputs:
      - data
      - log_entries

  - id: log-aggregator
    inputs:
      sensor_logs: sensor/log_entries
```

与 `send_stdout_as` 不同，这只发送成功解析为结构化日志的行（不包括原始 stdout）。每个条目序列化为完整的 JSON `LogMessage` 字符串。`min_log_level` 过滤器在路由之前应用 -- 被抑制的消息不会发送。

使用此功能在数据流内构建日志聚合、告警或监控节点。

### `adora/logs` -- 自动日志聚合

用一个输入行订阅**所有节点**的日志 -- 无需手动连线：

```yaml
nodes:
  - id: sensor
    path: sensor.py
    inputs:
      tick: adora/timer/millis/200
    outputs:
      - reading

  - id: processor
    path: processor.py
    inputs:
      reading: sensor/reading
    outputs:
      - result

  - id: log-viewer
    path: log_viewer.py
    inputs:
      logs: adora/logs              # 所有节点，所有级别
      errors: adora/logs/error      # 仅所有节点的 error+
      sensor: adora/logs/info/sensor  # 一个节点的 info+
```

`adora/logs` 虚拟输入像 `adora/timer` 一样工作 -- daemon 在内部处理订阅。每条日志消息作为 Arrow 数组中的 JSON 编码 `LogMessage` 字符串到达。为防止无限循环，节点永远不会收到自己的日志消息。

**语法：**

| 输入 | 描述 |
|------|------|
| `adora/logs` | 所有节点的所有日志 |
| `adora/logs/<level>` | 所有节点的 `<level>` 及以上级别日志 |
| `adora/logs/<level>/<node-id>` | 特定节点的 `<level>` 及以上级别日志 |

级别：`stdout`、`error`、`warn`、`info`、`debug`、`trace`。

**何时使用 `adora/logs` vs `send_logs_as`：**

| | `adora/logs` | `send_logs_as` |
|--|-------------|---------------|
| 范围 | 一次性所有节点 | 一次一个节点 |
| YAML 修改 | 仅消费者 | 每个源节点 |
| 添加节点 | 零连线更改 | 必须更新消费者 |
| 用例 | 仪表板、监控 | 每节点日志处理 |

参见 `examples/log-aggregator/` 获取完整的工作示例。

### `max_log_size`

启用基于大小的日志文件轮转。

```yaml
nodes:
  - id: sensor
    path: ./target/debug/sensor
    max_log_size: "50MB"
```

| 值 | 字节 |
|----|------|
| `"1KB"` 或 `"1K"` | 1,024 |
| `"50MB"` 或 `"50M"` | 52,428,800 |
| `"1GB"` 或 `"1G"` | 1,073,741,824 |
| `"1000"` | 1,000（纯数字 = 字节） |

当活跃日志文件超过配置大小时，daemon：

1. 刷新并关闭当前文件
2. 重命名已有的轮转文件：`.4.jsonl` -> `.5.jsonl`、`.3.jsonl` -> `.4.jsonl`，以此类推
3. 重命名当前文件：`log_sensor.jsonl` -> `log_sensor.1.jsonl`
4. 创建新的 `log_sensor.jsonl`
5. 删除超过轮转限制的文件（默认 5，可通过 `max_rotated_files` 配置）

**命名规范：**

```
log_sensor.jsonl       # 当前（活跃）
log_sensor.1.jsonl     # 上一个
log_sensor.2.jsonl     # 更早
log_sensor.3.jsonl
log_sensor.4.jsonl
log_sensor.5.jsonl     # 最旧（下次轮转时删除）
```

每节点最大磁盘使用量：`max_log_size * (1 + max_rotated_files)`（1 个活跃 + N 个轮转）。

不设置 `max_log_size` 时，日志文件无限增长。对于长时间运行的数据流，务必设置此项。

`adora logs --local` 命令自动读取节点的所有轮转文件，并按时间顺序合并（最旧的轮转文件在前，当前文件在后）。

### `max_rotated_files`

控制保留多少个轮转日志文件（默认：5，范围：1-100）。

```yaml
nodes:
  - id: sensor
    path: ./target/debug/sensor
    max_log_size: "50MB"
    max_rotated_files: 10    # 保留 10 个轮转文件而非 5 个
```

`max_rotated_files: 10` 和 `max_log_size: "50MB"` 时，每节点最大磁盘使用量为 `50MB * 11` = 550MB。较低值节省磁盘空间；较高值保留更多历史。

### 运行时节点限制

对于运行时节点（算子），每个运行时仅允许一个日志字段：

```yaml
# 正确 -- 单个算子
nodes:
  - id: runtime-node
    operator:
      python: process.py
      send_logs_as: logs
      min_log_level: info
      max_log_size: "100MB"

# 错误 -- 多个算子配置冲突
nodes:
  - id: runtime-node
    operators:
      - id: op1
        python: a.py
        send_logs_as: logs1
      - id: op2
        python: b.py
        send_logs_as: logs2    # 错误：多个 send_logs_as
```

当运行时中的单个算子设置这些字段时，输出名称以算子 ID 为前缀（如 `op1/logs`）。

---

## 节点日志 API

节点可以使用节点 API 以编程方式发出结构化日志消息。这等同于将 JSON 格式的日志行写入 stdout -- daemon 以相同方式解析它们。

### Rust

```rust
use adora_node_api::AdoraNode;
use std::collections::BTreeMap;

let (node, mut events) = AdoraNode::init_from_env()?;

// 通用日志，带级别字符串和可选目标
node.log("info", "sensor initialized", Some("sensor::init"));

// 便捷方法（无目标参数）
node.log_error("connection failed");
node.log_warn("temperature elevated");
node.log_info("reading acquired");
node.log_debug("raw bytes received");
node.log_trace("entering loop iteration");

// 结构化字段（键值上下文通过 send_logs_as 保留）
let mut fields = BTreeMap::new();
fields.insert("sensor_id".to_string(), "temp-01".to_string());
fields.insert("reading".to_string(), "42.5".to_string());
node.log_with_fields("info", "reading acquired", None, Some(&fields));
```

`level` 参数接受 `"error"`、`"warn"`（或 `"warning"`）、`"info"`、`"debug"`、`"trace"`。未知级别默认为 `"info"`。字段总量限制为 60 KB，以匹配下游 64 KB 解析限制。

### Python

Python 节点有三种记录日志的方式，都产生结构化日志条目：

```python
from adora import Node
import logging

node = Node()

# 方式 1：Python 的 logging 模块（推荐 -- 由 Node() 自动桥接）
logging.info("sensor initialized")
logging.warning("temperature elevated")
logging.debug("raw bytes: %s", data)

# 方式 2：显式 adora API，带级别字符串
node.log("info", "sensor initialized", target="sensor.init")
node.log("info", "reading acquired", fields={"sensor_id": "temp-01", "reading": "42.5"})

# 方式 3：便捷方法
node.log_error("connection failed")
node.log_warn("temperature elevated")
node.log_info("reading acquired")
node.log_debug("raw bytes received")
node.log_trace("entering loop iteration")

# 这也有效，但产生 "stdout" 级别条目（无结构）：
print("raw output")
```

**Python logging 桥接工作原理：** 当 `Node()` 创建时，它安装一个自定义 `logging.Handler`，将所有 Python `logging` 调用通过 Rust 的 `tracing` 系统路由。daemon 将这些解析为带级别、消息、文件路径和行号的结构化日志条目。这是自动发生的 -- 无需配置。

| 方法 | 结构化？ | 支持字段？ | 何时使用 |
|------|---------|-----------|----------|
| `logging.info()` | 是 | 否（使用 `extra=` 自定义格式器） | 通用日志 |
| `node.log("info", msg, fields={...})` | 是 | 是 | 需要结构化键值上下文时 |
| `node.log_info(msg)` | 是 | 否 | 快速一行，等同于 `node.log("info", msg)` |
| `print()` | 否（`stdout` 级别） | 否 | 遗留代码、快速调试 |

**常见陷阱：** 不要在创建 `Node()` 之前调用 `logging.basicConfig()`。节点构造函数设置日志桥接；先调用 `basicConfig()` 可能安装冲突的处理器。如果需要自定义格式器，在 `Node()` 创建后配置。

### C

```c
#include "node_api.h"

void *ctx = init_adora_context_from_env();
const char *level = "info";
const char *msg = "sensor initialized";
adora_log(ctx, level, strlen(level), msg, strlen(msg));
```

### C++

```cpp
// 通过 cxx 桥接
auto node = init_adora_node();
log_message(node.send_output, "info", "sensor initialized");
```

---

## 日志工具库（`adora-log-utils`）

`adora-log-utils` crate 提供解析、合并、过滤和格式化 `LogMessage` 条目的工具，用于自定义接收器节点。当构建通过 `send_logs_as` 消费日志数据的节点时使用。

### API

```rust
use adora_log_utils;

// 从 JSON 解析 LogMessage（来自 send_logs_as）
let log = adora_log_utils::parse_log(json_str)?;

// 直接从 Arrow 输入数据解析（事件处理器的便捷方法）
let log = adora_log_utils::parse_log_from_arrow(&data)?;

// 将多个日志流合并为单一时间线
let merged = adora_log_utils::merge_by_timestamp(vec![stream_a, stream_b]);

// 按最小级别过滤
let errors = adora_log_utils::filter_by_level(&logs, &min_level);

// 格式化为 JSON（单行，无尾部换行）
let json = adora_log_utils::format_json(&log);

// 格式化为紧凑单行："<timestamp> <node> <LEVEL>: <message>"
let compact = adora_log_utils::format_compact(&log);

// 格式化为美观格式："[<timestamp>][<LEVEL>][<node>] <message>"
let pretty = adora_log_utils::format_pretty(&log);
```

### 依赖

添加到接收器节点的 `Cargo.toml`：

```toml
[dependencies]
adora-log-utils = { workspace = true }
```

---

## 日志接收器示例

三个示例接收器节点演示如何消费通过 `send_logs_as` 路由的日志并转发到外部目标。

### 文件接收器（`examples/log-sink-file/`）

将来自多个节点的日志流合并到单个 JSONL 文件中。适用于统一日志收集。

```yaml
nodes:
  - id: sensor
    path: sensor.py
    send_logs_as: log_entries
    inputs:
      tick: adora/timer/millis/200
    outputs:
      - reading
      - log_entries

  - id: processor
    path: processor.py
    send_logs_as: log_entries
    inputs:
      reading: sensor/reading
    outputs:
      - result
      - log_entries

  - id: file_sink
    path: log-sink-file
    inputs:
      sensor_logs: sensor/log_entries
      processor_logs: processor/log_entries
    env:
      LOG_FILE: "./combined.jsonl"
```

文件接收器从环境读取 `LOG_FILE`（默认 `./combined.jsonl`），使用 `adora_log_utils::parse_log_from_arrow()` 解析每条传入的 Arrow 消息，格式化为 JSON 并追加到文件。

### TCP 接收器（`examples/log-sink-tcp/`）

通过 TCP 套接字将日志条目转发到远程日志收集器。适用于缺少本地文件系统且需要将日志流式传输到设备外部的嵌入式系统。

```yaml
nodes:
  - id: source
    path: source.py
    send_logs_as: log_entries
    inputs:
      tick: adora/timer/millis/500
    outputs:
      - data
      - log_entries

  - id: tcp_sink
    path: log-sink-tcp
    inputs:
      logs: source/log_entries
    env:
      SINK_ADDR: "127.0.0.1:9876"
```

TCP 接收器从环境读取 `SINK_ADDR`（默认 `127.0.0.1:9876`），在启动时连接到服务器，并将每条日志条目作为 JSON 行发送。在写入失败时自动重连。

### 告警路由器（`examples/log-sink-alert/`）

按严重性拆分传入的日志条目。所有日志转发到 `all_logs` 输出；仅错误和警告日志转发到 `alerts` 输出。这使下游节点能以不同方式处理告警（如触发通知、写入专用文件）。

```yaml
nodes:
  - id: source
    path: my_node.py
    send_stdout_as: log_entries
    inputs:
      tick: adora/timer/millis/200
    outputs:
      - log_entries

  - id: alert_router
    path: log-sink-alert
    inputs:
      logs: source/log_entries
    outputs:
      - all_logs
      - alerts
```

源节点使用 `send_stdout_as` 将其 stdout 行作为 Arrow 字符串数据路由。路由器使用 `adora_log_utils::parse_log_from_arrow()` 解析每条日志条目，检查级别，并使用 `node.send_output()` 将数据转发到相应的输出。使用节点 API 的节点也可以使用 `send_logs_as` 来路由来自 `node.log()` 的结构化日志。

### 构建自定义接收器

要构建自己的接收器节点，遵循此模式：

```rust
use adora_node_api::{AdoraNode, Event};

fn main() -> eyre::Result<()> {
    let (_node, mut events) = AdoraNode::init_from_env()?;

    while let Some(event) = events.recv() {
        match event {
            Event::Input { data, .. } => {
                let log = adora_log_utils::parse_log_from_arrow(&data)?;
                // 处理日志条目：写入文件、通过网络发送等
                let json = adora_log_utils::format_json(&log);
                println!("{json}");
            }
            Event::Stop(_) => break,
            _ => {}
        }
    }
    Ok(())
}
```

---

## Daemon 如何处理日志

了解内部管道有助于调试和调优。对于每个节点，daemon 运行一个专用异步任务，按顺序处理日志行：

```
节点进程 (stdout/stderr)
    |
    v
[1] 捕获：行在 mpsc 通道中缓冲（容量 100）
    |
    v
[2] send_stdout_as：原始行 -> Arrow 数据 -> 数据流输出
    |
    v
[3] 解析：尝试 JSON 结构化日志，回退到 Stdout 级别
    |
    v
[4] min_log_level 过滤：丢弃低于阈值的消息
    |
    v
[5] send_logs_as：LogMessage -> JSON -> Arrow 数据 -> 数据流输出
    |
    v
[6] 写入 JSONL：紧凑格式写入日志文件，跟踪已写入字节
    |
    v
[7] 轮转检查：如果 bytes_written >= max_log_size，轮转文件
    |
    v
[8] 转发：将 LogMessage 发送到显示通道（除非 ADORA_QUIET）
    |
    v
[9] 同步：fsync 日志文件到磁盘
```

关键细节：

- **步骤 2** 在解析前发生，所以 `send_stdout_as` 捕获每一行，包括非结构化输出
- **步骤 4** 在步骤 5-8 之前发生，所以 `min_log_level` 抑制所有下游处理的消息
- **步骤 5** 仅对成功解析的结构化日志触发（步骤 3 成功路径）
- **步骤 8** 发送到 flume 通道（`adora run` 直接模式）或 coordinator（分布式模式）
- **步骤 9** 每次写入后调用 `sync_all()`，确保持久性但有一定 I/O 开销

### 结构化日志解析

当节点发出 JSON 格式的日志输出（如使用 JSON 格式的 `tracing-subscriber`）时，daemon 提取：

- `level`：日志严重性
- `message`：日志文本
- `target`：模块路径
- `timestamp`：日志发出时间
- `fields`：任意键值对
- `build_id`、`dataflow_id`、`node_id`、`daemon_id`：作为后备从字段提取

daemon 还在所有消息上设置 `dataflow_id`、`node_id` 和 `daemon_id`，确保它们始终存在于日志文件中。

---

## Coordinator 日志流协议

当 daemon 在 coordinator 下运行时（分布式模式），日志转发通过 WebSocket 工作：

1. **Daemon -> Coordinator**：每个 `LogMessage` 包装在 `DaemonEvent::Log(message)` 中，通过 daemon 的 WebSocket 连接发送
2. **Coordinator 存储**：coordinator 存储/转发日志
3. **CLI 订阅**：CLI 通过其 WebSocket 连接发送 `ControlRequest::LogSubscribe { dataflow_id, level }`
4. **服务器端过滤**：coordinator 仅转发 `msg_level <= subscription_level` 的消息。这减少了过滤订阅的网络流量
5. **CLI 接收**：消息作为序列化的 `LogMessage` 结构体到达

`--level` 标志映射到 `log::LevelFilter`：
- `stdout` -> `LevelFilter::Trace`（最宽松，接收所有内容）
- `info` -> `LevelFilter::Info`（接收 Error、Warn、Info）
- 以此类推。

---

## 完整 YAML 参考

```yaml
nodes:
  - id: sensor
    path: ./target/debug/sensor
    outputs:
      - data
      - raw_output       # 用于 send_stdout_as
      - log_entries       # 用于 send_logs_as

    # 源级日志过滤（daemon 端）
    min_log_level: info          # 抑制 debug/trace/stdout

    # 将 stdout 路由到数据流
    send_stdout_as: raw_output   # 每个 stdout 行成为数据消息

    # 将结构化日志路由到数据流
    send_logs_as: log_entries    # 解析的日志条目成为数据消息

    # 日志文件轮转
    max_log_size: "50MB"         # 文件超过 50MB 时轮转
    max_rotated_files: 5         # 保留 5 个轮转文件（默认，范围 1-100）

    inputs:
      tick: adora/timer/millis/100
```

---

## 完整示例

`examples/python-logging/` 目录包含一个可运行的三节点管道，演示每个日志功能：

```
sensor（嘈杂、高量） --> processor（结构化日志） --> monitor（日志聚合器）
```

**数据流配置要点：**

```yaml
nodes:
  - id: sensor
    path: sensor.py
    min_log_level: info       # 在源端抑制调试噪音
    max_log_size: "1KB"       # 小尺寸用于演示（快速触发轮转）
    inputs:
      tick: adora/timer/millis/50
    outputs:
      - reading

  - id: processor
    path: processor.py
    send_logs_as: log_entries  # 将结构化日志作为数据路由
    inputs:
      reading: sensor/reading
    outputs:
      - result
      - log_entries

  - id: monitor
    path: monitor.py
    inputs:
      logs: processor/log_entries
      reading: sensor/reading
```

**每个节点演示的功能：**

- **sensor** -- 混合使用 `print()`（原始 stdout）、`logging.info()`、`logging.debug()` 和 `logging.warning()`。设置 `min_log_level: info` 后，daemon 在到达日志文件前丢弃 debug 消息。设置 `max_log_size: "1KB"` 后，几秒钟后日志轮转启动。
- **processor** -- 使用 `send_logs_as: log_entries` 将其结构化日志条目作为数据流数据路由。原始 `print()` 输出*不会*被路由（仅解析的结构化条目被路由）。
- **monitor** -- 订阅 `processor/log_entries` 并计数警告/错误，演示数据流内日志聚合。

**直接模式**（`adora run` -- 单进程，适合快速测试）：

```bash
# 基本运行
adora run examples/python-logging/dataflow.yml --stop-after 5s

# 仅显示警告及以上
adora run examples/python-logging/dataflow.yml --log-level warn --stop-after 5s

# 每节点覆盖
adora run examples/python-logging/dataflow.yml --log-filter "monitor=debug,sensor=warn" --stop-after 5s

# JSON 输出用于机器解析
adora run examples/python-logging/dataflow.yml --log-format json --stop-after 3s

# 环境变量控制
ADORA_LOG_LEVEL=warn adora run examples/python-logging/dataflow.yml --stop-after 5s
```

**分布式模式**（`adora up` + `adora start` -- coordinator/daemon 架构，多机部署必需）：

```bash
# 启动基础设施
adora up

# 附加启动（实时日志流）
adora start examples/python-logging/dataflow.yml --attach

# 或分离启动并单独查询日志
adora start examples/python-logging/dataflow.yml
adora logs <dataflow-id> sensor --follow                    # 流式传输一个节点
adora logs <dataflow-id> sensor --follow --level warn       # 仅警告
adora logs <dataflow-id> --all-nodes --tail 20              # 最后 20 行
adora logs <dataflow-id> processor --grep "error" --since 5m  # 定向搜索
```

在分布式模式下，日志流向为 节点 -> Daemon -> Coordinator -> CLI，通过 WebSocket。coordinator 缓冲日志消息直到订阅者连接，所以即使晚加入也不会错过日志。YAML 级设置（`min_log_level`、`send_logs_as`、`max_log_size`）工作方式完全相同，因为它们在 daemon 端应用。

| | `adora run` | `adora start` |
|---|---|---|
| 显示过滤 | `--log-level`、`--log-format`、`--log-filter` | `adora logs` 上的 `--level` |
| 每节点覆盖 | `--log-filter "sensor=debug"` | 每节点单独的 `adora logs` |
| 远程节点 | 否 | 是 |
| 实时流 | 始终附加 | `--attach` 或 `adora logs --follow` |

**运行后日志分析**（两种模式相同）：

```bash
# 读取所有本地日志
adora logs --local --all-nodes --tail 20

# 搜索传感器日志中的警告
adora logs --local sensor --grep "high temp"

# 检查轮转是否创建了多个文件
ls -la out/*/log_sensor*.jsonl
```

---

## 使用场景

### 1. 调试嘈杂的传感器管道

相机传感器节点用调试消息淹没日志，使其他节点的错误难以看到。

```yaml
nodes:
  - id: camera
    path: ./target/debug/camera
    min_log_level: warn          # 在源端抑制 info/debug/trace
    max_log_size: "10MB"         # 限制磁盘使用

  - id: detector
    path: ./target/debug/detector

  - id: planner
    path: ./target/debug/planner
```

```bash
# 开发时：看 detector 的所有内容，仅看 camera 的警告
adora run dataflow.yml --log-level debug --log-filter "camera=warn,detector=debug"

# 生产环境：仅错误
export ADORA_LOG_LEVEL=error
adora run dataflow.yml
```

**发生了什么：**
- camera 节点的 debug/info 消息在到达日志文件前被 daemon 丢弃（`min_log_level: warn`）
- CLI 根据 `--log-filter` 进一步过滤显示
- 日志文件在 10MB 时轮转，camera 节点最多保留 60MB 磁盘

### 2. 数据流内的日志聚合

构建一个数据流内的日志监控节点，监视多个节点的错误并发送告警。

```yaml
nodes:
  - id: camera
    path: ./target/debug/camera
    send_logs_as: logs
    outputs:
      - frames
      - logs

  - id: detector
    path: ./target/debug/detector
    send_logs_as: logs
    outputs:
      - detections
      - logs

  - id: log-monitor
    path: ./target/debug/log-monitor
    inputs:
      camera_logs: camera/logs
      detector_logs: detector/logs
    outputs:
      - alerts
```

**日志监控器中的节点端处理（使用 `adora-log-utils`）：**

```rust
use adora_node_api::{AdoraNode, Event};
use adora_message::common::{LogLevel, LogLevelOrStdout};

let (mut node, mut events) = AdoraNode::init_from_env()?;
while let Some(event) = events.recv() {
    match event {
        Event::Input { data, .. } => {
            let log = adora_log_utils::parse_log_from_arrow(&data)?;

            let is_error = matches!(log.level,
                LogLevelOrStdout::LogLevel(LogLevel::Error));

            if is_error || log.message.contains("timeout") {
                // 向下游发送告警
                node.send_output("alerts", /* ... */)?;
            }
        }
        Event::Stop(_) => break,
        _ => {}
    }
}
```

另见[日志接收器示例](#日志接收器示例)部分获取完整的可运行示例。

### 3. 崩溃后的事后调试

数据流崩溃后，调查最后几分钟发生了什么。

```bash
# 查找可用的数据流
ls out/

# 读取崩溃前后所有节点的最后 50 行
adora logs --local --all-nodes --tail 50

# 关注最近 5 分钟的错误
adora logs --local --all-nodes --since 5m --level error

# 搜索特定错误模式
adora logs --local --all-nodes --grep "out of memory"

# 深入特定节点
adora logs --local detector --since 2m

# 导出为 JSON 用于外部分析
adora run dataflow.yml --log-format json 2>logs.json
```

### 4. 长期运行的生产数据流

数据流运行数天或数周。不进行日志轮转，磁盘空间会被填满。

```yaml
nodes:
  - id: ingest
    path: ./target/debug/ingest
    min_log_level: info        # 生产环境无调试噪音
    max_log_size: "100MB"      # 每节点约 600MB 最大（100MB * 6）
    restart_policy: always
    inputs:
      tick: adora/timer/millis/1000
    outputs:
      - data

  - id: processor
    path: ./target/debug/processor
    min_log_level: warn        # 仅警告和错误
    max_log_size: "50MB"
    restart_policy: on-failure
    inputs:
      data: ingest/data
    outputs:
      - results

  - id: writer
    path: ./target/debug/writer
    min_log_level: error       # 最少日志
    max_log_size: "20MB"
    inputs:
      results: processor/results
```

**磁盘预算：**
- `ingest`：最多 600MB（100MB x 6 个文件）
- `processor`：最多 300MB（50MB x 6 个文件）
- `writer`：最多 120MB（20MB x 6 个文件）
- **总计**：所有日志约 1GB 最大磁盘使用量

### 5. 分布式部署的实时监控

在不同机器上运行多个 daemon，从中央工作站监控。

```bash
# 启动基础设施（coordinator + 本地 daemon）
adora up

# 在远程机器上，启动指向 coordinator 的 daemon：
#   adora daemon --coordinator-addr 192.168.1.10

# 启动数据流（分离）
adora start dataflow.yml

# 在不同终端打开定向日志流：

# 终端 1：所有传感器警告
adora logs <dataflow-id> sensor --follow --level warn

# 终端 2：带文本搜索的处理器错误
adora logs <dataflow-id> processor --follow --level error --grep "timeout"

# 终端 3：所有节点合并
adora logs <dataflow-id> --all-nodes --follow

# 终端 4：历史 + 实时（过去一小时的错误，然后流式传输）
adora logs <dataflow-id> processor --since 1h --level error --follow

# 从另一台机器监控远程 coordinator：
adora logs <dataflow-id> sensor --follow --coordinator-addr 192.168.1.10
```

**内部工作原理：**
1. CLI 连接到 coordinator（默认 `localhost:6013`，或 `--coordinator-addr`）
2. 历史日志：请求-回复，过滤在客户端应用（`--since`、`--grep`、`--tail`）
3. `--follow`：向 coordinator 打开 WebSocket 订阅
4. coordinator 在服务器端按 `--level` 过滤后转发（减少网络流量）
5. CLI 在实时流上客户端应用 `--grep` 和 `--since`
6. coordinator 缓冲日志消息直到订阅者连接，所以晚加入的订阅者能看到近期历史

### 6. CI/CD 管道中的结构化日志

在 CI 中，使用 JSON 格式获取机器可解析的输出，使用 compact 格式获取可读日志。

```bash
# CI 工具的机器可解析日志
adora run dataflow.yml --log-format json --stop-after 30s 2>test-logs.json

# CI 控制台输出的紧凑日志
adora run dataflow.yml --log-format compact --log-level info --stop-after 30s

# 运行后分析：统计每个节点的错误
adora logs --local --all-nodes --level error | wc -l
```

使用 JSON 格式时，每行是一个完整的 `LogMessage`，可以被 `jq`、日志聚合器或自定义脚本处理：

```bash
# 使用 jq 提取错误消息
cat test-logs.json | jq -r 'select(.level == "ERROR") | "\(.node_id): \(.message)"'
```

---

## 性能考虑

日志增加的 I/O 开销与日志量成正比。以下是调优方法：

**`min_log_level` 是影响最大的设置。** 它在 daemon 端任何 I/O 之前过滤：无日志文件写入、无 coordinator 转发、无 `send_logs_as` 路由。一个每秒发出 1000 条调试行的节点在 `min_log_level: info` 下对这些行产生零开销。

**`send_logs_as` 每条日志行增加一条数据流消息。** 每个解析的日志条目被序列化为 JSON、转换为 Arrow 并通过数据流发送。对于高量节点，这可能消耗大量带宽。使用 `min_log_level` 限制路由的内容。

**`adora/logs` 订阅者共享单次序列化。** daemon 将每行日志转换为 Arrow 一次，并为每个订阅者克隆结果。成本随订阅者数量线性增长，而非日志量 x 订阅者数量。对于大多数数据流（1-3 个日志订阅者），这可以忽略不计。

**日志行大小限制为 1 MB。** 来自节点 stdout/stderr 超过 1 MB 的行会被截断，以防止堆耗尽。这保护免受将大量二进制数据转储到 stdout 的有问题节点的影响。

**建议为长期运行的数据流启用日志文件轮转。** 不设置 `max_log_size` 时，日志文件无限增长。一个每秒发出 100 行、约 200 字节/行的节点在约 14 小时内填满 1 GB。

**推荐的生产设置：**

```yaml
nodes:
  - id: my-node
    path: ./my-node
    min_log_level: info        # 在源端丢弃 debug/trace
    max_log_size: "50MB"       # 50MB 时轮转
    max_rotated_files: 5       # 保留 5 个轮转文件（300MB 最大）
```

---

## 最佳实践

**在生产环境设置 `min_log_level`。** daemon 端的源级过滤防止调试噪音到达日志文件和网络。这是减少日志量最有效的方式，因为它在任何 I/O 之前过滤。

**为长期运行的数据流始终设置 `max_log_size`。** 不进行轮转，单个嘈杂节点就能填满磁盘。从 `"50MB"`（带轮转每节点总计 300MB）开始，根据存储预算调整。使用 `max_rotated_files` 调优保留多少历史（默认 5，范围 1-100）。

**使用环境变量设置团队默认值。** 在 shell 配置或 CI 配置中设置 `ADORA_LOG_LEVEL` 和 `ADORA_LOG_FORMAT`。个人开发者可以用 CLI 标志覆盖。

**开发时使用 `--log-filter`。** 无需更改 YAML 配置，使用每节点显示覆盖来关注你正在调试的节点：`--log-filter "my-node=debug"`。

**使用 `send_logs_as` 进行运维监控。** 构建监控节点来监视错误模式、计算错误率或转发告警。这将监控逻辑保留在数据流图内。使用 `adora-log-utils` 在自定义接收器节点中解析和格式化日志条目（参见 `examples/log-sink-file/` 和 `examples/log-sink-tcp/`）。

**结构化数据优先选择 `send_logs_as` 而非 `send_stdout_as`。** `send_stdout_as` 捕获每个 stdout 行（包括原始打印），而 `send_logs_as` 仅捕获带完整元数据的解析结构化日志条目。

**使用 `--local` 进行事后调试。** 崩溃后，`adora logs --local --all-nodes` 在不需要运行 coordinator 的情况下工作，并按时间顺序合并所有节点日志。

**结合 `--since` 和 `--grep` 进行定向调试。** 无需滚动数千行，缩小窗口：`adora logs --local sensor --since 5m --grep "error"`。

**使用 JSON 格式用于日志管道。** 当将日志馈送到外部系统（ELK、Grafana Loki、Datadog）时，使用 `--log-format json` 进行结构化接入。
