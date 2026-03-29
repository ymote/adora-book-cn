> **[English](../../getting-started/installation.md)**

# 安装

## 从 crates.io 安装（推荐）

```bash
cargo install adora-cli           # CLI（adora 命令）
pip install adora-rs              # Python 节点/算子 API
```

## 从源码构建

```bash
git clone https://github.com/dora-rs/adora.git
cd adora
cargo build --release -p adora-cli
PATH=$PATH:$(pwd)/target/release

# Python API（需要 maturin >= 1.8：pip install maturin）
# 必须从包目录运行以正确解析依赖
cd apis/python/node && maturin develop --uv && cd ../../..
```

## 平台安装器

**macOS / Linux：**

```bash
curl --proto '=https' --tlsv1.2 -LsSf \
  https://github.com/dora-rs/adora/releases/latest/download/adora-cli-installer.sh | sh
```

**Windows：**

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://github.com/dora-rs/adora/releases/latest/download/adora-cli-installer.ps1 | iex"
```

## 构建特性

| 特性 | 描述 | 默认启用 |
|------|------|----------|
| `tracing` | OpenTelemetry 追踪支持 | 是 |
| `metrics` | OpenTelemetry 指标收集 | 否 |
| `python` | Python 算子支持（PyO3） | 否 |
| `redb-backend` | 持久化 coordinator 状态（redb） | 否 |
| `prometheus` | coordinator 上的 Prometheus `/metrics` 端点 | 否 |

```bash
cargo install adora-cli --features redb-backend
```

## 验证

```bash
adora --version
adora status
```
