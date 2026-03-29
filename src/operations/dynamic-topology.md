> **[English](../../operations/dynamic-topology.md)**

# 动态拓扑

在运行中的数据流中添加和移除节点，无需重启。

## CLI 命令

```bash
# 从 YAML 定义添加节点
adora node add --from-yaml new-node.yml --dataflow my-app

# 移除节点（停止进程 + 清理映射）
adora node remove my-app filter-node

# 连接两个节点（添加实时映射）
adora node connect --dataflow my-app sender/value filter/input

# 断开两个节点（移除映射）
adora node disconnect --dataflow my-app sender/value filter/input
```

## 节点 YAML 定义

动态节点在独立的 YAML 文件中定义，格式与 `nodes:` 列表中的单个条目相同：

```yaml
# filter-node.yml
id: filter
path: filter.py
outputs:
  - output
```

添加后，使用 `adora node connect` 显式连接输入。

## 当前限制

- daemon 端 `AddNode` 的节点生成尚待完善
- 尚不支持跨 daemon 动态拓扑
- 动态节点不会在数据流重启后持久化
