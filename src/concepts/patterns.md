> **[English](../../concepts/patterns.md)**

# 通信模式

Adora 是一个基于发布/订阅消息传递的数据流框架。在基本的 topic 之上，框架支持 **service**（请求/应答）、**action**（目标/反馈/结果）和 **streaming**（会话/片段/块）模式，使用约定的元数据键实现。无需更改 daemon、coordinator 或 YAML 语法 -- 这些模式作为节点 API 级别的约定实现。

## 1. Topic（发布/订阅）

默认模式。节点在输出上发布数据，任何订阅该输出的节点都会收到它。

```yaml
nodes:
  - id: publisher
    outputs:
      - data
  - id: subscriber
    inputs:
      data: publisher/data
```

**适用场景**：流式传感器数据、周期性状态、即发即忘事件。

## 2. Service（请求/应答）

客户端发送请求并期望恰好一个响应，通过 `request_id` 元数据键关联。

### 约定的元数据键

| 键 | 描述 |
|----|------|
| `request_id` | 关联请求和响应的 UUID v7 |

### YAML

```yaml
nodes:
  - id: client
    inputs:
      tick: adora/timer/millis/500
      response: server/response
    outputs:
      - request

  - id: server
    inputs:
      request: client/request
    outputs:
      - response
```

### 节点 API 辅助函数

```rust
// 客户端：发送带有自动生成 request_id 的请求
let rid = node.send_service_request("request".into(), params, data)?;

// 服务端：传递 metadata.parameters（包含 request_id）
node.send_service_response("response".into(), metadata.parameters, result)?;
```

服务端**必须**将传入请求的元数据参数中的 `request_id` 传递到响应中。

## 3. Action（目标/反馈/结果）

客户端发送目标并接收周期性反馈加最终结果。Action 支持取消。

### 约定的元数据键

| 键 | 描述 |
|----|------|
| `goal_id` | 标识目标的 UUID v7 |
| `goal_status` | 目标的最终状态 |

目标状态值：

| 值 | 含义 |
|----|------|
| `succeeded` | 目标成功完成 |
| `aborted` | 目标被服务端中止 |
| `canceled` | 目标被客户端取消 |

### 取消模式

客户端在 `cancel` 输出上发送带有 `goal_id` 的消息。服务端在处理步骤之间检查取消请求，并发送带有 `goal_status = "canceled"` 的结果。

## 4. Streaming（会话/片段/块）

用于实时管道（语音、视频、传感器流），用户可以在流中途中断并丢弃排队数据。

### 约定的元数据键

| 键 | 类型 | 描述 |
|----|------|------|
| `session_id` | String | 标识对话/会话 |
| `segment_id` | Integer | 会话内的逻辑单元（例如一个语句） |
| `seq` | Integer | 片段内的块序列号 |
| `fin` | Bool | 片段的最后一个块为 `true` |
| `flush` | Bool | 为 `true` 时丢弃此输入上较旧的排队消息 |

### 队列 flush 行为

当消息的元数据中包含 `flush: true` 时，接收者的输入队列在 flush 消息传递之前会清除所有较旧的消息。这使得语音管道中的即时中断成为可能 -- 当用户在 TTS 输出上讲话时，ASR 节点发送带有 `flush: true` 的新片段，TTS 节点立即丢弃上一个响应中排队的所有音频块。

## 5. 选择模式

| 需要响应？ | 长时间运行？ | 可取消？ | 实时流？ | 模式 |
|:-:|:-:|:-:|:-:|---------|
| 否 | - | - | 否 | **Topic** |
| 是 | 否 | 否 | 否 | **Service** |
| 是 | 是 | 可选 | 否 | **Action** |
| 否 | 是 | 通过 flush | 是 | **Streaming** |

## 6. 重要细节

- **`goal_status` 匹配区分大小写。** 始终使用精确的小写值：`"succeeded"`、`"aborted"`、`"canceled"`。

## 7. Python 兼容性

Python 节点使用相同的元数据约定。参数是带有字符串键的普通字典：

```python
import uuid

# Service 客户端
params = {"request_id": str(uuid.uuid7())}
node.send_output("request", data, metadata={"parameters": params})

# Service 服务端 -- 传递参数
node.send_output("response", result, metadata=event["metadata"])
```

> **注意**：`uuid.uuid7()` 需要 Python 3.13+。在旧版本上使用 `uuid_utils` 包或 `uuid.uuid4()`。
