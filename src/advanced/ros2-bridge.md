> 🌐 **[English](../../advanced/ros2-bridge.md)**

# ROS2 桥接

Adora 提供基于 YAML 声明式的 ROS2 桥接，让任何 Adora 节点无需导入 ROS2 库即可与 ROS2 topics、services 和 actions 通信。你只需在数据流 YAML 中使用 `ros2:` 键定义桥接，框架会自动生成一个桥接二进制文件，在 Apache Arrow（Adora 的原生格式）和 ROS2 CDR/DDS 之间转换。你的用户节点无需 ROS2 -- 它们发送和接收纯 Arrow `StructArray` 数据。

## 功能概览

| 功能 | 配置 | 描述 |
|------|------|------|
| Topic 订阅 | `topic` + `direction: subscribe` | 从 ROS2 接收，转发为 Arrow |
| Topic 发布 | `topic` + `direction: publish` | 接收 Arrow，发布到 ROS2 |
| 多 Topic | `topics` | 单个 ROS2 节点上的多个 topic |
| Service 客户端 | `service` + `role: client` | 发送请求，接收响应 |
| Service 服务端 | `service` + `role: server` | 接收请求，发送响应 |
| Action 客户端 | `action` + `role: client` | 发送目标，接收反馈 + 结果 |
| Action 服务端 | `action` + `role: server` | 接收目标，发送反馈 + 结果 |
| QoS 策略 | `qos` | 可靠性、持久性、历史、活跃性 |
| 自动生成 | 自动 | 桥接二进制由 daemon 作为 Custom 节点生成 |

---

## 架构

当 Adora 描述符解析器遇到节点上的 `ros2:` 键时，将其转换为指向 `adora-ros2-bridge-node` 二进制的 `Custom` 节点。桥接配置序列化为 JSON 写入 `ADORA_ROS2_BRIDGE_CONFIG` 环境变量。

```
用户节点 <--(Arrow/SharedMem)--> 桥接二进制 <--(CDR/DDS)--> ROS2
```

桥接二进制：

1. 读取 `AMENT_PREFIX_PATH` 以定位已安装的 ROS2 消息包
2. 在启动时解析消息/service/action 定义
3. 创建 `ros2_client` 节点和相应的发布者、订阅者、客户端或服务器
4. 将传入的 ROS2 CDR 消息转换为 Arrow `StructArray`（subscribe/response/feedback）
5. 将传入的 Arrow `StructArray` 转换为 ROS2 CDR 消息（publish/request/goal）

你的用户节点永远不会链接到 ROS2 -- 所有 ROS2 通信都隔离在桥接二进制中。

---

## 前提条件

- **ROS2 环境已 source**：`AMENT_PREFIX_PATH` 必须设置并指向包含所需消息包的工作空间
- **消息包已安装**：如 `turtlesim`、`geometry_msgs`、`example_interfaces`
- **对于 service 客户端**：必须有 ROS2 service 服务器在运行（或使用配套服务器数据流）
- **对于 action 客户端**：必须在启动数据流*之前*有 ROS2 action 服务器在运行（没有 `wait_for_action_server` 机制）
- **对于 action 服务端**：ROS2 action 客户端向桥接发送目标（如 `ros2 action send_goal`）

---

## Topic 桥接

### 单 Topic（订阅）

订阅 ROS2 topic 并将消息作为 Arrow 数据转发到下游 Adora 节点。

```yaml
nodes:
  - id: pose_bridge
    ros2:
      topic: /turtle1/pose
      message_type: turtlesim/Pose
      direction: subscribe       # 默认，可省略
    outputs:
      - pose
```

桥接在 `/turtle1/pose` 上创建 ROS2 订阅，将每个传入的 `turtlesim/Pose` 消息反序列化为 Arrow `StructArray`，并在 `pose` 输出上发送。

### 单 Topic（发布）

从 Adora 节点接收 Arrow 数据并发布到 ROS2 topic。

```yaml
nodes:
  - id: cmd_bridge
    ros2:
      topic: /turtle1/cmd_vel
      message_type: geometry_msgs/Twist
      direction: publish
    inputs:
      cmd_vel: planner/cmd_vel
```

桥接在 `cmd_vel` 输入上接收 Arrow 数据，序列化为 `geometry_msgs/Twist` CDR，并发布到 `/turtle1/cmd_vel`。

### 多 Topic

在单个 ROS2 节点上下文上桥接多个 topic，混合订阅和发布方向。

```yaml
nodes:
  - id: turtle_bridge
    ros2:
      topics:
        - topic: /turtle1/pose
          message_type: turtlesim/Pose
          direction: subscribe
          output: pose
        - topic: /turtle1/cmd_vel
          message_type: geometry_msgs/Twist
          direction: publish
          input: velocity
      qos:
        reliable: true
        keep_last: 10
    inputs:
      velocity: planner/cmd_vel
    outputs:
      - pose
```

多 topic 模式每个桥接节点最多支持 64 个 topic。

### 输入/输出 ID 映射

默认情况下，topic 名称通过去除前导 `/` 并将剩余 `/` 替换为 `_` 转换为 Adora ID：

| ROS2 Topic | 默认 Adora ID |
|------------|---------------|
| `/turtle1/pose` | `turtle1_pose` |
| `/camera/image_raw` | `camera_image_raw` |

在多 topic 模式下，你可以用显式的 `output`（对于 subscribe）或 `input`（对于 publish）字段覆盖。在单 topic 模式下，直接使用节点声明的 `outputs` 或 `inputs`。

---

## Service 桥接

### Service 客户端

从 Adora 向外部 ROS2 service 发送请求并接收响应。

```yaml
nodes:
  - id: add_client
    ros2:
      service: /add_two_ints
      service_type: example_interfaces/AddTwoInts
      role: client
    inputs:
      request: requester/data
    outputs:
      - response
```

桥接等待 service 可用（最多 10 次重试，每次 2 秒），然后对于接收到的每个 Arrow 输入：

1. 将 Arrow 数据序列化为 `AddTwoInts_Request` CDR 消息
2. 向 ROS2 service 发送请求
3. 等待响应（30 秒超时）
4. 将响应反序列化为 Arrow 并在 `response` 输出上发送

### Service 服务端

将 Adora 处理器节点暴露为外部 ROS2 客户端可调用的 ROS2 service。

```yaml
nodes:
  - id: add_server
    ros2:
      service: /adora_add_two_ints
      service_type: example_interfaces/AddTwoInts
      role: server
    inputs:
      response: handler/result
    outputs:
      - request

  - id: handler
    path: path/to/handler-node
    inputs:
      request: add_server/request
    outputs:
      - result
```

桥接接收 ROS2 service 请求，为每个分配唯一的 `request_id`（UUID v7），将请求数据作为 Arrow 在 `request` 输出上转发（元数据中带 `request_id`），并等待处理器节点在 `response` 输入上发回带相同 `request_id` 的响应。然后响应被返回给正确的 ROS2 客户端。

参见 `examples/ros2-bridge/yaml-bridge-service/` 获取工作示例。

### 请求 ID 关联

每个传入的 ROS2 请求被分配一个 `request_id` 元数据参数。处理器节点**必须在发送响应时在元数据中包含相同的 `request_id`**。最简单的方式是传递 `metadata.parameters`：

```rust
Event::Input { id, metadata, data } => {
    // metadata.parameters 包含 request_id
    let result = compute(data);
    node.send_service_response("response".into(), metadata.parameters, result)?;
}
```

响应可以以任意顺序到达 -- 桥接通过 `request_id` 而非到达顺序进行关联。过期的待处理请求在 30 秒后被驱逐。最大待处理请求队列为 64 -- 队列满时额外请求被丢弃。

### Service 等待和超时

| 行为 | 值 |
|------|-----|
| Service 客户端：等待可用 | 10 次重试，每次 2 秒（总计 20 秒） |
| Service 客户端：响应超时 | 30 秒 |
| Service 服务端：待处理请求限制 | 64 |

---

## Action 桥接

### Action 客户端

从 Adora 向外部 ROS2 action 服务器发送目标，接收反馈和结果。

```yaml
nodes:
  - id: fib_client
    ros2:
      action: /fibonacci
      action_type: example_interfaces/Fibonacci
      role: client
    inputs:
      goal: goal_sender/goal
    outputs:
      - feedback
      - result
```

对于每个 Arrow 目标输入：

1. 将 Arrow 数据序列化为 `Fibonacci_Goal` CDR 消息
2. 向 action 服务器发送目标（30 秒超时）
3. 如果被接受，生成后台线程用于反馈和结果
4. 反馈消息在 `feedback` 输出上到达
5. 最终结果在 `result` 输出上到达（5 分钟超时）

### 反馈和结果流

action 桥接在不同输出上发送反馈和结果：

- **`feedback`**：在每个反馈消息从 action 服务器到达时流式传输。包含 action 的反馈消息作为 Arrow（如 Fibonacci 的 `{partial_sequence: int32[]}`）
- **`result`**：在 action 完成时发送一次。包含 action 的结果消息作为 Arrow（如 Fibonacci 的 `{sequence: int32[]}`）

### 并发目标

桥接支持最多 8 个并发进行中目标（`MAX_CONCURRENT_GOALS`）。额外目标会被丢弃并记录警告。每个目标生成专用的反馈和结果读取线程。

### 超时

| 行为 | 值 |
|------|-----|
| 目标发送超时 | 30 秒 |
| 结果获取超时 | 5 分钟 |
| 反馈 | 无超时（流式传输直到 action 完成） |

### Action 服务端

将 Adora 处理器节点暴露为外部 ROS2 客户端可调用的 ROS2 action 服务器。

```yaml
nodes:
  - id: fib_server
    ros2:
      action: /fibonacci
      action_type: example_interfaces/Fibonacci
      role: server
    inputs:
      feedback: handler/feedback
      result: handler/result
    outputs:
      - goal

  - id: handler
    path: path/to/handler-node
    inputs:
      goal: fib_server/goal
    outputs:
      - feedback
      - result
```

桥接从 ROS2 客户端接收目标，自动接受，并在 `goal` 输出上转发目标数据。处理器计算反馈和结果，并在 `feedback` 和 `result` 输入上发回。

参见 `examples/ros2-bridge/yaml-bridge-action-server/` 获取 Fibonacci 工作示例。

### 目标 ID 元数据

每个目标由作为 `goal_id` 元数据参数传递的 UUID 字符串标识。桥接在每个 `goal` 输出上设置 `goal_id`。处理器**必须在发送 `feedback` 和 `result` 时在元数据中包含相同的 `goal_id`**，以便桥接将它们关联到正确的目标。

最简单的方式是从目标事件传递 `metadata.parameters`：

```rust
Event::Input { id, metadata, data } => match id.as_str() {
    "goal" => {
        let params = metadata.parameters; // 包含 goal_id
        // ... 计算 ...
        node.send_output("feedback".into(), params.clone(), feedback)?;
        node.send_output("result".into(), params, result)?;
    }
    // ...
}
```

### Action 服务端生命周期

1. ROS2 客户端发送目标请求
2. 桥接自动接受目标并开始执行
3. 桥接在 `goal` 输出上发送目标数据，元数据中带 `goal_id`
4. 处理器发送 `feedback`（零次或多次），带相同 `goal_id`
5. 处理器发送 `result`（一次），带相同 `goal_id`；桥接将其返回给 ROS2 客户端
6. 如果客户端从不请求结果，结果发送在 5 分钟后超时

不包含数据或无法转发给处理器的目标会自动中止 -- 桥接向 ROS2 客户端发送 `Aborted` 状态，使其不会无限等待。

### 目标状态

默认情况下，结果以 `Succeeded` 状态返回。处理器可以通过在结果输出上设置 `goal_status` 元数据参数来覆盖：

| `goal_status` 值 | ROS2 状态 | 用例 |
|-------------------|-----------|------|
| `"succeeded"`（或省略） | `Succeeded` | 目标成功完成 |
| `"aborted"` | `Aborted` | 目标在执行期间失败 |
| `"canceled"` | `Canceled` | 目标被处理器取消 |

无法识别的 `goal_status` 值默认为 `Aborted` 并记录警告。完全省略 `goal_status` 默认为 `Succeeded`。

Rust 示例：
```rust
use adora_node_api::{GOAL_STATUS, GOAL_STATUS_ABORTED, Parameter};

let mut params = metadata.parameters; // 包含 goal_id
params.insert(GOAL_STATUS.to_string(), Parameter::String(GOAL_STATUS_ABORTED.to_string()));
node.send_output("result".into(), params, error_result)?;
```

### Action 服务端限制

| 行为 | 值 |
|------|-----|
| 最大并发目标 | 8（额外目标收到 `Aborted` 状态） |
| 自动接受 | 所有目标自动接受 |
| 结果发送超时 | 5 分钟 |

### Python Action 服务端处理器

Python 节点接收目标数据作为 PyArrow 数组，元数据字典中带 `goal_id`。在反馈/结果输出上传递：

```python
for event in node:
    if event["type"] == "INPUT" and event["id"] == "goal":
        goal_id = event["metadata"]["goal_id"]
        order = event["value"]["order"][0].as_py()

        # 发送反馈
        node.send_output("feedback", feedback_array, {"goal_id": goal_id})

        # 发送结果（带可选状态）
        node.send_output("result", result_array, {
            "goal_id": goal_id,
            "goal_status": "succeeded",  # 或 "aborted"、"canceled"
        })
```

### C++ Action 服务端处理器

C++ 节点通过类型安全的元数据访问器访问 `goal_id`：

```cpp
auto goal_id = metadata->get_str("goal_id");

// 带 goal_id 发送反馈
auto fb_metadata = new_metadata();
fb_metadata->set_string("goal_id", goal_id);
send_arrow_output_with_metadata("feedback", feedback_data, fb_metadata);

// 带 goal_id 发送结果
auto res_metadata = new_metadata();
res_metadata->set_string("goal_id", goal_id);
send_arrow_output_with_metadata("result", result_data, res_metadata);
```

---

## 服务质量（QoS）

### 配置

在桥接级别设置 QoS（适用于所有 topic/通道）或在多 topic 模式下按每个 topic 设置。

```yaml
nodes:
  - id: my_bridge
    ros2:
      topic: /sensor/data
      message_type: sensor_msgs/LaserScan
      qos:
        reliable: true
        durability: transient_local
        keep_last: 10
        liveliness: automatic
        lease_duration: 5.0
        max_blocking_time: 0.5
```

### 默认值

| 字段 | 默认值 |
|------|--------|
| `reliable` | `false`（尽力传输） |
| `durability` | `volatile` |
| `liveliness` | `automatic` |
| `lease_duration` | 无穷大 |
| `max_blocking_time` | 100ms（仅当 `reliable: true` 时适用） |
| `keep_last` | `1` |
| `keep_all` | `false` |

### 每 Topic QoS 覆盖

在多 topic 模式下，每个 topic 可以覆盖桥接级别的 QoS：

```yaml
ros2:
  topics:
    - topic: /fast_sensor
      message_type: sensor_msgs/Imu
      direction: subscribe
      qos:
        reliable: false          # 覆盖：此 topic 使用尽力传输
        keep_last: 1
    - topic: /cmd
      message_type: geometry_msgs/Twist
      direction: publish
      # 继承桥接级别 QoS（reliable: true）
  qos:
    reliable: true               # 所有 topic 的默认值
    keep_last: 10
```

### 验证规则

| 字段 | 有效值 |
|------|--------|
| `reliable` | `true`、`false` |
| `durability` | `"volatile"`、`"transient_local"` |
| `liveliness` | `"automatic"`、`"manual_by_participant"`、`"manual_by_topic"` |
| `keep_last` | `1` 到 `10000` |
| `keep_all` | `true`、`false`（与 `keep_last` 互斥） |
| `lease_duration` | 有限非负浮点数（秒） |
| `max_blocking_time` | 有限非负浮点数（秒） |

---

## 数据格式：Arrow 结构体

你的节点与桥接之间交换的所有数据使用单行的 Arrow `StructArray`。ROS2 消息中的每个字段成为结构体中的一列。

### 如何构建 Arrow 消息

Rust 示例：构建 `AddTwoInts_Request`（`{a: i64, b: i64}`）：

```rust
use std::sync::Arc;
use arrow::array::{Array, Int64Array, StructArray};
use arrow::datatypes::{DataType, Field};

fn make_add_request(a: i64, b: i64) -> StructArray {
    let fields = vec![
        Arc::new(Field::new("a", DataType::Int64, false)),
        Arc::new(Field::new("b", DataType::Int64, false)),
    ];
    let arrays: Vec<Arc<dyn Array>> = vec![
        Arc::new(Int64Array::from(vec![a])),
        Arc::new(Int64Array::from(vec![b])),
    ];
    StructArray::try_new(fields.into(), arrays, None)
        .expect("failed to create struct array")
}
```

读取响应（`{sum: i64}`）：

```rust
use arrow::array::{Int64Array, StructArray};

fn read_response(data: &dyn arrow::array::Array) -> i64 {
    let struct_array = data
        .as_any()
        .downcast_ref::<StructArray>()
        .expect("expected struct array");
    struct_array
        .column_by_name("sum")
        .expect("missing 'sum' field")
        .as_any()
        .downcast_ref::<Int64Array>()
        .expect("expected Int64Array")
        .value(0)
}
```

### ROS2 类型到 Arrow 类型映射

| ROS2 类型 | Arrow 类型 | Rust Arrow 数组 |
|-----------|-----------|-----------------|
| `bool` | `Boolean` | `BooleanArray` |
| `int8` | `Int8` | `Int8Array` |
| `int16` | `Int16` | `Int16Array` |
| `int32` | `Int32` | `Int32Array` |
| `int64` | `Int64` | `Int64Array` |
| `uint8` / `byte` / `char` | `UInt8` | `UInt8Array` |
| `uint16` | `UInt16` | `UInt16Array` |
| `uint32` | `UInt32` | `UInt32Array` |
| `uint64` | `UInt64` | `UInt64Array` |
| `float32` | `Float32` | `Float32Array` |
| `float64` | `Float64` | `Float64Array` |
| `string` | `Utf8` | `StringArray` |
| `wstring` | `Utf8`（CDR 端编码为 UTF-16） | `StringArray` |
| 嵌套消息 | `Struct` | `StructArray` |

### 序列和数组

| ROS2 类型 | Arrow 类型 | Rust Arrow 数组 |
|-----------|-----------|-----------------|
| 变长序列（`int32[]`） | `List` | `ListArray` |
| 有界序列（`int32[<=10]`） | `List`（长度验证） | `ListArray` |
| 固定大小数组（`int32[3]`） | `FixedSizeList` | `FixedSizeListArray` |

示例：从 Fibonacci 反馈读取 `ListArray`（`{partial_sequence: int32[]}`）：

```rust
use arrow::array::{Int32Array, ListArray, StructArray};

let struct_array = data.as_any().downcast_ref::<StructArray>().unwrap();
let list = struct_array
    .column_by_name("partial_sequence")
    .unwrap()
    .as_any()
    .downcast_ref::<ListArray>()
    .unwrap();
let values = list
    .value(0)
    .as_any()
    .downcast_ref::<Int32Array>()
    .unwrap()
    .values()
    .to_vec();
```

---

## 完整 YAML 参考

```yaml
nodes:
  - id: my_bridge
    ros2:
      # --- 模式（必须选择其一）---

      # 单 topic 模式
      topic: /topic_name               # ROS2 topic 名称
      message_type: package/TypeName    # ROS2 消息类型
      direction: subscribe             # subscribe（默认）| publish

      # 多 topic 模式（与 topic 互斥）
      topics:
        - topic: /topic_a
          message_type: package/TypeA
          direction: subscribe
          output: custom_output_id     # 覆盖默认 ID 映射
          qos:                         # 每 topic QoS 覆盖
            reliable: true
        - topic: /topic_b
          message_type: package/TypeB
          direction: publish
          input: custom_input_id       # 覆盖默认 ID 映射

      # Service 模式（与 topic/topics/action 互斥）
      service: /service_name           # ROS2 service 名称
      service_type: package/TypeName   # ROS2 service 类型
      role: client                     # client | server

      # Action 模式（与 topic/topics/service 互斥）
      action: /action_name             # ROS2 action 名称
      action_type: package/TypeName    # ROS2 action 类型
      role: client                     # client | server

      # --- QoS（可选，适用于所有通道）---
      qos:
        reliable: false                # true | false（默认：false = 尽力传输）
        durability: volatile           # volatile（默认）| transient_local
        liveliness: automatic          # automatic | manual_by_participant | manual_by_topic
        lease_duration: 5.0            # 秒（默认：无穷大）
        max_blocking_time: 0.1         # 秒（默认：0.1，仅 reliable）
        keep_last: 1                   # 1-10000（默认：1）
        keep_all: false                # true | false（默认：false）

      # --- 可选 ROS2 节点配置 ---
      namespace: /                     # ROS2 命名空间（默认："/"）
      node_name: my_ros_node           # ROS2 节点名称（默认：adora 节点 ID）

    # --- 标准 Adora 节点字段 ---
    inputs:
      input_id: source_node/output_id
    outputs:
      - output_id
```

---

## 使用场景

### 1. 订阅传感器数据（turtlesim 位姿）

```yaml
nodes:
  - id: pose_bridge
    ros2:
      topic: /turtle1/pose
      message_type: turtlesim/Pose
    outputs:
      - pose

  - id: my_processor
    path: ./target/debug/my-processor
    inputs:
      pose: pose_bridge/pose
```

```rust
// 在 my_processor 中：以 Arrow 形式接收 turtlesim/Pose
Event::Input { id, data, .. } if id.as_str() == "pose" => {
    let s = data.as_any().downcast_ref::<StructArray>().unwrap();
    let x = s.column_by_name("x").unwrap()
        .as_any().downcast_ref::<Float32Array>().unwrap().value(0);
    let y = s.column_by_name("y").unwrap()
        .as_any().downcast_ref::<Float32Array>().unwrap().value(0);
    println!("Turtle at ({x}, {y})");
}
```

### 2. 发布速度命令

```yaml
nodes:
  - id: planner
    path: ./target/debug/planner
    inputs:
      tick: adora/timer/millis/100
    outputs:
      - cmd_vel

  - id: cmd_bridge
    ros2:
      topic: /turtle1/cmd_vel
      message_type: geometry_msgs/Twist
      direction: publish
    inputs:
      cmd_vel: planner/cmd_vel
```

### 3. 多 Topic 双向桥接

在单个 ROS2 节点上订阅位姿并发布速度。

```yaml
nodes:
  - id: turtle_bridge
    ros2:
      topics:
        - topic: /turtle1/pose
          message_type: turtlesim/Pose
          direction: subscribe
          output: pose
        - topic: /turtle1/cmd_vel
          message_type: geometry_msgs/Twist
          direction: publish
          input: velocity
      qos:
        reliable: true
        keep_last: 10
    inputs:
      velocity: planner/cmd_vel
    outputs:
      - pose

  - id: planner
    path: ./target/debug/planner
    inputs:
      pose: turtle_bridge/pose
      tick: adora/timer/millis/100
    outputs:
      - cmd_vel
```

### 4. Service 客户端：调用外部 ROS2 Service

```yaml
nodes:
  - id: requester
    path: ./target/debug/requester
    inputs:
      tick: adora/timer/millis/1000
      response: add_client/response
    outputs:
      - request

  - id: add_client
    ros2:
      service: /add_two_ints
      service_type: example_interfaces/AddTwoInts
      role: client
    inputs:
      request: requester/request
    outputs:
      - response
```

前提条件：先运行 ROS2 service：
```bash
ros2 run examples_rclcpp_minimal_service service_main
```

### 5. Service 服务端：将 Adora 处理器暴露为 ROS2 Service

```yaml
nodes:
  - id: add_server
    ros2:
      service: /add_two_ints
      service_type: example_interfaces/AddTwoInts
      role: server
    inputs:
      response: handler/response
    outputs:
      - request

  - id: handler
    path: ./target/debug/handler
    inputs:
      request: add_server/request
    outputs:
      - response
```

处理器接收 `{a: i64, b: i64}` 作为 Arrow，计算结果，并发回 `{sum: i64}`。外部 ROS2 客户端可以调用此 service：
```bash
ros2 service call /add_two_ints example_interfaces/srv/AddTwoInts "{a: 3, b: 5}"
```

### 6. Action 客户端：长期运行的 Fibonacci 目标

```yaml
nodes:
  - id: goal_sender
    path: ./target/debug/goal-sender
    inputs:
      tick: adora/timer/millis/5000
      feedback: fib_client/feedback
      result: fib_client/result
    outputs:
      - goal

  - id: fib_client
    ros2:
      action: /fibonacci
      action_type: example_interfaces/Fibonacci
      role: client
    inputs:
      goal: goal_sender/goal
    outputs:
      - feedback
      - result
```

前提条件：在数据流**之前**启动 action 服务器：
```bash
ros2 run examples_rclcpp_action_server fibonacci_action_server
```

### 7. Action 服务端：将 Adora 处理器暴露为 ROS2 Action

```yaml
nodes:
  - id: fib_server
    ros2:
      action: /fibonacci
      action_type: example_interfaces/Fibonacci
      role: server
    inputs:
      feedback: handler/feedback
      result: handler/result
    outputs:
      - goal

  - id: handler
    path: ./target/debug/handler
    inputs:
      goal: fib_server/goal
    outputs:
      - feedback
      - result
```

处理器接收带元数据中 `goal_id` 的 `{order: int32}` 目标，发送 `{partial_sequence: int32[]}` 反馈和最终 `{sequence: int32[]}` 结果 -- 所有都带元数据中相同的 `goal_id`。外部 ROS2 客户端可以发送目标：
```bash
ros2 action send_goal /fibonacci example_interfaces/action/Fibonacci "{order: 10}"
```

---

## 限制和已知约束

- **Action 服务端自动接受**：所有传入目标自动接受。处理器无法在执行开始前拒绝目标。
- **不支持 action cancel**：客户端和服务端都不处理 ROS2 cancel 请求。
- **没有 `wait_for_action_server`**：`ros2_client` 库不提供此 API。在数据流之前启动 action 服务器。如果服务器不可用，第一个目标将超时（30 秒）。
- **单次 service 客户端**：service 客户端顺序处理请求 -- 每个请求阻塞直到响应到达（或 30 秒超时）。
- **QoS 对 service/action 通道统一**：`qos` 配置适用于所有 service/action 子通道（goal、result、cancel、feedback、status）。不可配置每通道 QoS。
- **需要 `AMENT_PREFIX_PATH`**：如果找不到 ROS2 消息定义，桥接在启动时失败。
- **最多 64 个 topic**：多 topic 模式每个桥接节点最多支持 64 个 topic。
- **最多 8 个并发 action 目标**：达到限制时额外目标收到 `Aborted` 状态。
- **最多 64 个待处理 service 请求（服务端）**：队列满时请求被丢弃。

---

## 最佳实践

**运行前 source ROS2 环境。** 确保 `AMENT_PREFIX_PATH` 已设置并包含所有需要的消息包。如果找不到定义，桥接会记录错误。

**在数据流之前启动 action 服务器。** 没有 action 服务器的等待机制。如果服务器未就绪，第一次目标发送将在 30 秒后超时。

**对相关 topic 使用多 topic 模式。** 在同一桥接节点上桥接 `/turtle1/pose`（subscribe）和 `/turtle1/cmd_vel`（publish），比使用两个独立桥接节点减少资源使用。

**精确匹配 Arrow 字段名。** 桥接验证 Arrow 结构体字段名与 ROS2 消息定义匹配。缺失字段使用默认值（数字为零，空字符串）。多余字段导致错误。

**在多 topic 模式下使用显式 `output`/`input`。** 默认 ID 映射（去除 `/`，用 `_` 替换 `/`）对深层 topic 名称可能令人困惑。显式 ID 使数据流 YAML 自文档化。

**设置 QoS 以匹配 ROS2 发布者/订阅者。** QoS 不匹配（如可靠订阅者配尽力发布者）导致静默通信失败。使用 `ros2 topic info -v /topic_name` 查看现有 QoS 设置。

**在 service 响应中传递 `request_id`。** 桥接使用 `request_id` 元数据参数将响应关联到请求。如果处理器不在响应元数据中包含 `request_id`，桥接无法将响应与原始 ROS2 请求匹配。
