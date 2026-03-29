> 🌐 **[English](../../development/testing.md)**

# Adora 测试指南

本指南介绍如何在 Adora 工作空间中运行、编写和排查测试。

## 快速入门（5 分钟验证）

运行以下三个命令来验证工作空间是否健康：

```bash
# 1. 格式检查（约 5 秒）
cargo fmt --all -- --check

# 2. 代码检查（首次约 60 秒，之后使用缓存）
cargo clippy --all \
  --exclude adora-node-api-python \
  --exclude adora-operator-api-python \
  --exclude adora-ros2-bridge-python \
  -- -D warnings

# 3. 单元 + 集成测试（首次约 90 秒）
cargo test --all \
  --exclude adora-node-api-python \
  --exclude adora-operator-api-python \
  --exclude adora-ros2-bridge-python
```

提交 PR 前这三项必须全部通过。Python 包因需要 maturin 而被排除。

## 测试层级

| 层级 | 覆盖内容 | 命令 | 速度 |
|------|----------|------|------|
| **格式** | 代码风格 | `cargo fmt --all -- --check` | 约 5 秒 |
| **代码检查** | 警告、正确性 | `cargo clippy --all ...` | 约 60 秒 |
| **单元测试** | 单个函数 | `cargo test --all ...` | 约 90 秒 |
| **CLI** | 命令解析、验证 | `cargo test -p adora-cli` | 约 5 秒 |
| **集成** | 通过环境变量的节点 I/O | `cargo test --test example-tests` | 约 30 秒 |
| **冒烟测试** | 完整 CLI 生命周期 | `cargo test --test example-smoke -- --test-threads=1` | 约 3 分钟 |
| **端到端** | 多数据流场景 | `cargo test --test ws-cli-e2e -- --ignored --test-threads=1` | 约 2 分钟 |
| **容错** | 重启策略、超时 | `cargo test --test fault-tolerance-e2e` | 约 45 秒 |
| **拼写检查** | 拼写 | 安装 [typos-cli](https://github.com/crate-ci/typos)，然后 `typos` | 约 2 秒 |

## 层级详情

### 单元测试

单元测试位于使用 `#[cfg(test)]` 模块的代码旁边。包含测试的关键 crate：

| Crate | 测试数量 | 测试内容 |
|-------|----------|---------|
| adora-arrow-convert | 约 26 | 往返 Arrow 类型转换 |
| adora-cli | 约 96 | 命令解析、值解析器、日志 grep/过滤、JSON 解析、WebSocket 客户端、集群配置 |
| adora-coordinator | 约 24 | WS 控制/daemon 平面、健康检查、并发请求、制品存储、速率限制器、错误净化 |
| adora-coordinator-store | 约 10 | 内存和 redb CRUD、schema 版本控制、持久化 |
| adora-core | 约 8 | 数据流描述符验证 |
| adora-daemon | 约 2 | Shlex 参数解析 |
| adora-node-api | 约 10 | 输入跟踪、service/action 辅助函数（ID 生成、send_service_request/response） |
| adora-log-utils | 约 11 | 日志解析工具 |
| adora-message | 约 36 | 通用类型、WS 协议、节点/数据 ID、元数据、认证令牌 |
| ros2-bridge | 约 30 | ROS2 消息/service/action 解析 |

运行单个 crate 的测试：

```bash
cargo test -p adora-cli
cargo test -p adora-core
cargo test -p adora-arrow-convert
```

### CLI 测试

CLI 测试验证命令解析、参数验证和值解析器，无需运行任何命令。它们位于 CLI crate 内的 `#[cfg(test)]` 模块中。

**测试内容：**
- Clap schema 验证（`Args::command().debug_assert()`）
- 每个子命令的解析（`run`、`up`、`down`、`start`、`stop`、`list`、`logs`、`build`、`graph`、`new`、`status`、`inspect top`、`topic list/hz/echo`、`node list`）
- 拒绝未知子命令
- `--help` 和 `--version` 退出码
- 值解析器：`parse_store_spec`（coordinator 存储后端）、`parse_window`（topic hz 窗口）
- 工具函数：`parse_version_from_pip_show`

**运行方式：**

```bash
cargo test -p adora-cli
```

**如何添加新测试：**

添加新 CLI 子命令或值解析器时，在同一文件的 `#[cfg(test)]` 模块中添加相应测试。对于子命令解析，在 `binaries/cli/src/command/mod.rs` 中添加 `parse_ok` 调用。对于值解析器，在定义解析函数的文件中添加测试。

### 集成测试（节点 I/O）

文件：`tests/example-tests.rs`

这些测试使用预录制输入运行编译的节点可执行文件，并将输出与预期基线进行比较。无需 coordinator 或 daemon。

```bash
cargo test --test example-tests
```

工作原理：
1. 构建并运行节点 crate（如 `rust-dataflow-example-node`）
2. 设置 `ADORA_TEST_WITH_INPUTS` 为包含定时事件的 JSON 文件
3. 设置 `ADORA_TEST_NO_OUTPUT_TIME_OFFSET=1` 以确保确定性输出
4. 将 JSONL 输出与 `tests/sample-inputs/expected-outputs-*.jsonl` 进行比较

示例输入/输出文件位于 `tests/sample-inputs/`。

### 冒烟测试

文件：`tests/example-smoke.rs`

每个适用示例测试两种执行模式：

- **网络模式**（`adora up` + `adora start --detach` + 轮询 + `adora stop` + `adora down`）：演练完整的 coordinator/daemon WS 控制平面。
- **本地模式**（`adora run --stop-after`）：一切在进程内运行，测试单进程数据流路径。

```bash
# 必须单线程运行（共享 coordinator 端口）
cargo test --test example-smoke -- --test-threads=1

# 仅运行网络或本地测试
cargo test --test example-smoke smoke_rust -- --test-threads=1
cargo test --test example-smoke smoke_local -- --test-threads=1
```

也提供 bash 脚本用于快速本地验证：

```bash
./scripts/smoke-all.sh              # 所有示例
./scripts/smoke-all.sh --rust-only  # 仅 Rust 示例
./scripts/smoke-all.sh --python-only # 仅 Python 示例
```

**网络测试（17 个）：**

| 测试 | 示例 | 超时 |
|------|------|------|
| `smoke_rust_dataflow` | rust-dataflow/dataflow.yml | 30s |
| `smoke_rust_dataflow_dynamic` | rust-dataflow/dataflow_dynamic.yml | 30s |
| `smoke_rust_dataflow_socket` | rust-dataflow/dataflow_socket.yml | 30s |
| `smoke_rust_dataflow_url` | rust-dataflow-url/dataflow.yml | 30s |
| `smoke_benchmark` | benchmark/dataflow.yml | 30s |
| `smoke_log_sink_file` | log-sink-file/dataflow.yml | 30s |
| `smoke_log_sink_alert` | log-sink-alert/dataflow.yml | 30s |
| `smoke_log_sink_tcp` | log-sink-tcp/dataflow.yml | 30s |
| `smoke_python_dataflow` | python-dataflow/dataflow.yml | 30s |
| `smoke_python_async` | python-async/dataflow.yaml | 15s |
| `smoke_python_drain` | python-drain/dataflow.yaml | 15s |
| `smoke_python_log` | python-log/dataflow.yaml | 15s |
| `smoke_python_logging` | python-logging/dataflow.yml | 15s |
| `smoke_python_multiple_arrays` | python-multiple-arrays/dataflow.yml | 15s |
| `smoke_python_concurrent_rw` | python-concurrent-rw/dataflow.yml | 15s |
| `smoke_service_example` | service-example/dataflow.yml | 30s |
| `smoke_action_example` | action-example/dataflow.yml | 30s |

**本地测试（9 个）：**

| 测试 | 示例 | stop-after |
|------|------|------------|
| `smoke_local_python_dataflow` | python-dataflow/dataflow.yml | 30s |
| `smoke_local_python_async` | python-async/dataflow.yaml | 10s |
| `smoke_local_python_drain` | python-drain/dataflow.yaml | 10s |
| `smoke_local_python_log` | python-log/dataflow.yaml | 10s |
| `smoke_local_python_logging` | python-logging/dataflow.yml | 10s |
| `smoke_local_python_multiple_arrays` | python-multiple-arrays/dataflow.yml | 10s |
| `smoke_local_python_concurrent_rw` | python-concurrent-rw/dataflow.yml | 10s |
| `smoke_local_service_example` | service-example/dataflow.yml | 10s |
| `smoke_local_action_example` | action-example/dataflow.yml | 10s |

需要特殊依赖（摄像头、CUDA、ROS2、C/C++ 工具链、多机器部署）的示例不包含在冒烟测试中。

### 端到端测试（WebSocket CLI）

文件：`tests/ws-cli-e2e.rs`

两组：

**非忽略（快速）：** 启动进程内 coordinator 并直接测试 `WsSession`：
```bash
cargo test --test ws-cli-e2e
```
- `cli_list_empty` -- 空数据流列表
- `cli_status_no_daemon` -- daemon 连接性检查
- `cli_stop_nonexistent` -- 缺失数据流的错误
- `cli_multiple_requests_same_session` -- 会话复用

**忽略（完整栈）：** 使用 `adora up` 配合真实节点：
```bash
cargo test --test ws-cli-e2e -- --ignored --test-threads=1
```
- `e2e_start_list_stop` -- 启动、列出、停止生命周期
- `e2e_sequential_dataflows` -- 顺序两个数据流

### 容错测试

文件：`tests/fault-tolerance-e2e.rs`

这些测试使用 `Daemon::run_dataflow` 直接测试重启策略和输入超时（无需 CLI）。

```bash
cargo test --test fault-tolerance-e2e
```

测试：
- `restart_recovers_from_failure` -- 带 `restart_policy: on-failure` 的节点经受 panic（15 秒）
- `max_restarts_limit_reached` -- 节点耗尽 `max_restarts: 2` 预算（15 秒）
- `input_timeout_closes_stale_input` -- `input_timeout: 2.0s` 在上游停止时触发（10 秒）

这些测试的数据流 YAML 位于 `tests/dataflows/`。

### Coordinator 集成测试

文件：`binaries/coordinator/tests/ws_control_tests.rs`、`binaries/coordinator/tests/ws_daemon_tests.rs`

这些启动进程内 coordinator 并测试 WebSocket 控制/daemon 平面。

```bash
cargo test -p adora-coordinator
```

覆盖主题：健康检查、list/stop/destroy 请求、无效 JSON/参数、并发请求、ping/pong、daemon 注册、断开清理、错误净化（无内部链泄漏）、制品存储丢弃时清理。

## CI 管道

CI 在推送/PR 到 `main` 时运行。参见 `.github/workflows/ci.yml`。

```
fmt  ──────────────┐
clippy ────────────┤ (全部并行运行)
test ──────────────┤
typos ─────────────┘
                   │
              e2e (依赖 test)
```

| 作业 | 运行器 | 运行内容 |
|------|--------|---------|
| **fmt** | ubuntu-latest | `cargo fmt --all -- --check` |
| **clippy** | ubuntu-latest | `cargo clippy --all ... -- -D warnings` |
| **test** | ubuntu-latest | `cargo test --all ...`（排除 Python + adora-examples） |
| **e2e** | ubuntu-latest | example-tests、fault-tolerance、冒烟测试、WS E2E |
| **typos** | ubuntu-latest | `crate-ci/typos@master` |

`e2e` 作业仅在 `test` 通过后运行。所有其他作业并行运行。

## 编写新测试

### 单元测试

在被测代码同一文件中添加 `#[cfg(test)]` 模块：

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_valid_input() {
        let result = parse("valid");
        assert_eq!(result, expected);
    }
}
```

### 节点的集成测试

使用 `adora-node-api` 中的集成测试框架。三种方式：

**1. `setup_integration_testing`（推荐）**

在节点的 `main` 函数之前调用以注入输入并捕获输出：

```rust
#[test]
fn test_main_function() -> eyre::Result<()> {
    let events = vec![
        TimedIncomingEvent {
            time_offset_secs: 0.01,
            event: IncomingEvent::Input {
                id: "tick".into(),
                metadata: None,
                data: None,
            },
        },
        TimedIncomingEvent {
            time_offset_secs: 0.055,
            event: IncomingEvent::Stop,
        },
    ];
    let inputs = TestingInput::Input(
        IntegrationTestInput::new("node_id".parse().unwrap(), events),
    );
    let (tx, rx) = flume::unbounded();
    let outputs = TestingOutput::ToChannel(tx);
    let options = TestingOptions { skip_output_time_offsets: true };

    integration_testing::setup_integration_testing(inputs, outputs, options);
    crate::main()?;

    let outputs = rx.try_iter().collect::<Vec<_>>();
    assert_eq!(outputs, expected_outputs);
    Ok(())
}
```

**2. 环境变量模式**

直接测试编译的可执行文件，最接近生产行为：

```bash
ADORA_TEST_WITH_INPUTS=path/to/inputs.json \
ADORA_TEST_NO_OUTPUT_TIME_OFFSET=1 \
ADORA_TEST_WRITE_OUTPUTS_TO=/tmp/out.jsonl \
cargo run -p my-node
```

**3. `AdoraNode::init_testing`**

用于测试节点逻辑而不经过 `main`：

```rust
let (node, events) = AdoraNode::init_testing(inputs, outputs, Default::default())?;
```

### 生成测试输入文件

通过设置 `ADORA_WRITE_EVENTS_TO` 录制真实数据流事件：

```bash
ADORA_WRITE_EVENTS_TO=/tmp/recorded-events adora run examples/rust-dataflow/dataflow.yml
```

这会写入 `inputs-{node_id}.json` 文件，可直接用于 `ADORA_TEST_WITH_INPUTS`。

### 工作空间级集成测试

在 `tests/` 目录中添加新测试文件。对于需要完整 CLI 栈的测试，遵循 `tests/example-smoke.rs` 中的模式：

**网络模式**（演练 coordinator + daemon）：
1. 用 `Once` 守卫构建节点（避免每个测试重复构建）
2. 用 `adora down` 清理残留进程
3. 用 `adora up` 启动集群
4. 用 `adora start --detach` 运行数据流
5. 轮询 `adora list --json` 等待完成
6. 用 `adora stop --all` 和 `adora down` 清理

**本地模式**（单进程、进程内 coordinator）：
1. 用 `Once` 守卫构建 CLI
2. 运行 `adora run <yaml> --stop-after <duration>`
3. 断言退出码为成功

### 约定

- 使用 `assert2::assert!` 获取更好的错误消息（作为 dev-dependency 可用）
- 使用 `tempfile::NamedTempFile` 用于临时输出文件
- 需要独占端口访问的端到端测试应标记为 `#[ignore]` 并使用 `--test-threads=1` 运行
- 异步测试使用 `#[tokio::test(flavor = "multi_thread")]`
- 容错测试数据流放在 `tests/dataflows/`
- 示例输入/输出基线放在 `tests/sample-inputs/`

## 故障排除

### `cargo test` 编译 Python 包失败

始终排除 Python 包：
```bash
cargo test --all \
  --exclude adora-node-api-python \
  --exclude adora-operator-api-python \
  --exclude adora-ros2-bridge-python
```

### 冒烟/端到端测试因 "address already in use" 失败

残留的 coordinator 或 daemon 仍在运行。清理：
```bash
adora down
# 或手动终止进程：
pkill -f adora-coordinator
pkill -f adora-daemon
```

### 冒烟测试挂起或超时

- 如果你的机器较慢，增加测试中的超时（查找 `Duration::from_secs(...)`）
- 检查示例节点是否构建成功：
  ```bash
  cargo build -p rust-dataflow-example-node -p rust-dataflow-example-status-node \
    -p rust-dataflow-example-sink -p rust-dataflow-example-sink-dynamic
  cargo build -p log-sink-file -p log-sink-alert -p log-sink-tcp
  cargo build --release -p benchmark-example-node -p benchmark-example-sink
  ```
- 对于 Python 冒烟测试，确保 `pyarrow` 和 `numpy` 已安装

### 端到端测试并行运行时失败

冒烟和忽略的端到端测试必须单线程运行：
```bash
cargo test --test example-smoke -- --test-threads=1
cargo test --test ws-cli-e2e -- --ignored --test-threads=1
```

### 集成测试输出不匹配预期

1. 检查 `ADORA_TEST_NO_OUTPUT_TIME_OFFSET=1` 是否已设置（时间偏移因机器而异）
2. 如果节点行为有意更改，重新生成基线：
   ```bash
   ADORA_TEST_WITH_INPUTS=tests/sample-inputs/inputs-rust-node.json \
   ADORA_TEST_NO_OUTPUT_TIME_OFFSET=1 \
   ADORA_TEST_WRITE_OUTPUTS_TO=tests/sample-inputs/expected-outputs-rust-node.jsonl \
   cargo run -p rust-dataflow-example-node
   ```

### 拼写检查失败

typos 配置在 `_typos.toml` 中。添加误报排除：
```toml
[default.extend-identifiers]
MyCustomIdent = "MyCustomIdent"
```

### 本地通过但 CI 失败

- CI 在 Ubuntu 上运行；检查平台特定假设（路径、进程信号）
- CI 使用 `rust-cache`，依赖版本可能与本地 lockfile 不同
- 确保 `cargo fmt --all -- --check` 通过（CI 强制执行）
