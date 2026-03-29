> **[English](../../operations/realtime-tuning.md)**

# 实时调优

Adora 为延迟敏感的机器人部署提供可选的实时功能。

## 快速入门

```bash
# 以实时配置启动 daemon（mlockall + SCHED_FIFO）
sudo adora daemon --rt

# 控制工作线程
adora daemon --worker-threads 4

# 固定到特定 CPU 核心
taskset -c 2,3 adora daemon --rt
```

## `--rt` 的作用

- `mlockall(MCL_CURRENT | MCL_FUTURE)` -- 锁定所有内存，防止页面错误
- `SCHED_FIFO` 优先级 50（仅 Linux）-- 主线程的实时调度
- 需要 `CAP_SYS_NICE` + `CAP_IPC_LOCK` 能力

## 每节点 CPU 亲和性

通过数据流 YAML 将单个节点固定到特定 CPU 核心：

```yaml
- id: controller
  path: ./controller
  cpu_affinity: [2, 3]

- id: sensor
  path: ./sensor
  cpu_affinity: [4, 5]
```

daemon 对生成的进程调用 `sched_setaffinity`。仅适用于 Linux；在其他平台上该字段被静默忽略。

结合 `--rt` 获得最佳效果：daemon 获得实时调度，而每个节点固定到专用核心，避免争用。
