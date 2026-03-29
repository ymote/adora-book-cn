> 🌐 **[English](../../operations/distributed.md)**

# 分布式部署指南

Adora 支持跨多台机器部署数据流，适用于多机器人车队、边缘 AI 管道和分布式机器人系统。本指南涵盖集群管理、节点调度、二进制分发、自动恢复和运维最佳实践。

## 目录

- [概述](#概述)
- [快速入门](#快速入门)
- [功能概览](#功能概览)
- [集群配置参考](#集群配置参考)
- [集群命令参考](#集群命令参考)
- [节点调度](#节点调度)
- [二进制分发](#二进制分发)
- [systemd 服务管理](#systemd-服务管理)
- [自动恢复](#自动恢复)
- [滚动升级](#滚动升级)
- [使用场景](#使用场景)
- [运维手册](#运维手册)
- [部署 YAML 参考](#部署-yaml-参考)
- [最佳实践](#最佳实践)

---

## 概述

Adora 的分布式架构有三层：

```
CLI  -->  Coordinator  -->  Daemon(s)  -->  节点 / 算子
              (一个)         (每台机器)       (用户代码)
```

- **CLI** 向 coordinator 发送控制命令（构建、启动、停止）。
- **Coordinator** 编排 daemon，解析节点放置，管理数据流生命周期。
- **Daemon** 在每台机器上运行，生成和监管节点进程。
- **节点** 通过共享内存（同机）或 Zenoh 发布/订阅（跨机器）通信。

分布式部署有两种路径：

**临时模式** -- 在每台机器上手动启动 `adora daemon`，然后使用 coordinator 进行控制。适合开发和测试。参见 [CLI 参考中的分布式部署](cli.md#分布式部署)。

**托管模式（cluster.yml）** -- 在 YAML 文件中定义集群拓扑，然后使用 `adora cluster` 命令进行基于 SSH 的生命周期管理。本指南重点介绍托管路径。

---

## 快速入门

1. 创建 `cluster.yml`：

```yaml
coordinator:
  addr: 10.0.0.1
machines:
  - id: robot
    host: 10.0.0.2
    user: ubuntu
  - id: gpu-server
    host: 10.0.0.3
    user: ubuntu
```

2. 启动集群：

```bash
adora cluster up cluster.yml
```

3. 启动数据流：

```bash
adora start dataflow.yml --name my-app --attach
```

4. 检查集群健康：

```bash
adora cluster status
```

5. 关闭：

```bash
adora cluster down
```

---

## 功能概览

| 功能 | 命令 / 配置 | 描述 |
|------|------------|------|
| 集群生命周期 | `adora cluster up/status/down` | 从单台机器进行基于 SSH 的 daemon 管理 |
| 标签调度 | `_unstable_deploy.labels` | 通过键值标签将节点路由到 daemon |
| 二进制分发 | `_unstable_deploy.distribute` | local、scp 或 http 策略 |
| systemd 服务 | `adora cluster install/uninstall` | 重启后存活的持久 daemon 服务 |
| 自动恢复 | 自动 | daemon 重连时重新生成节点 |
| 滚动升级 | `adora cluster upgrade` | SCP 二进制 + 逐机器顺序重启 |
| 数据流重启 | `adora cluster restart` | 按名称或 UUID 重启运行中的数据流 |

---

## 集群配置参考

`cluster.yml` 文件定义 coordinator 地址和集群中的机器集合。

### 完整 Schema

```yaml
coordinator:
  addr: 10.0.0.1            # coordinator 绑定的 IP 地址（必填）
  port: 6013                 # WebSocket 端口（默认：6013）

machines:
  - id: edge-01              # 唯一机器标识符（必填）
    host: 10.0.0.2           # SSH 可达的主机名或 IP（必填）
    user: ubuntu              # SSH 用户（可选，默认为当前用户）
    labels:                   # 用于调度的键值标签（可选）
      gpu: "true"
      arch: arm64

  - id: edge-02
    host: 10.0.0.3
    labels:
      arch: arm64
```

### 字段

**coordinator**

| 字段 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `addr` | IP 地址 | （必填） | coordinator 绑定的地址 |
| `port` | u16 | `6013` | WebSocket 端口 |

**machines[]**

| 字段 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `id` | string | （必填） | 唯一机器标识符，用于 `_unstable_deploy.machine` |
| `host` | string | （必填） | SSH 可达的主机名或 IP 地址 |
| `user` | string | 当前用户 | SSH 用户名 |
| `labels` | map | 空 | 用于基于标签调度的键值对 |

### 验证规则

- 必须定义至少一台机器。
- 机器 ID 必须非空且唯一。
- 机器主机必须非空。
- 未知字段会被拒绝（`deny_unknown_fields`）。

### 示例：3 台机器的 GPU 集群

```yaml
coordinator:
  addr: 192.168.1.1

machines:
  - id: coordinator-host
    host: 192.168.1.1
    labels:
      role: control

  - id: gpu-a100
    host: 192.168.1.10
    user: ml
    labels:
      gpu: a100
      arch: x86_64

  - id: jetson-01
    host: 192.168.1.20
    user: nvidia
    labels:
      gpu: jetson
      arch: arm64
```

---

## 集群命令参考

所有 `adora cluster` 命令操作 `cluster.yml` 文件，使用 SSH 管理远程机器。

使用的 SSH 选项：`BatchMode=yes`、`ConnectTimeout=10`、`StrictHostKeyChecking=accept-new`。

### adora cluster up

从 cluster.yml 文件启动多机器集群。在本地启动 coordinator，然后 SSH 到每台机器启动 daemon。

```
adora cluster up <PATH>
```

**参数：**

| 参数 | 描述 |
|------|------|
| `PATH` | 集群配置文件路径 |

**行为：**

1. 加载并验证集群配置。
2. 在 `addr:port` 本地启动 coordinator。
3. 对每台机器，SSH 进入并运行 `nohup adora daemon --machine-id <id> --coordinator-addr <addr> --coordinator-port <port> [--labels k1=v1,k2=v2] --quiet`。
4. 轮询直到所有预期 daemon 在 coordinator 注册（30 秒超时）。

**示例：**

```bash
$ adora cluster up cluster.yml
Starting coordinator on 10.0.0.1:6013...
Starting daemon on robot (ubuntu@10.0.0.2)... OK
Starting daemon on gpu-server (ubuntu@10.0.0.3)... OK
All 2 daemons connected.
```

### adora cluster status

显示集群的当前状态。显示已连接的 daemon 和活跃数据流数量。

```
adora cluster status [--coordinator-addr ADDR] [--coordinator-port PORT]
```

**标志：**

| 标志 | 默认值 | 描述 |
|------|--------|------|
| `--coordinator-addr` | `localhost` | Coordinator 主机名或 IP |
| `--coordinator-port` | `6013` | Coordinator WebSocket 端口 |

**示例：**

```bash
$ adora cluster status
DAEMON ID      LAST HEARTBEAT
robot          2s ago
gpu-server     1s ago

Active dataflows: 1
```

### adora cluster down

关闭集群（coordinator 和所有 daemon）。

```
adora cluster down [--coordinator-addr ADDR] [--coordinator-port PORT]
```

终止所有 daemon 和 coordinator 进程。

### adora cluster install

在每台机器上将 `adora-daemon` 安装为 systemd 服务。SSH 到每台机器，写入 systemd 单元文件并启用服务。

```
adora cluster install <PATH>
```

**参数：**

| 参数 | 描述 |
|------|------|
| `PATH` | 集群配置文件路径 |

**行为：**

对每台机器，创建并启用名为 `adora-daemon-<id>` 的 systemd 服务。单元文件：

```ini
[Unit]
Description=Adora Daemon (<id>)
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=adora daemon --machine-id <id> --coordinator-addr <addr> --coordinator-port <port> --labels k1=v1,k2=v2 --quiet
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

**示例：**

```bash
$ adora cluster install cluster.yml
Installing adora-daemon-robot on ubuntu@10.0.0.2... OK
Installing adora-daemon-gpu-server on ubuntu@10.0.0.3... OK
2/2 succeeded.
```

### adora cluster uninstall

从每台机器卸载 adora-daemon systemd 服务。停止、禁用并删除 systemd 单元。

```
adora cluster uninstall <PATH>
```

**行为：**

对每台机器，运行：

```bash
sudo systemctl stop adora-daemon-<id>
sudo systemctl disable adora-daemon-<id>
sudo rm -f /etc/systemd/system/adora-daemon-<id>.service
sudo systemctl daemon-reload
```

### adora cluster upgrade

滚动升级：SCP 本地 `adora` 二进制到每台机器并重启 daemon。逐台机器顺序处理以保持可用性。

```
adora cluster upgrade <PATH>
```

**行为：**

对每台机器顺序执行：

1. SCP 本地 `adora` 二进制到目标机器的 `/usr/local/bin/adora`。
2. 通过 `sudo systemctl restart adora-daemon-<id>` 重启 systemd 服务。
3. 轮询 coordinator 直到 daemon 重连（30 秒超时，500ms 间隔）。

其他机器上的节点在每台机器升级期间继续运行。

**示例：**

```bash
$ adora cluster upgrade cluster.yml
Upgrading robot (ubuntu@10.0.0.2)...
  SCP binary... OK
  Restart service... OK
  Waiting for reconnect... OK (3.2s)
Upgrading gpu-server (ubuntu@10.0.0.3)...
  SCP binary... OK
  Restart service... OK
  Waiting for reconnect... OK (2.8s)
2/2 succeeded.
```

### adora cluster restart

按名称或 UUID 重启运行中的数据流。停止数据流并立即使用存储的描述符重新启动（无需 YAML 路径）。

```
adora cluster restart <DATAFLOW>
```

**参数：**

| 参数 | 描述 |
|------|------|
| `DATAFLOW` | 要重启的数据流名称或 UUID |

**示例：**

```bash
$ adora cluster restart my-app
Restarting dataflow `my-app`
dataflow restarted: a1b2c3d4-... -> e5f6a7b8-...
```

---

## 节点调度

当 coordinator 接收到数据流时，它根据数据流 YAML 中的 `_unstable_deploy` 部分决定哪个 daemon 运行每个节点。解析优先级：**machine > labels > 未命名**。

### 基于机器的调度

通过 `cluster.yml` 中的 `id` 将节点分配到特定机器：

```yaml
nodes:
  - id: camera
    _unstable_deploy:
      machine: robot
    path: ./camera-driver
    outputs:
      - frames
```

coordinator 查找 `machine-id` 匹配的 daemon。如果没有匹配的 daemon 连接，部署失败并报告：`no matching daemon for machine id "robot"`。

### 基于标签的调度

通过要求目标 daemon 具有特定标签来分配节点：

```yaml
nodes:
  - id: inference
    _unstable_deploy:
      labels:
        gpu: "true"
    path: ./ml-model
    inputs:
      frames: camera/frames
    outputs:
      - predictions
```

coordinator 查找第一个标签是所需标签**超集**的已连接 daemon。所有必需的键值对必须精确匹配。如果没有 daemon 满足要求，部署失败并报告：`no daemon matches labels {"gpu": "true"}`。

### 未分配的节点

没有 `_unstable_deploy` 部分（或部分为空）的节点被分配到第一个未命名 daemon -- 即不带 `--machine-id` 标志连接的 daemon。

### resolve_daemon() 的内部工作原理

coordinator 在 `coordinator/run/mod.rs` 中解析节点放置：

```
resolve_daemon(connections, deploy) -> DaemonId
  1. 如果 deploy.machine 是 Some(id)：
       -> 按 machine-id 查找 daemon
  2. 否则如果 deploy.labels 非空：
       -> 查找第一个所有必需标签匹配的 daemon
  3. 否则：
       -> 选择第一个未命名 daemon
```

标签匹配函数遍历所有已连接 daemon，检查每个必需键值对是否存在于 daemon 的标签集中（`conn.labels.get(k) == Some(v)`）。这是超集检查：具有 `{gpu: "true", arch: "arm64", role: "edge"}` 的 daemon 满足要求 `{gpu: "true"}`。

---

## 二进制分发

通过 `distribute` 字段控制节点二进制如何传递到远程 daemon。

### Local（默认）

每个 daemon 在自己的机器上从源码构建。这是当前的默认行为。

```yaml
nodes:
  - id: my-node
    _unstable_deploy:
      machine: edge-01
      distribute: local
    path: ./my-node
```

### SCP 模式

CLI 在生成前通过 SSH/SCP 将本地构建的二进制推送到目标机器。

```yaml
nodes:
  - id: my-node
    _unstable_deploy:
      machine: edge-01
      distribute: scp
    path: ./my-node
```

### HTTP 模式

coordinator 运行制品存储。daemon 在生成前通过 HTTP 从 coordinator 拉取二进制。

```yaml
nodes:
  - id: my-node
    _unstable_deploy:
      machine: edge-01
      distribute: http
    path: ./my-node
```

制品通过 coordinator WebSocket 端口上的 `GET /api/artifacts/{build_id}/{node_id}` 提供。端点需要认证（Bearer token）并对节点 ID 进行净化以防止路径遍历。

### 何时使用每种策略

| 策略 | 最适合 | 权衡 |
|------|--------|------|
| `local` | 同构集群、CI 构建 | 每台机器需要构建工具链 |
| `scp` | 异构集群、交叉编译二进制 | 需要从 CLI 到所有机器的 SSH 访问 |
| `http` | 隔离的 daemon、防火墙网络 | 需要所有 daemon 可达 coordinator |

---

## systemd 服务管理

生产部署中，将 daemon 安装为 systemd 服务，使其在重启后存活并在故障时自动重启。

### 安装

```bash
adora cluster install cluster.yml
```

在每台机器上创建 systemd 单元文件（参见 [adora cluster install](#adora-cluster-install) 获取完整单元模板）。关键属性：

- **Restart=on-failure** 和 **RestartSec=5**：daemon 崩溃后自动重启。
- **After=network-online.target**：启动前等待网络。
- **WantedBy=multi-user.target**：开机启动。

### 卸载

```bash
adora cluster uninstall cluster.yml
```

在每台机器上停止、禁用并删除单元文件，然后重新加载 systemd daemon。

### 验证服务状态

安装后，直接检查服务：

```bash
ssh ubuntu@10.0.0.2 sudo systemctl status adora-daemon-robot
```

---

## 自动恢复

当 daemon 断开并重连时（如网络闪断、机器重启或服务重启后），coordinator 自动重新生成该 daemon 上的缺失数据流。

### 工作原理

1. Daemon 重连并发送 `StatusReport`，列出其当前运行的数据流。
2. Coordinator 将报告与其预期状态（应在此 daemon 上有节点的数据流）进行比较。
3. 对于每个在此 daemon 上有分配节点但 daemon 未报告的运行中数据流，coordinator 发送 `SpawnDataflowNodes` 命令重新生成缺失节点。

### 30 秒退避

为防止崩溃循环（如节点在生成后立即崩溃），恢复使用每 daemon、每数据流的退避：

- 恢复尝试后，coordinator 记录时间戳。
- 同一 daemon/数据流对的后续恢复在 30 秒内被跳过。
- 当 daemon 报告数据流正在运行时，退避清除。

这意味着立即崩溃的节点每 30 秒只会被重新生成一次，而不是在紧密循环中。

### 限制

- 自动恢复仅适用于通过 `adora start`（coordinator 管理的）启动的数据流。本地 `adora run` 数据流不被 coordinator 跟踪。
- 恢复重新生成分配到重连 daemon 的所有节点，而非单个节点。对于单节点崩溃重启，使用[重启策略](fault-tolerance.md#重启策略)。

---

## 滚动升级

使用逐台机器顺序升级来实现零停机时间地升级所有集群机器上的 `adora` 二进制。

### 流程

```bash
adora cluster upgrade cluster.yml
```

对每台机器，顺序执行：

1. **SCP** 本地 `adora` 二进制到目标机器的 `/usr/local/bin/adora`。
2. **重启** systemd 服务（`systemctl restart adora-daemon-<id>`）。
3. **轮询** coordinator 直到 daemon 重连（30 秒超时）。

因为机器逐一升级，其他机器上的节点继续运行。daemon 重连后，自动恢复重新生成之前在该机器上运行的数据流节点。

### 前提条件

- daemon 必须已安装为 systemd 服务（`adora cluster install`）。
- 本地 `adora` 二进制必须与集群 coordinator 版本兼容。
- 所有目标机器上的 SSH 访问具有 `sudo` 权限。

---

## 使用场景

### 1. 边缘 AI 管道（机器人 + GPU 服务器）

相机节点在机器人上运行，将帧发送到 GPU 服务器进行推理，结果回流到机器人上的执行器。

**cluster.yml：**

```yaml
coordinator:
  addr: 192.168.1.1

machines:
  - id: robot
    host: 192.168.1.10
    user: ubuntu
    labels:
      role: edge
  - id: gpu-server
    host: 192.168.1.20
    user: ml
    labels:
      gpu: "true"
```

**dataflow.yml：**

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
      labels:
        gpu: "true"
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

### 2. 多机器人车队

中央 coordinator 管理 N 个异构硬件的机器人。标签调度将节点路由到正确的机器，无需硬编码机器 ID。

**cluster.yml：**

```yaml
coordinator:
  addr: 10.0.0.1

machines:
  - id: bot-01
    host: 10.0.0.11
    user: robot
    labels:
      fleet: warehouse
      lidar: "true"

  - id: bot-02
    host: 10.0.0.12
    user: robot
    labels:
      fleet: warehouse
      camera: rgbd

  - id: bot-03
    host: 10.0.0.13
    user: robot
    labels:
      fleet: warehouse
      lidar: "true"
      camera: rgbd
```

**dataflow.yml：**

```yaml
nodes:
  - id: lidar-driver
    _unstable_deploy:
      labels:
        lidar: "true"
    path: ./lidar-driver
    outputs:
      - scans

  - id: camera-driver
    _unstable_deploy:
      labels:
        camera: rgbd
    path: ./camera-driver
    outputs:
      - frames
```

使用此配置，`lidar-driver` 在 bot-01 或 bot-03 上运行，`camera-driver` 在 bot-02 或 bot-03 上运行。

### 3. 机器人的 CI/CD 管道

在 CI 中自动化集群管理：

```bash
# 设置
adora cluster install cluster.yml

# 部署新版本
adora cluster upgrade cluster.yml

# 运行集成测试
adora start test-dataflow.yml --name integration-test --attach

# 监控
adora cluster status
adora top

# 清理
adora stop integration-test
```

### 4. 从开发到生产

| 阶段 | 方式 | 命令 |
|------|------|------|
| **本地开发** | 单进程，无 coordinator | `adora run dataflow.yml` |
| **预发布** | 临时 daemon，手动设置 | `adora up` + 每台机器 `adora daemon` |
| **生产** | 托管集群，systemd 服务 | `adora cluster install cluster.yml` |

---

## 运维手册

### 初始设置检查清单

1. **SSH 密钥**：分发 SSH 密钥，使 CLI 机器能无密码到达所有集群机器（`BatchMode=yes`）。
2. **Adora 二进制**：在所有机器上安装 `adora` 二进制（相同版本）。
3. **网络**：确保 coordinator 端口（默认 6013）从所有机器可达。确保 daemon 之间的 Zenoh 端口开放以支持跨机器节点通信。
4. **cluster.yml**：使用正确的 IP、用户和标签创建集群配置。

### 日常运维

```bash
# 启动数据流
adora start dataflow.yml --name my-app --attach

# 列出运行中的数据流
adora list

# 监控资源使用
adora top

# 查看节点日志
adora logs my-app <node-id> --follow

# 停止数据流
adora stop my-app

# 检查集群健康
adora cluster status
```

### 升级

1. 在本地构建或下载新的 `adora` 二进制。
2. 运行 `adora cluster upgrade cluster.yml`。
3. 使用 `adora cluster status` 验证所有 daemon 已重连。
4. 运行中的数据流通过自动恢复自动重新生成。

### 故障排除

**Daemon 未连接**

- 验证 coordinator 正在运行且可达：`curl http://<addr>:6013/api/health`（或检查 coordinator 日志）。
- 检查 daemon 日志：`journalctl -u adora-daemon-<id> -f`（systemd）或 daemon 的 stderr 输出（临时模式）。
- 确认 `--coordinator-addr` 和 `--coordinator-port` 与 coordinator 的实际绑定地址匹配。

**集群命令期间的 SSH 失败**

- 确保 `ssh -o BatchMode=yes <user>@<host> echo ok` 从 CLI 机器可用。
- 检查 `StrictHostKeyChecking=accept-new` 是否适合你的环境（首次连接自动接受主机密钥）。
- 验证 `cluster.yml` 中的 `user` 字段与目标上的有效 SSH 用户匹配。

**标签不匹配错误**

- 错误：`no daemon matches labels {"gpu": "true"}`。
- 检查 daemon 是否以正确的 `--labels` 标志启动。
- 运行 `adora cluster status` 查看已连接 daemon。标签在 daemon 启动时从 `cluster.yml` 设置，运行时不可更改。

**自动恢复未触发**

- 自动恢复仅适用于 coordinator 管理的数据流（`adora start`），不适用于 `adora run`。
- 检查 coordinator 日志中的 `auto-recovery: re-spawning` 消息。
- 如果节点立即崩溃，每个 daemon 每个数据流的恢复被限制为每 30 秒一次。

---

## 部署 YAML 参考

每个节点上的 `_unstable_deploy` 部分控制放置和分发。所有字段都是可选的。

```yaml
nodes:
  - id: my-node
    _unstable_deploy:
      machine: edge-01                # cluster.yml 中的目标机器 ID
      labels:                          # 标签要求（超集匹配）
        gpu: "true"
        arch: arm64
      distribute: local                # local | scp | http
      working_dir: /opt/my-app         # 目标机器上的工作目录
    path: ./my-node
```

### 字段

| 字段 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `machine` | string | 无 | 目标机器 ID。优先于标签。 |
| `labels` | map | 空 | 必需的 daemon 标签。所有键值对必须匹配。 |
| `distribute` | string | `local` | 二进制分发策略：`local`、`scp` 或 `http`。 |
| `working_dir` | path | 无 | 目标机器上的工作目录。 |

### 解析优先级

1. **machine** -- 如果设置，节点分配到具有该机器 ID 的 daemon。
2. **labels** -- 如果设置（且未设置 machine），节点分配到第一个标签是所需标签超集的 daemon。
3. **后备** -- 如果两者都未设置，节点分配到第一个未命名（无 machine-id）的 daemon。

---

## 最佳实践

- **优先使用标签而非机器 ID** 以获得灵活性。标签将数据流与特定机器解耦，使添加、移除或替换硬件更容易。
- **生产环境使用 systemd install**。Daemon 服务在重启后存活，并通过 `Restart=on-failure` 在故障时自动重启。
- **使用 coordinator 持久化**（`adora coordinator --store redb`）配合集群，使 coordinator 在重启后存活。参见[Coordinator 状态持久化](fault-tolerance.md#coordinator-状态持久化)。
- **在节点上设置重启策略** 以获得每节点的弹性。结合自动恢复实现纵深防御。参见[重启策略](fault-tolerance.md#重启策略)。
- **使用多种工具监控**：`adora cluster status` 查看 daemon 健康，`adora top` 查看资源使用，`adora logs` 查看节点输出。
- **先在本地测试**。使用 `adora run dataflow.yml` 开发，然后部署到集群。相同的数据流 YAML 在两种模式下都有效 -- 本地模式下 `_unstable_deploy` 字段被忽略。
- **使用滚动升级** 而非停止整个集群。`adora cluster upgrade` 一次处理一台机器以保持可用性。
- **将 cluster.yml 纳入版本控制** 与数据流定义一起。
