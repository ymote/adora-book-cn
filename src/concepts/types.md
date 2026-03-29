> **[English](../../concepts/types.md)**

# 类型注解

数据流输入和输出的可选类型注解。类型永远不是必需的 -- 未注解的端口保持完全动态。类型检查在构建时和验证时运行（默认无运行时开销）。

## 快速入门

```yaml
nodes:
  - id: camera
    path: camera.py
    outputs:
      - image
    output_types:
      image: std/media/v1/Image

  - id: detector
    path: detect.py
    inputs:
      image: camera/image
    input_types:
      image: std/media/v1/Image
    outputs:
      - bbox
    output_types:
      bbox: std/vision/v1/BoundingBox
```

验证：

```bash
adora validate dataflow.yml

# 在警告时以非零退出码失败（用于 CI）
adora validate --strict-types dataflow.yml

# 类型检查也在构建期间运行
adora build dataflow.yml --strict-types
```

## 类型 URN 格式

类型 URN 遵循 `std/<category>/v<version>/<TypeName>` 模式：

```
std/core/v1/Float32
std/media/v1/Image
std/vision/v1/BoundingBox
```

### 参数化类型

某些结构类型接受参数以区分变体：

```
std/media/v1/AudioFrame[sample_type=f32]
std/media/v1/AudioFrame[sample_type=f32,channels=2]
```

匹配规则：
- 相同基类 + 相同参数 -> 兼容
- 相同基类 + 一方未参数化 -> 兼容（通配符）
- 相同基类 + 不同参数值 -> **不匹配**

## 标准类型库

### `std/core/v1`

| 类型 | Arrow 类型 | 描述 |
|------|-----------|------|
| `Float32` | Float32 | 32 位浮点数 |
| `Float64` | Float64 | 64 位浮点数 |
| `Int32` | Int32 | 32 位有符号整数 |
| `Int64` | Int64 | 64 位有符号整数 |
| `UInt8` | UInt8 | 8 位无符号整数 |
| `UInt32` | UInt32 | 32 位无符号整数 |
| `UInt64` | UInt64 | 64 位无符号整数 |
| `String` | Utf8 | UTF-8 字符串 |
| `Bytes` | LargeBinary | 原始字节（通用接收器 -- 任何类型都兼容） |
| `Bool` | Boolean | 布尔值 |

### `std/math/v1`

| 类型 | 描述 |
|------|------|
| `Vector3` | 三维向量（x, y, z） |
| `Quaternion` | 四元数 |
| `Pose` | 六自由度位姿 |
| `Transform` | 坐标变换 |

### `std/control/v1`

| 类型 | 描述 |
|------|------|
| `Twist` | 线速度和角速度 |
| `JointState` | 关节位置、速度、力矩 |
| `Odometry` | 参考坐标系中的位姿 + 速度 |

### `std/media/v1`

| 类型 | 参数 | 描述 |
|------|------|------|
| `Image` | `encoding` | 原始图像（宽、高、编码、数据） |
| `CompressedImage` | `format` | JPEG/PNG 压缩图像 |
| `PointCloud` | `point_type` | 三维点云 |
| `AudioFrame` | `sample_type` | 音频采样 |

### `std/vision/v1`

| 类型 | 描述 |
|------|------|
| `BoundingBox` | 带置信度和标签的二维边界框 |
| `Detection` | 目标检测结果（BoundingBox 列表） |
| `Segmentation` | 像素级分割掩码 |

## 验证规则

`adora validate` 和 `adora build` 检查：

1. **键存在性**：`output_types` 的键必须出现在 `outputs` 中
2. **URN 解析**：所有类型 URN 必须存在于标准或用户定义的类型库中。拼写错误会得到"你是否想输入？"建议
3. **边兼容性**：连接的边必须具有兼容的类型
4. **定时器自动类型**：定时器输入自动标记为 `std/core/v1/UInt64`
5. **类型推断**：当只有上游标注了类型时，下游输入会被推断

## 类型兼容性规则

除精确匹配外，类型检查器支持隐式加宽转换：

| 从 | 到 |
|----|-----|
| `UInt8` | `UInt32` |
| `UInt32` | `UInt64` |
| `Int32` | `Int64` |
| `Float32` | `Float64` |
| 任何类型 | `Bytes`（通用接收器） |

### 用户定义的兼容性规则

在数据流 YAML 中添加自定义规则：

```yaml
type_rules:
  - from: myproject/SensorV1
    to: myproject/SensorV2

nodes:
  # ...
```

## 运行时类型检查

除静态验证外，Adora 支持在 `send_output()` 时进行可选的运行时类型检查。通过环境变量启用：

```bash
# 不匹配时发出警告（记录并继续）
ADORA_RUNTIME_TYPE_CHECK=warn adora run dataflow.yml

# 不匹配时报错（节点返回错误）
ADORA_RUNTIME_TYPE_CHECK=error adora run dataflow.yml
```

## 用户定义类型

项目可以在数据流旁边的 `types/` 目录中定义自定义类型：

```yaml
types:
  MySensor:
    arrow: Struct
    description: 自定义传感器读数
    fields:
      - name: temperature
        type: Float32
      - name: humidity
        type: Float32
```

## 图形可视化

当输出有类型注解时，`adora graph` 在边标签上显示类型：

```bash
adora graph dataflow.yml --open
```
