> **[English](../../languages/python.md)**

# Python API 参考

本文档涵盖用于构建 adora 节点、算子和数据流的 Python API。安装方式：

```bash
pip install adora-rs
```

---

## Node API

```python
from adora import Node
```

`Node` 类是自定义节点的主要接口。它连接到运行中的数据流，接收输入事件，并发送输出。

### Node 类

#### `__init__(node_id=None)`

创建新节点并连接到运行中的数据流。

```python
# 标准用法：节点 ID 从 daemon 设置的环境变量中读取
node = Node()

# 动态用法：通过显式节点 ID 连接到运行中的数据流
node = Node(node_id="my-dynamic-node")
```

#### `next(timeout=None)`

从事件流中检索下一个事件。阻塞直到事件可用或超时。

#### `send_output(output_id, data, metadata=None)`

在输出通道上发送数据。

```python
import pyarrow as pa

# 发送原始字节
node.send_output("status", b"OK")

# 发送 Apache Arrow 数组（支持零拷贝）
node.send_output("values", pa.array([1, 2, 3]))

# 带元数据发送
node.send_output("image", pa.array(pixels), {"camera_id": "front"})
```

#### 日志

Python 节点可以使用 Python 内置的 `logging` 模块（推荐）或显式节点 API 进行日志记录。

**Python `logging` 模块（自动桥接）：**

创建 `Node()` 时，它会自动安装一个处理器，将 Python 的 `logging` 模块路由到 adora daemon。无需额外配置：

```python
import logging
from adora import Node

node = Node()  # 安装日志桥接

logging.info("传感器已初始化")
logging.warning("温度过高")
```

#### 迭代支持

`Node` 类实现了 `__iter__` 和 `__next__`，因此可以直接迭代：

```python
for event in node:
    match event["type"]:
        case "INPUT":
            process(event["value"])
        case "STOP":
            break
```

---

### 事件字典

事件以普通 Python 字典的形式返回。

#### INPUT

```python
{
    "type": "INPUT",
    "id": "camera_image",
    "kind": "adora",
    "value": <pyarrow.Array>,
    "metadata": { "timestamp": datetime, ... },
}
```

#### STOP

```python
{
    "type": "STOP",
    "id": "MANUAL" | "ALL_INPUTS_CLOSED",
    "kind": "adora",
}
```

---

## 算子 API

算子在 adora 运行时进程内运行（无独立操作系统进程）。定义一个名为 `Operator` 的 Python 类并包含 `on_event` 方法。

```python
from adora import AdoraStatus

class Operator:
    def __init__(self):
        self.count = 0

    def on_event(self, adora_event, send_output) -> AdoraStatus:
        if adora_event["type"] == "INPUT":
            self.count += 1
            send_output("result", b"processed", adora_event["metadata"])
        return AdoraStatus.CONTINUE
```

---

## DataflowBuilder

```python
from adora.builder import DataflowBuilder, Node, Operator, Output
```

以编程方式在 Python 中构建数据流 YAML。

```python
flow = DataflowBuilder("my-robot")

sender = flow.add_node("sender")
sender.path("sender.py")
tick_output = sender.add_output("message")

receiver = flow.add_node("receiver")
receiver.path("receiver.py")
receiver.add_input("message", tick_output)

flow.to_yaml("dataflow.yml")
```

---

## CUDA 模块

```python
from adora.cuda import torch_to_ipc_buffer, ipc_buffer_to_ipc_handle, open_ipc_handle
```

用于通过 CUDA IPC 在节点之间零拷贝共享 GPU 张量的工具。需要带 CUDA 的 PyTorch 和带 CUDA 支持的 Numba。

```python
import torch
from adora import Node
from adora.cuda import torch_to_ipc_buffer

node = Node()
tensor = torch.randn(1024, 768, device="cuda")
ipc_buffer, metadata = torch_to_ipc_buffer(tensor)
node.send_output("gpu_data", ipc_buffer, metadata)
```
