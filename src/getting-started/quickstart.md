> **[English](../../getting-started/quickstart.md)**

# Python 快速入门

本指南将引导您为 adora 数据流编写 Python 节点和算子。

## 前提条件

```bash
cargo install adora-cli    # CLI（adora 命令）
pip install adora-rs       # Python 节点/算子 API
```

`adora-rs` 包将 `pyarrow` 作为依赖项。

**从源码构建**（替代 `pip install adora-rs`）：

```bash
pip install maturin  # 需要 >= 1.8
cd apis/python/node && maturin develop --uv && cd ../../..
```

## Hello World：发送者和接收者

创建三个文件：

**sender.py** -- 发送 100 条编号消息：

```python
import pyarrow as pa
from adora import Node

node = Node()
for i in range(100):
    node.send_output("message", pa.array([i]))
```

**receiver.py** -- 接收并打印消息：

```python
from adora import Node

node = Node()
for event in node:
    if event["type"] == "INPUT":
        values = event["value"].to_pylist()
        print(f"Received {event['id']}: {values}")
    elif event["type"] == "STOP":
        break
```

**dataflow.yml** -- 将发送者连接到接收者：

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

运行：

```bash
adora run dataflow.yml
```

## 事件

每次调用 `node.next()` 或迭代 `for event in node` 返回一个事件字典：

| 键 | 类型 | 描述 |
|----|------|------|
| `type` | str | `"INPUT"`、`"INPUT_CLOSED"`、`"STOP"` 或 `"ERROR"` |
| `id` | str | 输入名称（例如 `"message"`）-- 仅适用于 `INPUT` 事件 |
| `value` | pyarrow.Array 或 None | 数据有效载荷 |
| `metadata` | dict | 追踪/路由元数据 |

按 `event["type"]` 处理事件：

```python
for event in node:
    match event["type"]:
        case "INPUT":
            process(event["id"], event["value"])
        case "INPUT_CLOSED":
            print(f"Input {event['id']} closed")
        case "STOP":
            break
```

## 使用 Arrow 数据

所有数据以 Apache Arrow 数组的形式在 adora 中传输。常见模式：

```python
import pyarrow as pa

# 简单值
node.send_output("count", pa.array([42]))
node.send_output("names", pa.array(["alice", "bob"]))

# 读取值
values = event["value"].to_pylist()  # [42] 或 ["alice", "bob"]

# 结构化数据
struct = pa.StructArray.from_arrays(
    [pa.array([1.5]), pa.array(["hello"])],
    names=["x", "y"],
)
node.send_output("point", struct)

# 原始字节（图像、序列化数据等）
node.send_output("frame", pa.array(raw_bytes))
```

## 算子

算子是节点的轻量级替代方案。它们在 adora 运行时进程内运行（无独立操作系统进程），因此对于简单转换更加高效。

定义一个包含 `on_event` 方法的 `Operator` 类：

```python
# doubler_op.py
import pyarrow as pa
from adora import AdoraStatus

class Operator:
    def on_event(self, event, send_output) -> AdoraStatus:
        if event["type"] == "INPUT":
            value = event["value"].to_pylist()[0]
            send_output("doubled", pa.array([value * 2]), event["metadata"])
        return AdoraStatus.CONTINUE
```

在 YAML 中使用 `operator` 而非 `path` 来引用：

```yaml
nodes:
  - id: timer
    path: adora/timer/millis/500
    outputs:
      - tick

  - id: doubler
    operator:
      python: doubler_op.py
      inputs:
        tick: timer/tick
      outputs:
        - doubled
```

**何时使用算子 vs 节点：**

| | 节点 | 算子 |
|--|------|------|
| 进程模型 | 独立操作系统进程 | 进程内（共享运行时） |
| 启动成本 | 较高 | 较低 |
| 隔离性 | 完全进程隔离 | 共享内存空间 |
| 适用场景 | 长时间运行、重计算 | 轻量级转换、过滤 |

## 异步节点

对于需要异步 I/O（HTTP 调用、数据库查询等）的节点，使用 `recv_async()`：

```python
import asyncio
from adora import Node

async def main():
    node = Node()
    for _ in range(50):
        event = await node.recv_async()
        if event["type"] == "STOP":
            break
        # 在此执行异步操作
        result = await fetch_data(event["value"])
        node.send_output("result", result)

asyncio.run(main())
```

## 日志

使用 `node.log()` 进行结构化日志记录，可与 `adora logs` 集成：

```python
node.log("info", "Processing item", {"count": str(i)})
```

或使用 Python 标准 `logging` 模块 -- adora 会自动捕获 stdout/stderr：

```python
import logging
logging.info("Processing item %d", i)
```

## 定时器

内置定时器节点可生成周期性 tick，无需编写任何代码：

```yaml
nodes:
  - id: tick-source
    path: adora/timer/millis/100    # 每 100ms tick 一次
    outputs:
      - tick

  - id: my-node
    path: my_node.py
    inputs:
      tick: tick-source/tick
```

也支持：`adora/timer/hz/30` 表示 30 Hz。

## 下一步

- [Python API 参考](../languages/python.md) -- Node、Operator、DataflowBuilder、CUDA 的完整 API 文档
- [通信模式](../concepts/patterns.md) -- service（请求/应答）和 action（目标/反馈/结果）模式
- [分布式部署](../operations/distributed.md) -- 使用 `adora up` 跨多台机器运行
