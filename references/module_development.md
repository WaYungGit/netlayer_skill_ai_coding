# 功能模块开发指南

## 一、文件组织结构

每个功能模块按以下目录结构组织：

```
<module_name>/
├── readme.md      - 模块详细设计文档（必须）
├── api.h          - 模块对外API接口声明（含API接口结构体全局变量extern声明和事件回调函数extern声明）
├── api.c          - 模块API接口结构体全局变量定义与函数实现
├── core.h         - 模块内部核心数据结构和函数声明
├── core.c         - 模块核心业务逻辑实现
├── init.h         - 模块init/uninit函数声明（若模块仅有api.h导出则可省略）
├── init.c         - 模块初始化和去初始化函数实现（若逻辑简单可合并到api.c）
├── test.h         - 模块测试用例接口声明
└── test.c         - 模块测试用例实现
```

---

## 二、模块命名与多版本管理

同一模块类型（如ROUTE_MODULE）可以有多个不同实现版本，共享同一套标准API接口结构体，但内部算法和策略不同。命名必须携带**版本特征后缀**以区分。

### 命名规则

模块文件夹名、tag、name、API结构体变量名、函数前缀均使用统一的 `<类型简称>_<特征后缀>` 格式：

```
文件夹名:     <类型简称>_<特征后缀>/
tag:         <类型简称>_<特征后缀>
name:        "<类型简称>_<特征后缀>"
API变量名:    netlayer_module_<类型简称>_<特征后缀>_api
函数前缀:     netlayer_<类型简称>_<特征后缀>_<功能描述>
```

### 命名示例

以路由管理模块（ROUTE_MODULE）为例：

| 版本 | 文件夹名 | tag | API变量名 |
|------|---------|-----|----------|
| 链路状态主动路由 | `route_lsr/` | `route_lsr` | `netlayer_module_route_lsr_api` |
| 距离矢量被动路由 | `route_dv/` | `route_dv` | `netlayer_module_route_dv_api` |
| 定向天线路由 | `route_beam/` | `route_beam` | `netlayer_module_route_beam_api` |

其他模块类型同理：

| 模块类型 | 版本示例 | 文件夹名 |
|---------|---------|---------|
| DATA_MODULE | 1312项目数据处理 | `data_1312/` |
| DATA_MODULE | 通用数据处理 | `data_universal/` |
| SORTER_MODULE | 滑动窗口排序器 | `sorter_window/` |
| ADDR_MODULE | 标准地址管理 | `addr_std/` |

### 命名一致性要求

编写模块时，以下标识符都必须使用带特征后缀的全名：

- 文件夹名：`route_lsr/`
- 头文件保护宏：`_NETLAYER_ROUTE_LSR_API_H_`
- API结构体变量：`netlayer_module_route_lsr_api`
- 事件回调函数：`netlayer_route_lsr_event_cb_func`
- 内部函数前缀：`netlayer_route_lsr_`
- 内部结构体前缀：`struct netlayer_route_lsr_`
- 宏前缀：`ROUTE_LSR_`
- 全局上下文变量：`g_route_lsr_ctx`
- readme.md标题：以模块全名开头，如 `# 链路状态路由模块（route_lsr）设计文档`

### 多版本共存目录结构示例

```
netlayer/src/core/
├── base/
│   ├── module_declare.h    (同时声明route_lsr和route_dv两个路由模块)
│   └── module_declare_h.h  (同时include两者的api.h)
├── route_lsr/              (版本A：链路状态路由)
│   ├── readme.md
│   ├── api.h / api.c
│   ├── core.h / core.c
│   └── test.h / test.c
├── route_dv/               (版本B：距离矢量路由)
│   ├── readme.md
│   ├── api.h / api.c
│   ├── core.h / core.c
│   └── test.h / test.c
└── ...
```

---

## 三、模块开发完整流程

阅读 `api_reference.md` 获取详细的API接口说明。

### 步骤0：编写详细设计文档（readme.md）

在编写任何代码之前，首先输出 `<module_name>/readme.md` 设计文档。设计文档编写完成后询问用户是否需要修改，得到确认后再开始编写代码，确保方案先行、代码后行。

设计文档必须包含以下章节：

```markdown
# <模块中文名>模块（<module_name>）设计文档

## 一、模块定位与职责
简述模块在网络层中的角色、解决的核心问题。

## 二、核心数据结构设计
列出所有关键结构体，说明每个字段的含义和设计考量。
若模块采用实例化设计，还需包含"全局上下文结构体"的定义。

## 三、实例/资源存储管理设计
说明模块内部关键对象的存储和管理方式，必须包含：
- 存储数据结构选型及理由（参照 data_structure_guide.md）
- 对象的完整生命周期（创建→注册→使用→注销→释放）
- key 的计算方式（若使用 hashmap）
- 容量上限设计
- 遍历和查找的典型代码模式
- 并发访问保护方案

## 四、API接口详细说明
逐一说明模块API结构体中每个函数指针的：
- 函数签名和内部实现函数名
- 调用时机（谁在什么场景下调用）
- 职责步骤列表
- 各参数的含义、取值范围、约束条件
- 返回值语义
- 并发安全注意事项
- 回调触发时机（若涉及回调）
最后附"接口调用关系总览"图。

## 五、协议报文设计（若有自定义协议）
报文格式图示、各字段说明、消息类型枚举。

## 六、核心算法说明
关键算法的文字描述、伪代码或流程图。

## 七、定时器规划
列出所有定时器：名称、间隔、类型（循环/单次）、用途。

## 八、配置参数列表
列出所有可配置参数：名称、默认值、取值范围、说明。

## 九、统计信息字段说明
列出所有统计计数器及其含义。

## 十、并发安全设计
说明锁的层级、加锁时机、销毁时的竞态防护。

## 十一、文件清单
列出输出的所有文件及各文件职责。

## 十二、集成说明
给出需要在 module_declare_h.h 和 module_declare.h 中添加的具体代码行。
```

#### 关于"三、实例/资源存储管理设计"补充

该章节必须回答以下问题：

1. **用什么数据结构存储？** 按 key 查找场景用 hashmap，少量顺序遍历场景可用链表。必须说明选型理由。
2. **对象的完整生命周期是什么？** 从创建到销毁的每一步，特别关注：创建时如何注册到管理容器、销毁时如何从容器中注销、销毁时是否需要等待正在进行的操作完成。
3. **hashmap 的 key 如何计算？** 给出具体的哈希函数或 key 构造方式。
4. **容量规划**：最大条目数、hashmap 槽位数的关系，是否可配置。
5. **并发保护**：管理容器本身的锁和单个对象的锁是否分离，锁的粒度如何选择。

#### 关于"四、API接口详细说明"补充

每个接口函数需说明：

1. **调用时机**：框架调用还是上层业务调用，是否可重入、可并发。
2. **职责步骤**：按执行顺序列出关键操作（如"加锁→校验参数→分配内存→注册→解锁"）。
3. **参数约束**：不只写类型，要写取值范围和约束。
4. **返回值语义**：成功和失败分别代表什么状态。
5. **回调机制**：回调在什么条件下触发、在哪个线程上下文中执行、返回值含义。
6. **接口调用关系图**：展示上层与模块的典型交互时序。

### 步骤1：定义模块内部数据结构（core.h / core.c）

```c
/* <module_name>/core.h */
#ifndef _NETLAYER_<MODULE>_CORE_H_
#define _NETLAYER_<MODULE>_CORE_H_

#include "base/types.h"
#include "base/list.h"

struct netlayer_<module>_t
{
    U32 field1;
    U8  field2;
};

struct netlayer_<module>_stats_t
{
    U64 counter1;
    U64 counter2;
};

Bool netlayer_<module>_core_init(void);
void netlayer_<module>_core_uninit(void);

#endif /* _NETLAYER_<MODULE>_CORE_H_ */
```

### 步骤2：实现模块API接口结构体（api.h / api.c）

`api.h` 对外暴露模块的API接口结构体全局变量和事件回调函数。`api.c` 定义并填充该结构体。

模块API接口结构体的类型由模块类型决定（定义在 `base/module.h` 中）：

| 模块类型 | API结构体类型 | 枚举值 |
|---------|-------------|-------|
| 路由管理 | `struct netlayer_module_route_api_t` | `ROUTE_MODULE` |
| 中继转发 | `struct netlayer_module_data_api_t` | `DATA_MODULE` |
| 地址管理 | `struct netlayer_module_addr_api_t` | `ADDR_MODULE` |
| 重排序   | `struct netlayer_module_sorter_api_t` | `SORTER_MODULE` |
| 异构管理 | `struct netlayer_module_hetnet_api_t` | `HETNET_MODULE` |
| 维测管理 | `struct netlayer_module_kpi_api_t` | `KPI_MODULE` |
| pktable | `struct netlayer_module_pktable_api_t` | `PKTABLE_MODULE` |
| 拓扑管理 | `struct netlayer_module_topo_api_t` | `TOPO_MODULE` |

**api.h 示例（以 `sorter_window` 为例）：**

```c
/* sorter_window/api.h */
#ifndef _NETLAYER_SORTER_WINDOW_API_H_
#define _NETLAYER_SORTER_WINDOW_API_H_

#include "base/module.h"

extern struct netlayer_module_sorter_api_t netlayer_module_sorter_window_api;

#endif /* _NETLAYER_SORTER_WINDOW_API_H_ */
```

**api.c 示例：**

```c
/* sorter_window/api.c */
#include "api.h"
#include "core.h"
#include "base/api.h"

static Bool netlayer_sorter_window_init(void *param);
static Bool netlayer_sorter_window_uninit(void *param);
static void *netlayer_sorter_window_alloc(const char *name, U32 timeout_detect_ms,
    U32 timeout_ms, U32 window_size, U32 max_seqno_num);
static Bool netlayer_sorter_window_destroy(void *sorter);
static Bool netlayer_sorter_window_packet_process(void *sorter, U32 seqno_num,
    Bool (*cb)(U32 seqno_num, void *param), void *param);
static Bool netlayer_sorter_window_stats_get(void *sorter, U64 *total_recv,
    U64 *total_sorted, U64 *timeout, U64 *out_of_order,
    U64 *duplicate, U64 *expired, U64 *big_detected, U64 *restart_detected);

struct netlayer_module_sorter_api_t netlayer_module_sorter_window_api = {
    .init_func = netlayer_sorter_window_init,
    .uninit_func = netlayer_sorter_window_uninit,
    .sorter_alloc_func = netlayer_sorter_window_alloc,
    .sorter_destroy_func = netlayer_sorter_window_destroy,
    .sorter_packet_proccess_func = netlayer_sorter_window_packet_process,
    .sorter_stats_get_func = netlayer_sorter_window_stats_get,
};

static Bool netlayer_sorter_window_init(void *param)
{
    Bool ret = TRUE;
    netlayer_api_log_info_print("sorter_window module init\n");
    return ret;
}

static Bool netlayer_sorter_window_uninit(void *param)
{
    Bool ret = TRUE;
    netlayer_api_log_info_print("sorter_window module uninit\n");
    return ret;
}

static void *netlayer_sorter_window_alloc(const char *name, U32 timeout_detect_ms,
    U32 timeout_ms, U32 window_size, U32 max_seqno_num)
{
    struct netlayer_sorter_window_t *sorter = NULL;

    sorter = netlayer_api_mem_malloc(sizeof(struct netlayer_sorter_window_t));
    if (!sorter) {
        netlayer_api_log_error_print("malloc sorter failed\n");
        return NULL;
    }
    netlayer_rt_memset(sorter, 0, sizeof(struct netlayer_sorter_window_t));
    return (void *)sorter;
}

static Bool netlayer_sorter_window_destroy(void *sorter)
{
    if (!sorter) {
        return FALSE;
    }
    netlayer_api_mem_free(sorter);
    return TRUE;
}

/* ... 其余接口实现 ... */
```

### 步骤3：注册模块声明（module_declare.h / module_declare_h.h）

**3a. 在 `base/module_declare_h.h` 中添加头文件包含：**
```c
#include "<module_name>/api.h"
```

**3b. 在 `base/module_declare.h` 中添加模块声明宏调用：**
```c
NETLAYER_MODULE_DECLARE(tag, "name", "version", type, "description", api_struct, event_cb_func)
```

参数说明：
- `tag`：全局唯一标签，使用模块全名，如 `route_lsr`
- `"name"`：模块名字符串，与tag一致
- `"version"`：版本号，四段式，如 `"0.0.0.1"`
- `type`：模块类型枚举值，如 `ROUTE_MODULE`
- `"description"`：模块描述，应体现本版本特征
- `api_struct`：API接口结构体全局变量名
- `event_cb_func`：事件回调函数名，无事件处理时填 `NULL`

**注意：同一模块类型可注册多个不同实现，编译时全部参与编译，运行时仅能启动一个。**

对应的 `module_declare.h` 示例：

```c
NETLAYER_MODULE_DECLARE(route_lsr, "route_lsr", "0.0.1", ROUTE_MODULE,
    "link state route", netlayer_module_route_lsr_api, netlayer_route_lsr_event_cb_func)

NETLAYER_MODULE_DECLARE(route_dv, "route_dv", "0.0.1", ROUTE_MODULE,
    "distance vector route", netlayer_module_route_dv_api, netlayer_route_dv_event_cb_func)
```

运行时选择启动：
```
routectl md start route_lsr    /* 启动链路状态路由 */
routectl md start route_dv     /* 启动距离矢量路由 */
```

### 步骤4：注册参数控制命令（在init函数中）

通过 `netlayer_api_attr_add_v2` 注册模块的可配置参数：

```c
static int my_module_status_get(char *buf, int size)
{
    int len = 0;

    len += snprintf(buf + len, size - len, "status: running\n");
    return 0;
}

static int my_module_config_set(int argc, char **argv, char *buf, int size)
{
    int len = 0;

    if (argc < 1) {
        len += snprintf(buf, size, "need at least 1 argument\n");
        return -1;
    }
    len += snprintf(buf, size, "set ok\n");
    return 0;
}

/* 在init函数中调用 */
netlayer_api_attr_add_v2("my_module_status", "ms", "[show]",
    my_module_status_get, my_module_config_set,
    "Get and configure my module status",
    "获取及配置我的模块状态信息");
```

### 步骤5：实现事件通知回调（可选）

当模块需要感知其他模块的启停事件时，实现事件回调函数并在模块声明时传入：

```c
void netlayer_mymodule_event_cb_func(struct module_event_msg_t *event_msg)
{
    if (!event_msg) {
        return;
    }

    if (event_msg->event_type == MODULE_EVENT_STARTED) {
        netlayer_api_log_info_print("module %s started, type=%d\n",
            event_msg->src_module_name, event_msg->src_module_type);

        if (event_msg->src_module_type == ROUTE_MODULE) {
            /* 路由模块已启动，可获取路由描述对象并使用其接口 */
        }
    }
    else if (event_msg->event_type == MODULE_EVENT_STOPPED) {
        netlayer_api_log_info_print("module %s stopped\n",
            event_msg->src_module_name);
    }
}
```

### 步骤6：跨模块调用（通过模块描述对象）

模块之间无编译依赖，通过运行时获取目标模块描述对象来调用其接口：

```c
struct netlayer_module_desc *route_desc = NULL;
NODE_ID_T next_hop = 0;
Bool found = FALSE;

route_desc = netlayer_module_get_by_type(ROUTE_MODULE);
if (route_desc && route_desc->route_api && route_desc->route_api->get_nexthop_by_dstid_func) {
    found = route_desc->route_api->get_nexthop_by_dstid_func(dst_node_id, &next_hop, NULL);
    if (found) {
        netlayer_api_log_info_print("next hop for node %d is %d\n", dst_node_id, next_hop);
    }
}
```

---

## 四、输出代码交付清单

开发完成后，确认以下文件已生成并交付：

1. `<module_name>/readme.md` — 详细设计文档
2. `<module_name>/core.h` — 内部数据结构定义
3. `<module_name>/core.c` — 核心业务逻辑实现
4. `<module_name>/api.h` — 对外API声明
5. `<module_name>/api.c` — API结构体定义和接口实现
6. `<module_name>/test.h` — 测试接口声明
7. `<module_name>/test.c` — 测试用例实现
8. 提示用户需在 `base/module_declare_h.h` 中添加：`#include "<module_name>/api.h"`
9. 提示用户需在 `base/module_declare.h` 中添加模块声明宏

---

## 五、常见错误防范

| 错误 | 后果 |
|------|------|
| 忘记在函数开头声明所有变量 | 内核态编译失败（`error: mixed declarations and code`） |
| 使用malloc/free/printf | 内核态链接失败 |
| 使用C99的 `for(int i=0;...)` | 内核态编译失败 |
| 使用 `//` 单行注释 | 部分严格C89编译器报错 |
| 消息队列send传入栈地址 | 运行时数据损坏 |
| 忘记对malloc返回值判空 | 空指针崩溃 |
| 模块描述对象使用前未判空 | 目标模块未启动时崩溃 |
| snprintf不检查剩余空间 | 缓冲区溢出 |

**注意：** 结构体初始化使用C99指定初始化器 `.field = value` 在内核态是允许的（GCC扩展），可以使用。
