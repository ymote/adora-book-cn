> 🌐 **[English](https://ymote.github.io/adora-book/)**

# Adora

**Agentic Dataflow-Oriented Robotic Architecture** -- 一个 100% Rust 编写的实时机器人与 AI 应用框架。

## 为什么选择 Adora?

### 性能

- **比 ROS2 Python 快 10-17 倍** -- 100% Rust 内核，零拷贝共享内存 IPC 适用于大于 4KB 的消息，从 4KB 到 4MB 有效载荷延迟平坦
- **Apache Arrow 原生支持** -- 端到端列式内存格式，零序列化开销；所有语言绑定共享
- **Zenoh SHM 数据平面** -- 节点通过 Zenoh 共享内存直接发布，实现零拷贝数据传输；跨机器自动回退到网络传输
- **非阻塞事件循环** -- Zenoh 发布任务卸载到 drain task；指标收集在后台运行

### 开发者体验

- **单一 CLI，完整生命周期** -- `adora run` 用于本地开发，`adora up/start` 用于分布式生产环境，另有构建、日志、监控、录制/回放等功能，全部集成在一个工具中
- **声明式 YAML 数据流** -- 以有向图定义管道，通过类型化输入/输出连接节点，支持可选的[类型注解](concepts/types.md)与静态验证
- **多语言节点** -- 使用 Rust、Python、C 或 C++ 编写节点，均为原生 API（非封装层）；可在一个数据流中自由混合不同语言
- **[可复用模块](concepts/modules.md)** -- 将子图组合为独立的 YAML 文件，支持类型化输入/输出、参数和嵌套组合
- **热重载** -- 无需重启数据流即可实时重载 Python 算子

### 生产就绪

- **容错机制** -- 按节点设置重启策略（never/on-failure/always），支持指数退避、健康监控、带有可配置输入超时的断路器
- **默认分布式** -- 同机节点之间使用本地共享内存，跨机器通信自动使用 [Zenoh](https://zenoh.io/) 发布/订阅，基于 SSH 的集群管理与标签调度
- **Coordinator 高可用** -- 持久化 redb 状态存储，daemon 自动重连，coordinator 重启后数据流状态重建
- **动态拓扑** -- 通过 CLI 在运行中的数据流中添加和移除节点，无需重启
- **可配置队列策略** -- 每个输入支持 `drop_oldest`（默认）或 `backpressure`，附带丢弃消息的指标统计
- **软实时** -- 可选 `--rt` 标志启用 mlockall + SCHED_FIFO；支持按节点 `cpu_affinity` 绑定
- **OpenTelemetry** -- 内置结构化日志（支持轮转/路由）、指标、分布式追踪

### 调试与可观测性

- **录制/回放** -- 将数据流消息捕获到 `.adorec` 文件，可离线以任意速度回放并替换节点
- **主题检查** -- `topic echo` 打印实时数据，`topic hz` TUI 频率分析，`topic info` 查看 schema 和带宽
- **资源监控** -- `adora top` TUI 显示所有机器上每个节点的 CPU、内存、队列深度、网络 I/O
- **日志聚合** -- 订阅 `adora/logs` 即可接收所有节点的结构化日志消息，无需额外配置
- **追踪检查** -- `trace list` 和 `trace view` 用于查看 coordinator span，无需外部基础设施

### 生态系统

- **通信模式** -- 内置 [service (请求/应答)](concepts/patterns.md)、[action (目标/反馈/结果)](concepts/patterns.md) 和 [streaming (会话/片段/块)](concepts/patterns.md) 模式，通过约定的元数据键实现
- **ROS2 桥接** -- 与 ROS2 topics、services 和 actions 双向互操作
- **进程内算子** -- 在共享运行时中运行的轻量级函数，避免了逐节点的进程开销

## 下一步

- [安装 Adora](getting-started/installation.md)
- [快速开始教程](getting-started/quickstart.md)
- [架构概述](concepts/architecture.md)
- [数据流 YAML 参考](concepts/dataflow-yaml.md)
- [类型注解](concepts/types.md)
- [通信模式](concepts/patterns.md)
