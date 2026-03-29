> **[English](../../concepts/dataflow-yaml.md)**

# 数据流 YAML 规范

数据流在 YAML 文件中定义。每个文件描述一个节点图、它们的输入/输出以及执行参数。

仓库根目录提供了 JSON Schema（`adora-schema.json`），可用于编辑器自动补全和验证。

## 快速入门

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

使用 `adora run dataflow.yml`（本地模式）或 `adora up && adora start dataflow.yml`（网络模式）运行。

## 根级字段

| 字段 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `nodes` | 列表 | **必填** | 节点配置列表 |
| `strict_types` | bool | `false` | 在 `validate` 和 `build` 中将类型警告视为错误 |
| `type_rules` | 列表 | `[]` | 用户定义的类型兼容性规则（见[类型注解](types.md#用户定义的兼容性规则)） |
| `health_check_interval` | float | `5.0` | daemon 健康检查扫描的间隔秒数 |
| `_unstable_deploy` | 对象 | -- | 根级部署配置 |
| `_unstable_debug` | 对象 | -- | 调试选项 |

## 节点配置

每个节点需要一个 `id`。所有其他字段都是可选的（但大多数节点至少需要 `path` 或 `operator`/`operators`）。

### 身份

| 字段 | 类型 | 描述 |
|------|------|------|
| `id` | string | **必填。** 唯一标识符。不能包含 `/`。不建议使用空格 |
| `name` | string | 人类可读的显示名称（仅元数据） |
| `description` | string | 文档字符串（仅元数据，运行时不使用） |

### 数据 I/O

#### 输入

输入使用格式 `<node-id>/<output-id>` 订阅另一个节点的输出：

```yaml
inputs:
  # 简写形式
  image: camera/frames
  tick: adora/timer/millis/100

  # 带选项的详写形式
  sensor_data:
    source: sensor/frames
    queue_size: 10
    queue_policy: drop_oldest
    input_timeout: 5.0

  # 无损输入（队列满时阻塞发送方）
  commands:
    source: controller/cmd
    queue_size: 100
    queue_policy: backpressure
```

| 输入选项 | 类型 | 默认值 | 描述 |
|----------|------|--------|------|
| `source` | string | **必填** | `<node-id>/<output-id>` 或定时器路径 |
| `queue_size` | integer | `10` | 输入缓冲区大小 |
| `queue_policy` | string | `drop_oldest` | `drop_oldest`：队列满时丢弃最旧消息。`backpressure`：最多缓冲 `queue_size` 的 10 倍后再丢弃 |
| `input_timeout` | float | -- | 断路器超时（秒）。如果在此期间没有消息到达，daemon 关闭该输入 |

#### 内置定时器

```yaml
inputs:
  tick: adora/timer/millis/100   # 每 100ms
  slow: adora/timer/millis/1000  # 每 1s
  fast: adora/timer/hz/30        # 30 Hz（约 33ms）
```

#### 输出

节点产生的输出标识符列表：

```yaml
outputs:
  - processed_image
  - metadata
```

### 类型注解

输入和输出的可选类型注解。类型永远不是必需的 -- 未注解的端口保持完全动态。

```yaml
- id: camera
  path: camera.py
  outputs:
    - image
  output_types:
    image: std/media/v1/Image

- id: detector
  path: detect.py
  inputs:
    image: camera/image
  input_types:
    image: std/media/v1/Image
  outputs:
    - bbox
  output_types:
    bbox: std/vision/v1/BoundingBox
```

使用 `adora validate <file>` 静态检查类型注解。

### 容错

| 字段 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `restart_policy` | string | `never` | `never`、`on-failure` 或 `always` |
| `max_restarts` | integer | `0` | 最大重启次数。0 = 无限 |
| `restart_delay` | float | -- | 初始退避延迟（秒）。每次翻倍 |
| `max_restart_delay` | float | -- | 指数退避上限 |
| `restart_window` | float | -- | 重启计数时间窗口（秒） |
| `health_check_timeout` | float | -- | 节点不与 daemon 通信的超时时间（秒），超时则杀死进程 |

### 部署

使用 `_unstable_deploy` 将节点分配到特定机器：

```yaml
- id: camera-driver
  _unstable_deploy:
    machine: robot-arm
  path: ./target/debug/camera
  outputs:
    - frames
```

| 部署字段 | 类型 | 默认值 | 描述 |
|----------|------|--------|------|
| `machine` | string | -- | 目标机器/daemon ID |
| `working_dir` | string | -- | 目标机器上的工作目录 |
| `labels` | 对象 | -- | 用于调度的键值标签 |
| `distribute` | string | `local` | 构建产物到达目标 daemon 的方式：`local`、`scp`、`http` |

## 算子节点

算子在共享运行时的进程内运行（无独立进程）。使用 `operator` 表示单个算子，使用 `operators` 表示多个。

```yaml
- id: detector
  operator:
    python: detect.py
    build: pip install -r requirements.txt
    inputs:
      image: camera/frames
    outputs:
      - bbox
```

## 完整示例

```yaml
health_check_interval: 10.0

_unstable_debug:
  publish_all_messages_to_zenoh: true

nodes:
  - id: webcam
    operator:
      python: webcam.py
      inputs:
        tick: adora/timer/millis/100
      outputs:
        - image

  - id: detector
    operator:
      python: detect.py
      build: pip install ultralytics
      inputs:
        image: webcam/image
      outputs:
        - bbox

  - id: plotter
    operator:
      python: plot.py
      inputs:
        image: webcam/image
        bbox: detector/bbox

  - id: logger
    path: ./logger
    inputs:
      bbox: detector/bbox
    send_stdout_as: logs
    min_log_level: info
    restart_policy: on-failure
    max_restarts: 3
    outputs:
      - logs
```
