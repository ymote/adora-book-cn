> 🌐 **[English](../../operations/performance.md)**

# 性能

Adora 通过零拷贝共享内存 IPC、Apache Arrow 列式格式和 100% Rust 内核，实现了比 ROS2 Python 低 10-17 倍的延迟。本文档涵盖方法论、复现步骤和调优方法。

## 架构优势

| 层 | Adora | ROS2 (rclpy) |
|----|-------|---------------|
| 运行时 | Rust 异步（tokio） | Python + C++ 中间件 |
| IPC（>4KB） | Zenoh SHM 零拷贝 | DDS 序列化 + 拷贝 |
| IPC（<4KB） | TCP + bincode | DDS 序列化 + 拷贝 |
| 数据格式 | Apache Arrow（零序列化） | CDR 序列化 |
| 线程 | 无锁通道（flume） | GIL 绑定的回调 |
| 扇出 | Arc 包装（每接收者 O(1)） | 每接收者拷贝 |

## 基准测试套件

### 内部基准测试（`examples/benchmark/`）

测量 Adora 在 10 种有效载荷大小（0B 到 4MB）下的延迟和吞吐量。

```bash
cd examples/benchmark
./compare.sh          # Rust vs Python 发送者对比
```

报告的指标：avg、p50、p95、p99、p99.9、min、max 延迟；msg/s 吞吐量。

### ROS2 对比（`examples/ros2-comparison/`）

使用两个框架上相同的 Python 工作负载进行同等对比。

```bash
cd examples/ros2-comparison
./run_comparison.sh   # 需要 ROS2 Humble+
```

双方都使用 `time.perf_counter_ns()` 时间戳嵌入有效载荷前 8 字节。相同的消息数量、大小和休眠间隔确保结果可比。

### Criterion 微基准测试

针对内部热路径的隔离基准测试：

```bash
# Daemon 消息路由（扇出 x 有效载荷大小矩阵）
cargo bench -p adora-daemon

# 消息序列化/反序列化
cargo bench -p adora-message
```

CI 通过 `benchmark-action/github-action-benchmark` 跟踪这些测试，告警阈值为 120%。

## 复现结果

### 环境要求

- Linux 或 macOS（共享内存 IPC）
- Rust 1.85+ 并使用 release 配置
- Python 3.10+，安装 `numpy`、`pyarrow`
- ROS2 Humble+（仅用于对比）

### 步骤

1. 构建 Adora：
   ```bash
   cargo install --path binaries/cli --locked
   ```

2. 运行内部基准测试：
   ```bash
   cd examples/benchmark
   BENCH_CSV=results/rust.csv adora run dataflow.yml
   ```

3. 运行 ROS2 对比：
   ```bash
   cd examples/ros2-comparison
   ./run_comparison.sh
   ```

### 环境注意事项

- 关闭后台应用以减少方差
- 使用 `taskset` 或 `cpuset` 将进程固定到特定 CPU 以获得一致的结果
- 至少运行 3 次迭代并报告中位数
- 共享内存的优势在有效载荷 >4KB 时体现

## 性能调优

### 队列大小

默认队列大小为 10。对于高吞吐量输出，增加它：

```yaml
inputs:
  data:
    source: producer/output
    queue_size: 1000
```

### 有效载荷大小

Adora 对 >4KB 的消息自动使用共享内存，避免拷贝。当低延迟很重要时，将数据结构设计为超过此阈值。

### Arrow 格式

直接使用 Arrow 数组，而不是在 Python 列表之间转换：

```python
# 快速：直接传递 Arrow 数组
node.send_output("out", pa.array(data, type=pa.uint8()))

# 慢：通过 Python 列表转换
node.send_output("out", pa.array(list(data), type=pa.uint8()))
```

### Operator 与 Node

Operator 在运行时进程内运行（零 IPC 开销），但在 Python 中共享 GIL。对计算密集型工作使用 Rust operator，对胶水逻辑使用 Python operator。

### 分布式部署

对于跨机器通信，Adora 使用 Zenoh pub-sub。延迟取决于网络质量。当需要亚毫秒级延迟时，使用本地部署（单机）。

## CSV 输出格式

所有基准测试支持 `BENCH_CSV` 环境变量以输出机器可读的结果：

```
latency,<bytes>,<label>,<n>,<avg_ns>,<p50_ns>,<p95_ns>,<p99_ns>,<p999_ns>,<min_ns>,<max_ns>
throughput,<bytes>,<label>,<n>,<msg_per_sec>,<elapsed_ns>,0,0,0,0,0
```
