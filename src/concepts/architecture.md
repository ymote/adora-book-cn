> **[English](../../concepts/architecture.md)**

# Adora 架构

Adora（AI-Dora，Agentic Dataflow-Oriented Robotic Architecture）的完整架构参考 -- 一个 100% Rust 编写的实时机器人与 AI 应用框架。

## 概述与设计理念

Adora 建立在四个核心原则之上：

1. **面向数据流**：应用是由节点组成的有向图，节点之间通过类型化数据通道连接。节点声明输入和输出；框架负责路由、调度和生命周期管理。
2. **零拷贝性能**：超过 4 KiB 的消息使用 128 字节对齐缓冲区的共享内存和原子协调，实现比 ROS2 低 10-17 倍的延迟。
3. **多语言支持**：对 Rust、Python（PyO3）、C 和 C++ 节点提供一等支持 -- 全部共享相同的 Apache Arrow 数据格式。
4. **四层架构**：消息协议、核心库、daemon/运行时执行层、CLI/coordinator 编排层。

### 架构栈

```
┌─────────────────────────────────────────────────┐
│  CLI (adora)          Coordinator (orchestrator) │  第 4 层：编排
├─────────────────────────────────────────────────┤
│  Daemon (每台机器)      Runtime (算子)            │  第 3 层：执行
├─────────────────────────────────────────────────┤
│  adora-core    shared-memory-server    Node API  │  第 2 层：核心库
├─────────────────────────────────────────────────┤
│  adora-message (协议 + Arrow 类型)               │  第 1 层：协议
└─────────────────────────────────────────────────┘
```

## 工作空间结构

**Rust edition 2024，MSRV 1.85.0，工作空间版本 0.1.0。**
所有 crate 共享工作空间版本。

### 可执行文件（7 个）

| 路径 | Crate | 角色 |
|------|-------|------|
| `binaries/cli` | adora-cli | CLI 二进制文件（`adora` 命令）-- 构建、运行、停止数据流 |
| `binaries/coordinator` | adora-coordinator | 编排分布式多 daemon 部署；WebSocket 服务器 |
| `binaries/daemon` | adora-daemon | 生成节点，管理每台机器的共享内存/TCP 通信 |
| `binaries/runtime` | adora-runtime | 进程内算子执行（通过 dlopen/PyO3 支持 Python/C/C++） |
| `binaries/ros2-bridge-node` | adora-ros2-bridge-node | ROS2 集成节点 |
| `binaries/record-node` | adora-record-node | 将数据流消息录制为 `.adorec` 格式 |
| `binaries/replay-node` | adora-replay-node | 从 `.adorec` 文件回放录制的消息 |

### 核心库（6 个）

| 路径 | Crate | 角色 |
|------|-------|------|
| `libraries/message` | adora-message | 所有组件间消息类型、协议定义、Arrow 元数据 |
| `libraries/core` | adora-core | 数据流描述符解析、构建工具、Zenoh 配置 |
| `libraries/shared-memory-server` | shared-memory-server | 大于等于 4 KiB 消息的零拷贝 IPC |
| `libraries/recording` | adora-recording | 录制格式（.adorec）：bincode 头部 + 条目 + 尾部 |
| `libraries/arrow-convert` | adora-arrow-convert | Arrow 类型转换（数值、日期时间） |
| `libraries/coordinator-store` | adora-coordinator-store | coordinator 状态持久化（内存或 redb 后端） |

### API Crate（9 个）

| 路径 | Crate | 语言 |
|------|-------|------|
| `apis/rust/node` | adora-node-api | Rust |
| `apis/rust/operator` | adora-operator-api | Rust |
| `apis/python/node` | adora-node-api-python | Python（PyO3）-- 构建 `adora` 模块 |
| `apis/c/node` | adora-node-api-c | C |
| `apis/c/operator` | adora-operator-api-c | C/C++ |

## 组件架构

### CLI

`adora` 命令提供三组命令：

**生命周期**（run、up、down、build、start、stop、restart）：
- `adora run` 在本地执行数据流，无需 coordinator/daemon（单机快捷方式）
- `adora up` / `adora down` 管理 coordinator + daemon 基础设施
- `adora start` / `adora stop` 控制运行中的 coordinator 上的数据流

**监控**（list、logs、inspect、topic、node、record、replay、trace）：
- 使用 `adora inspect top` 进行实时检查
- 主题订阅和数据检查
- 通过 `.adorec` 文件进行录制和回放

**设置**（status、new、graph、system、completion、self）：
- 项目脚手架、数据流可视化、自更新

### Coordinator

coordinator 是一个基于 Axum 的 WebSocket 服务器，用于编排分布式部署。

**WebSocket 路由：**
- `/api/control` -- CLI 控制平面（构建、启动、停止、列表、日志、主题订阅）
- `/api/daemon` -- daemon 注册和事件流
- `/api/artifacts/{build_id}/{node_id}` -- 二进制产物下载
- `/health` -- 健康检查端点

**状态管理：** 默认使用内存存储，可选通过 `redb` 后端进行持久化存储。

### Daemon

daemon 在每台机器上运行一个，管理该机器上所有节点的生命周期。

**事件循环**（`Daemon::run_inner()`）：异步 Tokio 事件循环，合并：
- coordinator 命令（WebSocket）
- 节点事件（TCP/共享内存）
- daemon 间事件（Zenoh）
- 心跳（5 秒间隔）、指标收集（2 秒）、健康检查（默认 5 秒）

**节点生成：**
1. 为节点创建工作目录
2. 建立通信通道（TCP、共享内存或 Unix 域套接字）
3. 将 `NodeConfig` 序列化到环境变量
4. 以净化的环境生成进程（阻止 `LD_PRELOAD`、`DYLD_INSERT_LIBRARIES` 等）
5. 通过 `ProcessHandle` 进行监控

### 节点

节点是与 daemon 通信的独立进程。

**生命周期：**
1. 节点启动，从环境读取 `NodeConfig`
2. 通过 `DaemonRequest::Register` 向 daemon 注册
3. 通过 `DaemonRequest::Subscribe` 订阅事件
4. 在循环中处理事件（`NextEvent` -> 处理 -> `SendMessage`）
5. 报告 drop token 用于共享内存清理
6. 通过 `OutputsDone` 发送完成信号

## 通信协议

### CLI 到 Coordinator（WebSocket）

| 属性 | 值 |
|------|-----|
| 传输 | TCP 上的 WebSocket |
| 默认端口 | 6013 |
| 认证 | `Authorization` 头部中的 Bearer token |
| 控制消息 | JSON 文本帧（请求/响应/事件） |
| 主题数据 | 二进制帧：`[16 字节 UUID][bincode 有效载荷]` |
| 速率限制 | 每 IP 每 60 秒 20 个连接 |
| 最大连接数 | 256 |

### Daemon 到节点（本地）

三种传输选项，通过 `LocalCommunicationConfig` 配置：

**TCP**（默认）：
- 绑定 `127.0.0.1:0`（临时端口），启用 `TCP_NODELAY`
- 帧格式：`[8 字节 u64 LE 长度][bincode 有效载荷]`
- 最大消息：64 MiB，读取超时：30 秒

**共享内存**（零拷贝）：
- 每个节点四个 4 KiB 区域：控制、事件、drop token、事件关闭
- 用于大于等于 4096 字节（`ZERO_COPY_THRESHOLD`）的消息
- 使用 acquire/release 顺序的原子同步

**Unix 域套接字**（仅 Unix）：
- 套接字位于 `/tmp/{dataflow_id}/{node_id}.sock`
- 权限：`0o700`
- 与 TCP 相同的 bincode 帧格式

### Daemon 到 Daemon（Zenoh）

| 属性 | 值 |
|------|-----|
| 传输 | Zenoh 发布/订阅 |
| 路由端口 | 7447 |
| 对等端口 | 5456 |
| 路由 | linkstate |
| 序列化 | bincode |

**主题模式：**
```
adora/{network_id}/{dataflow_id}/output/{node_id}/{output_id}
```

## 数据格式与元数据

### Apache Arrow

所有数据有效载荷使用 128 字节对齐的 Apache Arrow 列式格式。

### 元数据

每条消息携带结构化元数据，包括时间戳、Arrow 类型信息和用户定义的参数。

### 约定的元数据键

| 键 | 用途 |
|----|------|
| `request_id` | Service 请求/应答关联 |
| `goal_id` | Action 目标标识符 |
| `goal_status` | Action 完成状态：`succeeded`、`aborted`、`canceled` |
| `session_id` | Streaming 会话标识符 |
| `segment_id` | Streaming 会话内的片段 |
| `seq` | Streaming 块序列号 |
| `fin` | Streaming 片段的最后一个块 |
| `flush` | 丢弃输入上较旧的排队消息 |

## 零拷贝共享内存

### 阈值和限制

| 参数 | 值 |
|------|-----|
| `ZERO_COPY_THRESHOLD` | 4096 字节 |
| 控制区域大小 | 每节点 4 KiB |
| 事件区域大小 | 每节点 4 KiB |
| Drop 区域大小 | 每节点 4 KiB |
| 最大缓存数量 | 20 个区域 |
| 最大缓存字节 | 256 MiB |

## 关键常量和默认值

| 常量 | 值 | 位置 |
|------|-----|------|
| `ADORA_COORDINATOR_PORT_WS_DEFAULT` | 6013 | Coordinator WebSocket 端口 |
| `ADORA_DAEMON_LOCAL_LISTEN_PORT_DEFAULT` | 53291 | Daemon TCP 监听端口 |
| `ZERO_COPY_THRESHOLD` | 4096 字节 | 共享内存激活阈值 |
| `MAX_MESSAGE_BYTES` | 64 MiB | 最大 TCP/bincode 消息 |
| `TCP_READ_TIMEOUT` | 30 秒 | 套接字读取超时 |
| `WS_PING_INTERVAL` | 10 秒 | WebSocket 心跳保活 |
| `MAX_WS_CONNECTIONS` | 256 | 并发 WebSocket 限制 |
| `WATCHDOG_INTERVAL` | 5 秒 | 到 coordinator 的心跳 |
| `METRICS_INTERVAL` | 2 秒 | 指标收集 |
| `HEALTH_CHECK_INTERVAL` | 5 秒 | 默认节点健康检查 |
| 默认输入队列大小 | 10 | 每输入消息缓冲区 |
