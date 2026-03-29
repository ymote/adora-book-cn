> **[English](../../languages/c.md)**

# C API 参考

本文档涵盖 Adora 框架提供的两个 C API：用于独立 C 进程的**节点 API** 和用于由 Adora 运行时加载的共享库算子的**算子 API**。

---

## 节点 API（adora-node-api-c）

头文件：`apis/c/node/node_api.h`
Crate：`adora-node-api-c`（构建为 `staticlib`）

### 初始化

```c
void *init_adora_context_from_env();
void free_adora_context(void *adora_context);
```

从 daemon 设置的环境变量初始化 Adora 节点上下文。

### 事件循环

```c
void *adora_next_event(void *adora_context);
void free_adora_event(void *adora_event);
```

阻塞直到下一个事件可用。

### 事件检查

```c
enum AdoraEventType read_adora_event_type(void *adora_event);
void read_adora_input_id(void *adora_event, char **out_ptr, size_t *out_len);
void read_adora_input_data(void *adora_event, char **out_ptr, size_t *out_len);
```

### 输出

```c
int adora_send_output(
    void *adora_context,
    const char *id_ptr, size_t id_len,
    const char *data_ptr, size_t data_len
);
```

### 日志

```c
int adora_log(
    void *adora_context,
    const char *level_ptr, size_t level_len,
    const char *msg_ptr, size_t msg_len
);
```

有效日志级别：`"error"`、`"warn"`、`"info"`、`"debug"`、`"trace"`。

---

## 算子 API（adora-operator-api-c）

头文件：`apis/c/operator/operator_api.h`、`apis/c/operator/operator_types.h`

算子 API 由加载到 Adora 运行时进程中的共享库使用。它们导出三个函数：

```c
AdoraInitResult_t adora_init_operator(void);
AdoraResult_t adora_drop_operator(void *operator_context);
OnEventResult_t adora_on_event(RawEvent_t *event, const SendOutput_t *send_output, void *operator_context);
```

---

## 节点示例

```c
#include <stdio.h>
#include <string.h>
#include "node_api.h"

int main() {
    void *ctx = init_adora_context_from_env();
    if (ctx == NULL) {
        fprintf(stderr, "failed to init adora context\n");
        return 1;
    }

    for (int i = 0; i < 100; i++) {
        void *event = adora_next_event(ctx);
        if (event == NULL) break;

        enum AdoraEventType ty = read_adora_event_type(event);
        if (ty == AdoraEventType_Input) {
            char out_id[] = "message";
            char out_data[64];
            int out_len = snprintf(out_data, sizeof(out_data), "iteration %d", i);
            adora_send_output(ctx, out_id, strlen(out_id), out_data, out_len);
        } else if (ty == AdoraEventType_Stop) {
            free_adora_event(event);
            break;
        }
        free_adora_event(event);
    }

    free_adora_context(ctx);
    return 0;
}
```

## 构建与链接

### 节点（静态库）

```bash
cargo build -p adora-node-api-c --release
clang node.c -ladora_node_api_c -L ../../target/release -o build/c_node <FLAGS>
```

### 算子（共享库）

```bash
clang -c operator.c -o build/operator.o -fdeclspec -fPIC
clang -shared build/operator.o -o build/liboperator.so
```
