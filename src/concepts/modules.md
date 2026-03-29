> **[English](../../concepts/modules.md)**

# 模块（可复用子数据流）

模块允许您在单独的 YAML 文件中定义可复用的节点子图，并将它们组合到更大的数据流中。模块在编译时展开 -- 运行时不会看到它们。

## 快速入门

**模块文件**（`modules/transform_module.yml`）：

```yaml
module:
  name: transform_pipeline
  inputs: [raw_data]
  outputs: [filtered]

nodes:
  - id: doubler
    path: doubler.py
    inputs:
      data: _mod/raw_data
    outputs:
      - doubled

  - id: filter
    path: filter_even.py
    inputs:
      data: doubler/doubled
    outputs:
      - filtered
```

**数据流文件**（`dataflow.yml`）：

```yaml
nodes:
  - id: sender
    path: sender.py
    outputs:
      - value

  - id: pipeline
    module: modules/transform_module.yml
    inputs:
      raw_data: sender/value

  - id: receiver
    path: receiver.py
    inputs:
      filtered: pipeline/filtered
```

展开后，`pipeline` 变成两个节点：`pipeline.doubler` 和 `pipeline.filter`，所有接线自动解析。

## 模块定义文件

模块文件有两个部分：

### `module:` 头部

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `name` | string | 是 | 模块名称（仅元数据） |
| `inputs` | 列表 | 否 | 必需的输入端口名称 |
| `inputs_optional` | 列表 | 否 | 可选输入端口（未连接时静默跳过） |
| `outputs` | 列表 | 否 | 暴露给父数据流的输出端口名称 |

### `nodes:` 列表

标准节点定义，有一个特殊语法：**`_mod/port_name`** 引用模块输入端口。展开时，`_mod/port_name` 会被替换为父级连接到该端口的实际源。

## 参数

通过 `params:` 向模块传递配置值：

```yaml
- id: fast_pipeline
  module: modules/transform_module.yml
  inputs:
    raw_data: sender/value
  params:
    speed: "2.0"
    mode: turbo
```

在模块内部，参数在 `args:` 中以 `$PARAM_<UPPERCASE_KEY>` 的形式引用，并作为环境变量注入到模块内的每个节点。

## 展开规则

1. 加载模块 YAML 文件并验证其头部
2. 为所有内部节点 ID 添加 `{module_id}.` 前缀
3. 将 `_mod/port_name` 引用替换为父级输入映射中的实际源
4. 重写内部交叉引用
5. 将模块声明的输出映射到内部节点输出
6. 用展开的扁平节点替换模块节点
7. 在 `args:` 字段中替换 `params:` 值并注入为环境变量

使用 `adora expand` 查看结果：

```bash
adora expand dataflow.yml
```

## 嵌套模块

模块可以引用其他模块。展开是递归的，深度限制为 8 层。

## 可选输入

当模块应该在有或没有某些连接的情况下都能工作时，将输入声明为可选：

```yaml
module:
  name: flexible_processor
  inputs: [data]
  inputs_optional: [config]
  outputs: [result]
```

## 安全性

- **路径限制**：模块文件路径必须解析在数据流的基目录内
- **文件大小限制**：模块文件上限 1 MB
- **深度限制**：递归嵌套上限 8 层
- **参数键验证**：参数键必须仅包含字母数字和下划线
