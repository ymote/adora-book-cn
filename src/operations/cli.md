> 🌐 **[English](../../operations/cli.md)**

# Adora CLI 参考

Adora（AI-Dora，Dataflow-Oriented Robotic Architecture）是一个 100% Rust 编写的实时机器人与 AI 应用框架。本文档涵盖 `adora` CLI 的使用。

## 目录

- [快速入门](#快速入门)
- [安装](#安装)
- [核心概念](#核心概念)
- [数据流描述符](#数据流描述符)
- [命令参考](#命令参考)
  - [生命周期命令](#生命周期命令)
  - [监控命令](#监控命令)
  - [调试命令](#调试命令)
  - [设置命令](#设置命令)
  - [实用命令](#实用命令)
  - [自管理命令](#自管理命令)
- [环境变量](#环境变量)
- [架构指南](#架构指南)
- [编写节点](#编写节点)
- [编写算子](#编写算子)
- [分布式部署](#分布式部署)
- [故障排除](#故障排除)
- **API 参考：** [Rust](api-rust.md) | [Python](api-python.md) | [C](api-c.md) | [C++](api-cxx.md)

---

## 快速入门

```bash
# 创建新项目
adora new my-robot --kind dataflow --lang rust

# 本地运行（无需 coordinator/daemon）
adora run dataflow.yml

# 或使用 coordinator/daemon 用于生产环境
adora up
adora start dataflow.yml --attach
# Ctrl-C 停止
adora down
```

## 安装

### 从 crates.io 安装（推荐）

```bash
cargo install adora-cli
```

### 从源码安装

```bash
cargo install --path binaries/cli --locked
```

### 验证

```bash
adora --version
adora status
```

---

## 核心概念

### 数据流

**数据流**是由节点组成的有向图，节点之间通过类型化数据通道连接。节点产生**输出**，其他节点作为**输入**消费。框架处理数据路由、序列化（Apache Arrow）和生命周期管理。

### 执行模式

| 模式 | 命令 | 基础设施 | 用途 |
|------|------|----------|------|
| **本地** | `adora run` | 无 | 开发、测试、单机 |
| **分布式** | `adora up` + `adora start` | Coordinator + Daemon | 生产环境、多机器 |

### 组件角色

```
CLI  -->  Coordinator  -->  Daemon(s)  -->  节点 / 算子
              (控制平面)     (每台机器)       (用户代码)
```

- **CLI**：用户界面。发送命令，显示日志。
- **Coordinator**：跨机器编排数据流生命周期。
- **Daemon**：生成节点进程，管理 IPC，收集指标。
- **节点**：产生和消费 Arrow 数据的独立进程。
- **算子**：在共享运行时内运行的进程内代码（比节点延迟更低）。

### 数据格式

所有数据以 **Apache Arrow** 列式数组的形式在系统中流动。这使得同机节点之间可以进行零拷贝共享内存传输，且零序列化开销。

---

## 数据流描述符

数据流在 YAML 文件中定义。以下是完整的 schema：

### 最小示例

```yaml
nodes:
  - id: sender
    path: sender.py
    outputs:
      - message

  - id: receiver
    path: receiver.py
    inputs:
      message: sender/message
```

### 完整 Schema

```yaml
# 数据流级设置
health_check_interval: 5.0    # 健康检查扫描间隔（秒）（默认：5.0）

nodes:
  - id: my-node                 # 唯一标识符（必填）
    name: "My Node"             # 人类可读名称（可选）
    description: "..."          # 描述（可选）

    # --- 来源（选择其一） ---
    path: ./target/debug/my-node          # 本地可执行文件
    # path: https://example.com/node.zip  # 从 URL 下载
    # git: https://github.com/org/repo.git  # 从 git 构建
    #   branch: main            # git 分支（与 tag/rev 互斥）
    #   tag: v1.0               # git 标签
    #   rev: abc123             # git 提交哈希

    # --- 构建 ---
    build: cargo build -p my-node   # 构建 shell 命令（可选）

    # --- 输入 ---
    inputs:
      # 简写形式：source_node/output_id
      tick: adora/timer/millis/100
      data: other-node/output

      # 带选项的完整形式
      sensor_data:
        source: sensor/frames
        queue_size: 10            # 输入缓冲区大小（默认：10）
        queue_policy: drop_oldest # 或 "backpressure"（缓冲最多 10 倍 queue_size）
        input_timeout: 5.0        # 断路器超时（秒）

    # --- 输出 ---
    outputs:
      - processed
      - status

    # --- 环境变量 ---
    env:
      MY_VAR: "value"
      FROM_ENV:
        __adora_env: HOST_VAR     # 从主机环境读取
    args: "--verbose"             # 命令行参数

    # --- 容错 ---
    restart_policy: on-failure    # never（默认）| on-failure | always
    max_restarts: 5               # 0 = 无限
    restart_delay: 1.0            # 初始退避延迟（秒）
    max_restart_delay: 30.0       # 退避上限（秒）
    restart_window: 300.0         # N 秒后重置计数器
    health_check_timeout: 30.0    # N 秒无活动则终止

    # --- 日志 ---
    min_log_level: info           # 源级过滤（daemon 端）
    send_stdout_as: raw_output    # 将原始 stdout 作为数据输出路由
    send_logs_as: log_entries     # 将结构化日志作为数据输出路由
    max_log_size: "50MB"          # 日志文件达到此大小时轮转
    max_rotated_files: 5          # 保留的轮转文件数（1-100）

    # --- 部署 ---
    _unstable_deploy:
      machine: A                  # 目标机器/daemon ID

# 调试设置
_unstable_debug:
  publish_all_messages_to_zenoh: true   # topic echo/hz/info 所需
```

### 内置定时器节点

定时器是以固定间隔发送 tick 的虚拟节点：

```yaml
inputs:
  tick: adora/timer/millis/100   # 每 100ms
  slow: adora/timer/millis/1000  # 每 1s
  fast: adora/timer/hz/30        # 30 Hz（约 33ms）
```

### 算子节点

算子在共享运行时内进程内运行（无单独进程）：

```yaml
nodes:
  # 单个算子（简写）
  - id: detector
    operator:
      python: detect.py
      build: pip install -r requirements.txt
      inputs:
        image: camera/frames
      outputs:
        - bbox

  # 共享运行时的多个算子
  - id: runtime-node
    operators:
      - id: preprocessor
        shared-library: ../../target/debug/libpreprocess
        inputs:
          raw: sensor/data
        outputs:
          - processed
      - id: analyzer
        shared-library: ../../target/debug/libanalyze
        inputs:
          data: runtime-node/preprocessor/processed
        outputs:
          - result
```

### 分布式部署

使用 `_unstable_deploy` 将节点分配到特定机器：

```yaml
nodes:
  - id: camera-driver
    _unstable_deploy:
      machine: robot-arm
    path: ./target/debug/camera
    outputs:
      - frames

  - id: ml-inference
    _unstable_deploy:
      machine: gpu-server
    path: ./target/debug/inference
    inputs:
      frames: camera-driver/frames
    outputs:
      - predictions
```

当节点在不同机器上时，通信自动从共享内存切换到 Zenoh 发布/订阅。

---

## 命令参考

### 生命周期命令

#### `adora run`

在本地运行数据流，无需 coordinator 或 daemon。最适合开发和测试。

```
adora run <PATH> [OPTIONS]
```

| 参数/标志 | 默认值 | 描述 |
|-----------|--------|------|
| `<PATH>` | 必填 | 数据流描述符 YAML 路径 |
| `--stop-after <DURATION>` | | 指定时长后自动停止（如 `30s`、`5m`） |
| `--uv` | false | 使用 `uv` 管理 Python 节点 |
| `--debug` | false | 启用调试主题（等同于 `publish_all_messages_to_zenoh: true`） |
| `--allow-shell-nodes` | false | 启用基于 shell 的节点执行 |
| `--log-level <LEVEL>` | `stdout` | 最小显示级别：`error\|warn\|info\|debug\|trace\|stdout` |
| `--log-format <FORMAT>` | `pretty` | 输出格式：`pretty\|json\|compact` |
| `--log-filter <FILTER>` | | 每节点级别覆盖：`"node1=debug,node2=warn"` |

**示例：**

```bash
# 基本运行
adora run dataflow.yml

# 10 秒后停止，仅显示警告
adora run dataflow.yml --stop-after 10s --log-level warn

# 使用 uv 的 Python 数据流
adora run dataflow.yml --uv

# 调试一个节点，静默其他
adora run dataflow.yml --log-level warn --log-filter "sensor=debug"

# CI 管道的 JSON 输出
adora run dataflow.yml --log-format json --stop-after 30s 2>test.json
```

#### `adora up`

以本地模式启动 coordinator 和 daemon。

```
adora up
```

作为后台进程生成 `adora coordinator` 和 `adora daemon`。等待两者就绪后返回。幂等操作：如果已在运行，则不执行任何操作。

#### `adora down`（别名：`adora destroy`）

关闭 coordinator 和 daemon。首先停止所有运行中的数据流。

```
adora down [OPTIONS]
```

| 标志 | 默认值 | 描述 |
|------|--------|------|
| `--coordinator-addr <IP>` | `127.0.0.1` | Coordinator 地址 |
| `--coordinator-port <PORT>` | `6013` | Coordinator 端口 |

#### `adora build`

运行数据流描述符中定义的构建命令。

```
adora build <PATH> [OPTIONS]
```

| 标志 | 默认值 | 描述 |
|------|--------|------|
| `<PATH>` | 必填 | 数据流描述符路径 |
| `--uv` | false | 使用 `uv` 进行 Python 构建 |
| `--local` | false | 强制本地构建（跳过 coordinator） |
| `--strict-types` | false | 将类型警告视为错误（非零退出码） |

**类型检查：** 展开模块后，`build` 运行与 `validate` 相同的类型检查。默认打印警告；使用 `--strict-types`（或在 YAML 中设置 `strict_types: true`）可在类型不匹配时使构建失败。数据流旁边 `types/` 目录中的用户定义类型会自动加载。

**构建策略：** 如果节点有 `_unstable_deploy` 部分且 coordinator 可达，构建会分发到目标机器。否则在本地构建。

**Git 来源：** 带有 `git:` 字段的节点在构建前会被克隆/更新。构建命令从 git 仓库根目录运行。

#### `adora start`

在运行中的 coordinator 上启动数据流。

```
adora start <PATH> [OPTIONS]
```

| 标志 | 默认值 | 描述 |
|------|--------|------|
| `<PATH>` | 必填 | 数据流描述符路径 |
| `--name <NAME>`, `-n` | | 为数据流分配名称 |
| `--attach` | 自动 | 附加到日志流并等待完成 |
| `--detach` | 自动 | 生成后立即返回 |
| `--debug` | false | 启用调试主题（等同于 `publish_all_messages_to_zenoh: true`） |
| `--hot-reload` | false | 监视 Python 文件并在更改时重新加载 |
| `--uv` | false | 使用 `uv` 管理 Python 节点 |
| `--coordinator-addr <IP>` | `127.0.0.1` | Coordinator 地址 |
| `--coordinator-port <PORT>` | `6013` | Coordinator 端口 |

如果未指定 `--attach` 或 `--detach`：在 TTY 中运行时附加，否则分离。

**附加模式：** 流式传输日志，优雅处理 Ctrl-C（第一次 = 停止，第二次 = 强制终止）。

**热重载：** 监视 Python 算子源文件。更改时向 coordinator 发送重载请求，coordinator 传播到 daemon。

#### `adora stop`

停止运行中的数据流。

```
adora stop [UUID_OR_NAME] [OPTIONS]
```

| 标志 | 默认值 | 描述 |
|------|--------|------|
| `[UUID_OR_NAME]` | 交互式 | 数据流 UUID 或名称 |
| `--name <NAME>`, `-n` | | 替代名称指定 |
| `--grace-duration <DURATION>` | | 优雅关闭超时 |
| `--force`, `-f` | false | 立即终止 |
| `--coordinator-addr <IP>` | `127.0.0.1` | Coordinator 地址 |
| `--coordinator-port <PORT>` | `6013` | Coordinator 端口 |

如果未给出标识符且在 TTY 中运行，将呈现交互式选择器。

**停止顺序：** 发送 Event::Stop -> 等待优雅期 -> SIGTERM -> 强制终止。

#### `adora restart`

重启运行中的数据流（停止 + 使用存储的描述符重新启动）。无需 YAML 路径 -- coordinator 保留了原始描述符。

```
adora restart [UUID] [OPTIONS]
```

| 标志 | 默认值 | 描述 |
|------|--------|------|
| `[UUID]` | | 数据流 UUID |
| `--name <NAME>`, `-n` | | 按名称而非 UUID 重启 |
| `--grace-duration <DURATION>` | | 停止阶段的优雅关闭超时 |
| `--force`, `-f` | false | 重启前强制终止 |
| `--coordinator-addr <IP>` | `127.0.0.1` | Coordinator 地址 |
| `--coordinator-port <PORT>` | `6013` | Coordinator 端口 |

**示例：**

```bash
# 按名称重启
adora restart --name my-app

# 按 UUID 强制重启
adora restart a1b2c3d4-... --force
```

#### `adora record`

将数据流消息录制到 `.adorec` 文件，用于离线回放。详见[调试指南](debugging.md#录制与回放)。

```
adora record <DATAFLOW_YAML> [OPTIONS]
```

| 标志 | 默认值 | 描述 |
|------|--------|------|
| `<DATAFLOW_YAML>` | 必填 | 数据流描述符路径 |
| `-o, --output <PATH>` | `recording_{timestamp}.adorec` | 输出文件路径 |
| `--topics <TOPICS>` | 全部 | 逗号分隔的 `node/output` 主题 |
| `--proxy` | false | 通过 WebSocket 流式传输而非在目标上录制 |
| `--output-yaml <PATH>` | | 写出修改后的 YAML 而不运行（试运行） |

默认模式在数据流中注入一个录制节点。`--proxy` 模式需要运行中的数据流和 `publish_all_messages_to_zenoh: true`。

#### `adora replay`

回放录制的 `.adorec` 文件，用回放节点替换源节点。详见[调试指南](debugging.md#回放录制)。

```
adora replay <FILE> [OPTIONS]
```

| 标志 | 默认值 | 描述 |
|------|--------|------|
| `<FILE>` | 必填 | `.adorec` 录制文件路径 |
| `--speed <FLOAT>` | `1.0` | 播放速度（0 = 最大速度） |
| `--loop` | false | 循环播放 |
| `--replace <NODE_IDS>` | 所有已录制 | 逗号分隔的要替换的节点 |
| `--output-yaml <PATH>` | | 写出修改后的 YAML 而不运行（试运行） |

---

### 监控命令

#### `adora list`（别名：`adora ps`）

列出运行中的数据流及其指标。

```
adora list [OPTIONS]
```

| 标志 | 默认值 | 描述 |
|------|--------|------|
| `--format <FMT>`, `-f` | `table` | 输出格式：`table\|json` |
| `--status <STATUS>` | | 过滤：`running\|finished\|failed` |
| `--name <PATTERN>` | | 按名称过滤（不区分大小写的子串） |
| `--sort-by <FIELD>` | | 排序：`cpu\|memory` |
| `--quiet`, `-q` | false | 仅打印 UUID |
| `--coordinator-addr <IP>` | `127.0.0.1` | Coordinator 地址 |
| `--coordinator-port <PORT>` | `6013` | Coordinator 端口 |

**输出列：** UUID, Name, Status, Nodes, CPU, Memory

#### `adora logs`

显示和跟踪数据流和节点的日志。

```
adora logs [UUID_OR_NAME] [NODE] [OPTIONS]
```

| 标志 | 默认值 | 描述 |
|------|--------|------|
| `[UUID_OR_NAME]` | | 数据流 UUID 或名称 |
| `[NODE]` | | 节点名称（除非 `--all-nodes` 否则必填） |
| `--all-nodes` | false | 按时间戳合并所有节点的日志 |
| `--tail <N>` | 全部 | 显示最后 N 行 |
| `--follow`, `-f` | false | 流式传输新日志条目 |
| `--local` | false | 从本地 `out/` 目录读取 |
| `--since <DURATION>` | | 显示此时长前之后的日志 |
| `--until <DURATION>` | | 显示此时长前之前的日志 |
| `--level <LEVEL>` | `stdout` | 最小日志级别 |
| `--log-format <FORMAT>` | `pretty` | 输出格式 |
| `--log-filter <FILTER>` | | 每节点级别覆盖 |
| `--grep <PATTERN>` | | 不区分大小写的文本搜索 |
| `--coordinator-addr <IP>` | `127.0.0.1` | Coordinator 地址 |
| `--coordinator-port <PORT>` | `6013` | Coordinator 端口 |

**过滤管道：** 读取/解析 -> 时间过滤 -> Grep -> Tail -> 显示

**示例：**

```bash
# 实时跟踪所有节点
adora logs my-dataflow --all-nodes --follow

# 特定节点的最后 50 条错误
adora logs my-dataflow sensor --level error --tail 50

# 搜索最近 5 分钟的日志
adora logs my-dataflow --all-nodes --since 5m --grep "timeout"

# 读取本地文件（无需 coordinator）
adora logs --local --all-nodes --tail 100

# 事后分析：时间窗口内的错误
adora logs --local sensor --since 1h --until 30m --level error
```

**时长格式：** `30`（秒）、`30s`、`5m`、`1h`、`2d`

#### `adora inspect top`（别名：`adora top`）

节点资源使用的实时 TUI 监视器（类似 `top`）。

```
adora inspect top [OPTIONS]
adora top [OPTIONS]
```

| 标志 | 默认值 | 描述 |
|------|--------|------|
| `--refresh-interval <SECONDS>` | `2` | 更新间隔（最小：1） |
| `--once` | false | 打印单个 JSON 快照并退出（用于脚本/CI） |
| `--coordinator-addr <IP>` | `127.0.0.1` | Coordinator 地址 |
| `--coordinator-port <PORT>` | `6013` | Coordinator 端口 |

**需要交互式终端**（除非使用 `--once`）。

| 按键 | 操作 |
|------|------|
| `q` / `Esc` | 退出 |
| `Up` / `k` | 选择上一个节点 |
| `Down` / `j` | 选择下一个节点 |
| `n` | 按节点名称排序 |
| `c` | 按 CPU 排序 |
| `m` | 按内存排序 |
| `r` | 强制刷新 |

**列：** NODE, STATUS, DATAFLOW, PID, CPU%, MEMORY (MB), RESTARTS, QUEUE, NET TX, NET RX, I/O READ (MB/s), I/O WRITE (MB/s)

- **STATUS**：Running、Restarting、Degraded（断开的输入）或 Failed
- **RESTARTS**：每节点当前重启次数
- **QUEUE**：节点输入队列中的待处理消息
- **NET TX/RX**：通过 Zenoh 跨 daemon 发送/接收的累计网络字节

CPU 值按每核计算（多核可超过 100%）。指标来自 daemon，因此适用于分布式部署。

**脚本示例：**

```bash
# CI/监控管道的 JSON 快照
adora top --once | jq '.[].cpu_usage'
```

#### `adora topic list`

列出运行中数据流的所有主题（输出）。

```
adora topic list [OPTIONS]
```

| 标志 | 默认值 | 描述 |
|------|--------|------|
| `-d <DATAFLOW>`, `--dataflow` | 交互式 | 数据流 UUID 或名称 |
| `--format <FMT>` | `table` | 输出格式：`table\|json` |

#### `adora topic echo`

订阅主题并实时显示消息。

```
adora topic echo [OPTIONS] [DATA...]
```

| 标志 | 默认值 | 描述 |
|------|--------|------|
| `-d <DATAFLOW>`, `--dataflow` | 必填 | 数据流 UUID 或名称 |
| `[DATA...]` | 所有输出 | 要回显的主题（如 `node1/output`） |
| `--format <FMT>` | `table` | 输出格式：`table\|json` |

需要描述符中的 `_unstable_debug.publish_all_messages_to_zenoh: true`。

#### `adora topic hz`

使用 TUI 仪表板测量主题发布频率。

```
adora topic hz [OPTIONS] [DATA...]
```

| 标志 | 默认值 | 描述 |
|------|--------|------|
| `-d <DATAFLOW>`, `--dataflow` | 必填 | 数据流 UUID 或名称 |
| `[DATA...]` | 所有输出 | 要测量的主题 |
| `--window <SECONDS>` | `10` | 滑动窗口（最小：1） |

**需要交互式终端。** 显示：平均值 (ms)、平均值 (Hz)、最小值 (ms)、最大值 (ms)、标准差 (ms)，以及选定主题的频率迷你图和直方图。

#### `adora topic info`

显示单个主题的详细元数据。

```
adora topic info [OPTIONS] DATA
```

| 标志 | 默认值 | 描述 |
|------|--------|------|
| `-d <DATAFLOW>`, `--dataflow` | 必填 | 数据流 UUID 或名称 |
| `DATA` | 必填 | 单个主题（如 `camera/image`） |
| `--duration <SECONDS>` | `5` | 收集时长（最小：1） |

在指定时长内订阅主题并报告：类型（Arrow schema）、发布者、订阅者、消息数量、带宽。

#### `adora node`

管理和检查数据流节点。

##### `adora node list`

```
adora node list [OPTIONS]
```

列出运行中数据流的节点及其状态、CPU、内存和重启次数。

**列：** NODE, STATUS, PID, CPU%, MEMORY (MB), RESTARTS, DATAFLOW

##### `adora node info`

显示特定节点的详细信息，包括状态、输入、输出和指标。

```
adora node info <NODE> [OPTIONS]
```

| 标志 | 默认值 | 描述 |
|------|--------|------|
| `<NODE>` | 必填 | 要检查的节点 ID |
| `-d <DATAFLOW>`, `--dataflow` | 交互式 | 数据流 UUID 或名称 |
| `-f <FORMAT>`, `--format` | `table` | 输出格式：`table\|json` |

##### `adora node restart`

重启运行中数据流中的单个节点。daemon 停止节点进程并重新生成。

```
adora node restart <NODE> [OPTIONS]
```

| 标志 | 默认值 | 描述 |
|------|--------|------|
| `<NODE>` | 必填 | 要重启的节点 ID |
| `-d <DATAFLOW>`, `--dataflow` | 交互式 | 数据流 UUID 或名称 |
| `--grace <DURATION>` | | 强制终止节点前的优雅期 |

##### `adora node stop`

停止运行中数据流中的单个节点，不停止整个数据流。

```
adora node stop <NODE> [OPTIONS]
```

| 标志 | 默认值 | 描述 |
|------|--------|------|
| `<NODE>` | 必填 | 要停止的节点 ID |
| `-d <DATAFLOW>`, `--dataflow` | 交互式 | 数据流 UUID 或名称 |
| `--grace <DURATION>` | | 强制终止节点前的优雅期 |

#### `adora topic pub`

向运行中数据流的主题发布 JSON 数据。需要 `publish_all_messages_to_zenoh: true`。

```
adora topic pub <TOPIC> [DATA] [OPTIONS]
```

| 标志 | 默认值 | 描述 |
|------|--------|------|
| `<TOPIC>` | 必填 | 要发布到的主题（格式：`node_id/output_id`） |
| `[DATA]` | | 要发布的 JSON 数据（除非 `--file` 否则必填） |
| `--file <PATH>` | | 从 JSON 文件读取数据而非命令行 |
| `--count <N>` | `1` | 要发布的消息数量 |
| `-d <DATAFLOW>`, `--dataflow` | 必填 | 数据流 UUID 或名称 |

**示例：**

```bash
# 发布单个值
adora topic pub -d my-app sensor/threshold '[42]'

# 从文件发布，10 次
adora topic pub -d my-app sensor/config --file config.json --count 10
```

#### `adora param`

管理节点的运行时参数。参数持久化在 coordinator 存储中，可选择转发到运行中的节点。

##### `adora param list`

列出节点的所有运行时参数。

```
adora param list <NODE> [OPTIONS]
```

| 标志 | 默认值 | 描述 |
|------|--------|------|
| `<NODE>` | 必填 | 节点 ID |
| `-d <DATAFLOW>`, `--dataflow` | 交互式 | 数据流 UUID 或名称 |
| `--format <FMT>` | `table` | 输出格式：`table\|json` |

##### `adora param get`

获取单个运行时参数值。

```
adora param get <NODE> <KEY> [OPTIONS]
```

| 标志 | 默认值 | 描述 |
|------|--------|------|
| `<NODE>` | 必填 | 节点 ID |
| `<KEY>` | 必填 | 参数键 |
| `-d <DATAFLOW>`, `--dataflow` | 交互式 | 数据流 UUID 或名称 |

##### `adora param set`

设置运行时参数。值为 JSON。参数存储在 coordinator 中，如果节点正在运行则转发到节点。

```
adora param set <NODE> <KEY> <VALUE> [OPTIONS]
```

| 标志 | 默认值 | 描述 |
|------|--------|------|
| `<NODE>` | 必填 | 节点 ID |
| `<KEY>` | 必填 | 参数键（最大 256 字节） |
| `<VALUE>` | 必填 | JSON 格式的参数值（序列化后最大 64KB） |
| `-d <DATAFLOW>`, `--dataflow` | 交互式 | 数据流 UUID 或名称 |

**示例：**

```bash
# 设置数值参数
adora param set -d my-app sensor threshold 42

# 设置字符串参数
adora param set -d my-app camera resolution '"1080p"'

# 设置复杂参数
adora param set -d my-app detector config '{"confidence": 0.8, "nms": 0.5}'
```

##### `adora param delete`

删除运行时参数。

```
adora param delete <NODE> <KEY> [OPTIONS]
```

| 标志 | 默认值 | 描述 |
|------|--------|------|
| `<NODE>` | 必填 | 节点 ID |
| `<KEY>` | 必填 | 参数键 |
| `-d <DATAFLOW>`, `--dataflow` | 交互式 | 数据流 UUID 或名称 |

#### `adora doctor`

诊断环境、coordinator/daemon 连接性，以及可选地验证数据流 YAML。

```
adora doctor [OPTIONS]
```

| 标志 | 默认值 | 描述 |
|------|--------|------|
| `--dataflow <PATH>` | | 要验证的数据流 YAML 路径 |

执行的检查：
1. Coordinator 可达性
2. Daemon 连接性
3. 活跃数据流状态
4. 数据流 YAML 验证（如果提供了 `--dataflow`）

**示例：**

```bash
# 基本健康检查
adora doctor

# 检查环境 + 验证数据流
adora doctor --dataflow dataflow.yml
```

#### `adora trace list`

列出 coordinator 捕获的最近追踪。coordinator 在内存中从 `adora_coordinator` 和 `adora_core` crate 捕获 span（最多 4096 个 span）。无需外部追踪基础设施。

```
adora trace list [OPTIONS]
```

| 标志 | 默认值 | 描述 |
|------|--------|------|
| `--coordinator-addr <IP>` | `127.0.0.1` | Coordinator 地址 |
| `--coordinator-port <PORT>` | `6013` | Coordinator 端口 |

**输出列：** TRACE ID（前 12 个字符）、ROOT SPAN、SPANS、STARTED、DURATION

**示例：**

```bash
adora trace list
```

```
TRACE ID      ROOT SPAN          SPANS  STARTED              DURATION
a1b2c3d4e5f6  spawn_dataflow     12     2026-03-01 10:30:05  1.234s
f8e7d6c5b4a3  build_dataflow     5      2026-03-01 10:29:58  0.500s
```

#### `adora trace view`

以缩进树形式查看特定追踪的 span。支持追踪 ID 前缀匹配。

```
adora trace view <TRACE_ID> [OPTIONS]
```

| 参数/标志 | 默认值 | 描述 |
|-----------|--------|------|
| `<TRACE_ID>` | 必填 | 完整追踪 ID 或唯一前缀 |
| `--coordinator-addr <IP>` | `127.0.0.1` | Coordinator 地址 |
| `--coordinator-port <PORT>` | `6013` | Coordinator 端口 |

**示例：**

```bash
adora trace view a1b2c3d4
```

```
spawn_dataflow [INFO 1.234s] {build_id="abc", session_id="def"}
  build_dataflow [INFO 0.500s]
    download_node [DEBUG 0.200s] {url="..."}
  start_inner [INFO 0.734s]
    spawn_node [INFO 0.100s] {node_id="camera"}
    spawn_node [INFO 0.080s] {node_id="detector"}
```

追踪 ID 使用前缀匹配：如果前缀唯一标识一个追踪，则自动解析。如果有歧义，系统会提示使用更长的前缀。

---

### 设置命令

#### `adora status`（别名：`adora check`）

检查系统健康状况和连接性。

```
adora status [OPTIONS]
```

报告 coordinator 连接性、daemon 状态和活跃数据流数量。

#### `adora new`

从模板生成新项目或节点。

```
adora new <NAME> [OPTIONS]
```

| 标志 | 默认值 | 描述 |
|------|--------|------|
| `<NAME>` | 必填 | 项目或节点名称 |
| `--kind <KIND>` | `dataflow` | `dataflow\|node` |
| `--lang <LANG>` | `rust` | `rust\|python\|c\|cxx` |

#### `adora expand`

展开数据流中的模块引用并打印结果的扁平 YAML。用于调试模块组合。

```
adora expand <PATH> [OPTIONS]
```

| 标志 | 默认值 | 描述 |
|------|--------|------|
| `<PATH>` | 必填 | 数据流描述符（或使用 `--module` 的模块文件） |
| `--module` | false | 验证独立模块文件而非完整数据流 |

**示例：**

```bash
# 展开带模块的数据流
adora expand dataflow.yml

# 验证模块文件
adora expand --module modules/navigation.module.yml
```

详见[模块指南](modules.md)以获取模块组合的完整文档。

#### `adora graph`

将数据流可视化为图形。

```
adora graph <PATH> [OPTIONS]
```

| 标志 | 默认值 | 描述 |
|------|--------|------|
| `<PATH>` | 必填 | 数据流描述符路径 |
| `--mermaid` | false | 输出 Mermaid 图表文本 |
| `--open` | false | 在浏览器中打开 HTML |

不使用 `--mermaid` 时，使用 mermaid.js 生成交互式 HTML 文件。当输出有类型标注时，边标签包含类型名称（如 `image [Image]`）。

```bash
# 生成 HTML
adora graph dataflow.yml --open

# 生成用于 GitHub markdown 的 Mermaid
adora graph dataflow.yml --mermaid
```

#### `adora validate`

验证数据流 YAML 文件并检查类型标注。

```
adora validate <PATH> [OPTIONS]
```

| 标志 | 默认值 | 描述 |
|------|--------|------|
| `<PATH>` | 必填 | 数据流描述符路径 |
| `--strict-types` | false | 将警告视为错误（CI 的非零退出码） |

检查内容：
1. **键存在性**：`output_types`/`input_types` 键存在于对应的 `outputs`/`inputs` 列表中
2. **URN 解析**：所有类型 URN 在标准或用户定义类型库中可解析
3. **边兼容性**：连接的边具有兼容类型（精确匹配、拓宽或用户定义规则）
4. **参数化类型**：参数不匹配（如 `AudioFrame[sample_type=f32]` vs `AudioFrame[sample_type=i16]`）
5. **定时器自动类型**：定时器输入自动类型化为 `std/core/v1/UInt64`
6. **类型推断**：当仅上游标注了类型时，下游输入推断该类型
7. **元数据模式**：验证 `output_metadata` 键和 `pattern` 简写
8. **Schema 兼容性**：在字段级别检查结构体类型（缺失/错误字段）

数据流旁边 `types/` 目录中的用户定义类型会自动加载。

```bash
# 带警告验证
adora validate dataflow.yml

# CI 严格模式（警告时退出码 1）
adora validate --strict-types dataflow.yml
```

详见[类型标注指南](types.md)以获取完整类型库和使用详情。

---

### 实用命令

#### `adora completion`

生成 shell 补全脚本。

```
adora completion [SHELL]
```

如果省略 shell 则自动检测。支持：bash、zsh、fish、elvish、powershell。

```bash
# Bash
eval "$(adora completion bash)"
echo 'eval "$(adora completion bash)"' >> ~/.bashrc

# Zsh
eval "$(adora completion zsh)"
echo 'eval "$(adora completion zsh)"' >> ~/.zshrc

# Fish
adora completion fish > ~/.config/fish/completions/adora.fish
```

#### `adora system`

系统管理命令。

```
adora system status [OPTIONS]
```

目前提供 `status` 作为子命令（等同于 `adora status`）。

---

### 自管理命令

#### `adora self update`

检查并安装 CLI 更新。

```
adora self update [--check-only]
```

从 GitHub releases（`dora-rs/adora`）下载。

#### `adora self uninstall`

从系统中移除 CLI。

```
adora self uninstall [--force]
```

不使用 `--force` 时，提示确认（需要 TTY）。依次尝试 `uv pip uninstall`、`pip uninstall`、二进制自删除。

---

## 环境变量

所有环境变量作为后备。CLI 标志始终优先。

| 变量 | 默认值 | 命令 | 描述 |
|------|--------|------|------|
| `ADORA_COORDINATOR_ADDR` | `127.0.0.1` | 所有 coordinator 命令 | Coordinator IP 地址 |
| `ADORA_COORDINATOR_PORT` | `6013` | 所有 coordinator 命令 | Coordinator WebSocket 端口 |
| `ADORA_LOG_LEVEL` | `stdout` | `run`, `logs` | 默认最小日志级别 |
| `ADORA_LOG_FORMAT` | `pretty` | `run`, `logs` | 默认输出格式 |
| `ADORA_LOG_FILTER` | | `run`, `logs` | 默认每节点级别覆盖 |
| `ADORA_ALLOW_SHELL_NODES` | | `run` | 启用 shell 节点执行 |
| `ADORA_RUNTIME_TYPE_CHECK` | | `run`, `start` | 运行时类型检查：`warn`（记录不匹配）或 `error`（不匹配时失败）。详见[类型标注](types.md#运行时类型检查) |

```bash
# 为开发会话设置默认值
export ADORA_COORDINATOR_ADDR=192.168.1.10
export ADORA_LOG_LEVEL=info
export ADORA_LOG_FORMAT=compact
```

---

## 架构指南

本节面向希望了解框架内部原理、扩展框架或调试问题的开发者。

### 通信栈

```
                    ┌─────────────────────────────────────┐
                    │           CLI (adora)                │
                    │   WebSocket (JSON 请求/应答)         │
                    └─────────────┬───────────────────────┘
                                  │
                    ┌─────────────▼───────────────────────┐
                    │        Coordinator                   │
                    │   WebSocket 控制 + daemon 管理       │
                    │   状态：InMemoryStore | RedbStore    │
                    └──┬──────────────────────────────┬───┘
                       │                              │
          ┌────────────▼──────────┐     ┌─────────────▼──────────┐
          │     Daemon A          │     │     Daemon B           │
          │  (machine: robot)     │     │  (machine: gpu-server) │
          │                       │     │                        │
          │  ┌─────┐  ┌─────┐    │     │  ┌──────┐  ┌───────┐  │
          │  │Node1│  │Node2│    │     │  │Node3 │  │Node4  │  │
          │  └──┬──┘  └──┬──┘    │     │  └──┬───┘  └───┬───┘  │
          │     │shmem    │shmem  │     │     │shmem      │shmem │
          │     └────┬────┘       │     │     └─────┬─────┘      │
          └──────────┼────────────┘     └───────────┼────────────┘
                     │                              │
                     └──────── Zenoh 发布/订阅 ──────┘
                              (跨机器)
```

### 协议层

| 层 | 传输 | 格式 | 用途 |
|----|------|------|------|
| CLI <-> Coordinator | WebSocket | JSON (ControlRequest/Reply) | 命令、日志流 |
| Coordinator <-> Daemon | WebSocket | JSON (DaemonCoordinatorEvent) | 节点生命周期、指标 |
| Daemon <-> 节点（小数据） | TCP / Unix socket | 自定义二进制 | 控制消息、小数据 |
| Daemon <-> 节点（大数据） | 共享内存 | 零拷贝 Arrow | 数据消息 > 4KB |
| Daemon <-> Daemon | Zenoh 发布/订阅 | Arrow + 元数据 | 跨机器数据路由 |

### Coordinator 内部原理

Coordinator 是事件驱动的异步服务器：

```
事件源：
  - CLI WebSocket 连接 (ControlRequest)
  - Daemon WebSocket 连接 (DaemonEvent)
  - 心跳定时器（3 秒间隔）
  - 外部事件（用于嵌入）

事件循环：
  merge_all(cli_events, daemon_events, heartbeat, external)
    -> handle_event()
    -> 更新状态
    -> 持久化到存储（如果是 redb）
    -> 发送回复
```

**关键类型：**

```rust
// 状态
RunningDataflow { uuid, name, descriptor, daemons, node_metrics, ... }
RunningBuild    { build_id, errors, log_subscribers, pending_results, ... }
DaemonConnection { sender, pending_replies, last_heartbeat }

// 存储 trait
trait CoordinatorStore: Send + Sync {
    fn put_dataflow(&self, record: &DataflowRecord) -> Result<()>;
    fn get_dataflow(&self, uuid: &Uuid) -> Result<Option<DataflowRecord>>;
    fn list_dataflows(&self) -> Result<Vec<DataflowRecord>>;
    // ... daemon 和 build 方法
}
```

**存储后端：**
- `memory`（默认）：内存中，重启后丢失。
- `redb`：持久化到磁盘（`~/.adora/coordinator.redb`）。崩溃后存活。需要 `redb-backend` 特性。

```bash
adora coordinator --store redb
adora coordinator --store redb:/custom/path.redb
```

### Daemon 内部原理

Daemon 管理单台机器上的节点进程：

```
每个节点：
  1. 构建（如果指定了构建命令）
  2. 使用 ADORA_NODE_CONFIG 环境变量生成进程
  3. 节点通过 TCP/共享内存握手注册
  4. 在节点之间路由输入/输出
  5. 收集指标（CPU、内存、I/O）
  6. 处理退出时的重启策略
  7. 将日志转发到 coordinator

通信：
  - 共享内存用于 > 4KB 的消息（零拷贝）
  - TCP 用于控制消息和小数据
  - flume 通道用于内部事件路由
```

**指标收集：**

```rust
struct NodeMetrics {
    pid: u32,
    cpu_usage: f32,      // 每核百分比
    memory_mb: f64,
    disk_read_mb_s: Option<f64>,
    disk_write_mb_s: Option<f64>,
    status: NodeStatus,  // Running | Restarting | Degraded | Failed
    restart_count: u32,
    pending_messages: u64,
}
```

### 消息类型

所有组件间消息定义在 `libraries/message/` 中：

```rust
// 节点标识
struct NodeId(String);      // [a-zA-Z0-9_.-]
struct DataId(String);      // 同样的验证规则
type DataflowId = uuid::Uuid;

// 数据元数据
struct Metadata {
    timestamp: uhlc::Timestamp,    // 混合逻辑时钟
    type_info: ArrowTypeInfo,      // Arrow schema
    parameters: MetadataParameters, // 自定义键值对
}

// 节点事件 (daemon -> node)
enum NodeEvent {
    Stop,
    Reload { operator_id },
    Input { id, metadata, data },
    InputClosed { id },
    InputRecovered { id },
    NodeRestarted { id },
    AllInputsClosed,
}
```

### 时间戳

Adora 使用**统一混合逻辑时钟**（UHLC）实现分布式因果性。每条消息携带一个 `uhlc::Timestamp`，在无需同步时钟的情况下跨机器保持因果顺序。

### 零拷贝共享内存

对于大消息（> 4KB），daemon 使用共享内存区域：

1. 发送节点从 daemon 请求共享内存槽
2. Daemon 分配区域并返回 ID
3. 发送节点直接将 Arrow 数据写入共享内存
4. Daemon 通知接收节点区域 ID
5. 接收节点直接从共享内存读取（零拷贝）
6. 接收节点完成后发送释放令牌

这使得大负载的延迟比 ROS2 低 10-17 倍。

---

## 编写节点

### Rust 节点

```rust
use adora_node_api::{AdoraNode, Event, IntoArrow};
use adora_core::config::DataId;

fn main() -> eyre::Result<()> {
    let (mut node, mut events) = AdoraNode::init_from_env()?;

    let output = DataId::from("result".to_owned());

    while let Some(event) = events.recv() {
        match event {
            Event::Input { id, metadata, data } => {
                // 处理输入数据（Arrow 数组）
                let result: u64 = 42;
                node.send_output(
                    output.clone(),
                    metadata.parameters,
                    result.into_arrow(),
                )?;
            }
            Event::Stop(_) => break,
            Event::InputClosed { id } => {
                eprintln!("input {id} closed");
            }
            Event::InputRecovered { id } => {
                eprintln!("input {id} recovered");
            }
            _ => {}
        }
    }
    Ok(())
}
```

**Cargo.toml：**

```toml
[dependencies]
adora-node-api = { workspace = true }
eyre = "0.6"
```

### Python 节点

```python
import pyarrow as pa
from adora import Node

node = Node()

for event in node:
    if event["type"] == "INPUT":
        # event["value"] 是一个 PyArrow 数组
        values = event["value"].to_pylist()
        result = pa.array([sum(values)])
        node.send_output("result", result)
    elif event["type"] == "STOP":
        break
```

### C 节点

```c
#include "node_api.h"

int main() {
    void *ctx = init_adora_context_from_env();
    // ... 使用 adora_next_event / adora_send_output 的事件循环
    free_adora_context(ctx);
    return 0;
}
```

### 节点日志

节点可以发出结构化日志：

**Rust：**

```rust
// 通过 tracing（推荐）
tracing::info!("processing frame {}", frame_id);

// 通过节点 API
node.log_info("processing complete");
node.log_with_fields("info", "reading", None, Some(&fields));
```

**Python：**

```python
import logging
logging.info("processing frame %d", frame_id)

# 或通过节点 API
node.log("info", "processing complete")
```

---

## 编写算子

算子在共享运行时内进程内运行，避免进程生成开销。

### Rust 算子

```rust
use adora_operator_api::{register_operator, AdoraOperator, AdoraOutputSender, AdoraStatus, Event};

#[register_operator]
#[derive(Default)]
pub struct MyOperator {
    counter: u32,
}

impl AdoraOperator for MyOperator {
    fn on_event(
        &mut self,
        event: &Event,
        output_sender: &mut AdoraOutputSender,
    ) -> Result<AdoraStatus, String> {
        match event {
            Event::Input { id, data } => {
                self.counter += 1;
                output_sender.send(
                    "count".to_string(),
                    arrow::array::UInt32Array::from(vec![self.counter]),
                )?;
                Ok(AdoraStatus::Continue)
            }
            Event::Stop => Ok(AdoraStatus::Stop),
            _ => Ok(AdoraStatus::Continue),
        }
    }
}
```

**Cargo.toml：**

```toml
[lib]
crate-type = ["cdylib"]

[dependencies]
adora-operator-api = { workspace = true }
arrow = "53"
```

### Python 算子

```yaml
nodes:
  - id: my-node
    operator:
      python: my_operator.py
      inputs:
        data: source/output
      outputs:
        - result
```

```python
# my_operator.py
class Operator:
    def __init__(self):
        self.counter = 0

    def on_event(self, event, send_output):
        if event["type"] == "INPUT":
            self.counter += 1
            send_output("result", pa.array([self.counter]))
```

---

## 分布式部署

### 设置

```bash
# 机器 A（coordinator + daemon）
adora up

# 机器 B（仅 daemon，指向机器 A 的 coordinator）
adora daemon --interface 0.0.0.0 --coordinator-addr 192.168.1.10 --machine-id B

# 机器 C（同上）
adora daemon --interface 0.0.0.0 --coordinator-addr 192.168.1.10 --machine-id C
```

### 带机器分配的数据流

```yaml
nodes:
  - id: camera
    _unstable_deploy:
      machine: robot
    path: ./camera-driver
    outputs:
      - frames

  - id: inference
    _unstable_deploy:
      machine: gpu-server
    path: ./ml-model
    inputs:
      frames: camera/frames
    outputs:
      - predictions

  - id: actuator
    _unstable_deploy:
      machine: robot
    path: ./actuator-driver
    inputs:
      commands: inference/predictions
```

### 构建和启动

```bash
# 从任何能访问 coordinator 的机器
adora build dataflow.yml       # 在目标机器上分布式构建
adora start dataflow.yml --name my-robot --attach
```

### 监控

```bash
# 跨所有机器的资源使用
adora top

# 任何节点的日志，无论在哪台机器
adora logs my-robot inference --follow

# 列出所有数据流
adora list
```

### Coordinator 持久化

生产环境中，使用 redb 存储后端使 coordinator 在重启后存活：

```bash
adora coordinator --store redb
```

状态持久化到 `~/.adora/coordinator.redb`。重启时，过期的数据流被标记为失败，coordinator 恢复正常运行。

> 有关托管集群部署（cluster.yml、基于 SSH 的生命周期、标签调度、systemd 服务、滚动升级），请参见[分布式部署指南](distributed-deployment.md)。

---

## 故障排除

> 有关涵盖录制/回放工作流、主题检查、资源监控和端到端调试场景的全面调试指南，请参见[调试与可观测性指南](debugging.md)。

### 常见问题

**"Could not connect to adora-coordinator"**
- 先运行 `adora up`，或检查 `ADORA_COORDINATOR_ADDR`/`ADORA_COORDINATOR_PORT`
- 使用 `adora status` 验证

**"publish_all_messages_to_zenoh not enabled"**
- 使用 `--debug` 标志：`adora start dataflow.yml --debug` 或 `adora run dataflow.yml --debug`
- 或添加到数据流 YAML：
  ```yaml
  _unstable_debug:
    publish_all_messages_to_zenoh: true
  ```
- `topic echo`、`topic hz`、`topic info` 所需

**"`adora top` requires an interactive terminal"**
- 这些 TUI 命令需要真正的终端（不能是管道输出）
- `topic hz` 同样适用

**节点未收到输入**
- 检查输出名称匹配：`source_node/output_id`
- 验证源节点在其 `outputs:` 数组中列出了该输出
- 使用 `adora topic list` 检查可用主题

**日志未出现**
- 检查 `--log-level` 设置（默认 `stdout` 显示所有内容）
- 检查 YAML 中的 `min_log_level`（在源端过滤）
- 分布式环境：验证 coordinator/daemon 连接性

**使用 git 源的构建失败**
- 验证 `git:` URL 可访问
- 检查 `branch`、`tag` 或 `rev` 存在
- 构建命令从 git 仓库根目录运行，而非数据流目录

### 调试工作流

```bash
# 1. 完整环境诊断
adora doctor --dataflow dataflow.yml

# 2. 使用详细日志和调试主题启动
adora run dataflow.yml --log-level trace --debug

# 3. 检查特定节点
adora node info -d my-dataflow problem-node

# 4. 监控特定节点日志
adora logs my-dataflow problem-node --follow --level debug

# 5. 检查资源使用
adora top

# 6. 检查主题数据
adora topic echo -d my-dataflow problem-node/output

# 7. 发布测试数据到主题
adora topic pub -d my-dataflow problem-node/input '[1, 2, 3]'

# 8. 测量频率
adora topic hz -d my-dataflow --window 5

# 9. 查看/修改运行时参数
adora param list -d my-dataflow problem-node
adora param set -d my-dataflow problem-node threshold 42

# 10. 重启行为异常的节点而不停止数据流
adora node restart -d my-dataflow problem-node

# 11. 查看 coordinator 追踪（无需外部基础设施）
adora trace list
adora trace view <trace-id-prefix>

# 12. 可视化数据流图
adora graph dataflow.yml --open
```

### 日志文件位置

```
out/
  <dataflow-uuid>/
    log_<node-id>.jsonl          # 当前日志
    log_<node-id>.1.jsonl        # 轮转的（上一个）
    log_<node-id>.2.jsonl        # 轮转的（更早）
```

直接读取：

```bash
adora logs --local --all-nodes
adora logs --local <node-name> --tail 50
```
