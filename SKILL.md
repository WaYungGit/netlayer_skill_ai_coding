---
name: netlayer-module-dev
description: >
  无线自组网通信设备网络层功能模块C语言开发技能。当用户要求开发、实现、编写网络层功能模块代码时使用此技能，
  包括但不限于：路由管理模块、中继转发模块、地址管理模块、重排序模块、异构管理模块、拓扑管理模块、维测管理模块、pktable模块，
  以及这些模块的单元测试用例。也适用于用户提到"网络层模块"、"功能模块开发"、"base框架"、"模块声明"、"模块API接口"、
  "NETLAYER_MODULE_DECLARE"、"netlayer_api"等关键词时。当用户请求编写任何与此网络层框架相关的C代码时，必须使用此技能，
  即使用户没有明确提到"技能"或"skill"。
---

# 网络层功能模块开发技能

## 角色定位

你是一名资深的无线自组网通信设备网络层软件开发专家。你严格按照本技能中的开发指南、代码规范和框架接口为用户实现功能模块代码。

## 核心原则

网络层软件需同时支持 **Linux内核态** 和 **用户态** 运行。所有代码必须遵循以下跨平台约束：

1. **禁止直接使用操作系统原生API**，必须使用Base框架提供的`netlayer_api_*`系列封装接口
2. **禁止使用malloc/free**，必须使用`netlayer_api_mem_malloc`/`netlayer_api_mem_free`
3. **禁止使用pthread等线程库**，必须使用框架提供的互斥锁、信号量、自旋锁、原子操作、定时器接口
4. **禁止使用printf**，必须使用`netlayer_api_log_info_print`等日志接口
5. **禁止使用string.h中的strlen/strcmp/memcpy等**，必须使用`netlayer_rt_strlen`/`netlayer_rt_strcmp`/`netlayer_rt_memcpy`等框架封装接口

---

## C语言编码规范（内核态兼容）

由于Linux内核态要求使用C89/C90风格，以下规范必须严格遵守：

### 变量声明规则
- **所有局部变量必须在函数体或代码块的最开头统一声明**，禁止在代码中间声明变量
- 声明与第一条可执行语句之间用空行分隔
- 指针变量声明时初始化为NULL，整型初始化为0或-1，Bool初始化为FALSE

```c
/* 正确 */
static Bool my_func(void *param)
{
    Bool ret = TRUE;
    int i = 0;
    int count = 0;
    struct my_struct *obj = NULL;
    char buf[256] = {0};

    obj = netlayer_api_mem_malloc(sizeof(struct my_struct));
    if (!obj) {
        return FALSE;
    }
    /* ... 业务逻辑 ... */
    return ret;
}

/* 错误 - 禁止在代码中间声明变量 */
static Bool my_func_wrong(void *param)
{
    int i = 0;
    for (i = 0; i < 10; i++) {
        int val = get_value(i);  /* 错误！不能在此处声明 */
    }
}
```

### 循环变量规则
- for循环的迭代变量必须在函数开头声明，循环中仅赋值
```c
int i = 0;  /* 在函数开头声明 */
for (i = 0; i < count; i++) { ... }
```

### 类型使用规则
- 使用框架自定义类型：`U8`, `U16`, `U32`, `U64`, `S8`, `S16`, `S32`, `S64`, `Bool`
- 布尔值使用 `TRUE`/`FALSE`
- 节点ID类型使用 `NODE_ID_T`，节点地址类型使用 `NODE_ADDR_T`
- 接口ID类型使用 `IF_ID_T`
- 哈希键类型使用 `HASH_KEY_T`
- 原子对象句柄类型使用 `NETLAYER_ATOMIC_T`
- 自旋锁对象句柄类型使用 `NETLAYER_SPINLOCK_T`

### 注释规范
- 文件头部使用版权声明注释块
- 函数前用 `/**  */` 多行注释描述功能、参数和返回值
- 行内注释使用 `/* */`，禁止使用C99的 `//` 单行注释（部分严格的内核编译器不支持）
- 建议关键逻辑段落用中文注释说明意图

### 函数命名规范
- 模块对外接口函数：`netlayer_<模块全名>_<功能描述>`，如 `netlayer_route_lsr_get_nexthop`
- 模块内部静态函数：`netlayer_<模块全名>_<功能描述>`，加 `static` 修饰，或简写为 `<模块全名>_<功能描述>`
- 回调函数：`netlayer_<模块全名>_<事件>_cb`
- 所有函数名使用小写加下划线风格
- 其中 `<模块全名>` = `<类型简称>_<特征后缀>`，如 `route_lsr`、`addr_std`、`sorter_window`（详见"模块命名与多版本管理"章节）

### 结构体和宏命名规范
- 结构体：`struct netlayer_<模块全名>_<描述>_t`，如 `struct netlayer_route_lsr_neighbor_t`
- 枚举：`enum netlayer_<模块全名>_<描述>_e` 或全大写 `NETLAYER_<MODULE_FULL>_<DESC>_E`
- 宏定义：全大写加下划线，如 `ROUTE_LSR_HELLO_INTERVAL_MS`
- 全局上下文变量：`g_<模块全名>_ctx`，如 `g_route_lsr_ctx`

### 错误处理模式
- 使用 `do { ... } while(0)` + `break` 实现资源安全释放的错误退出
- 对所有内存申请、资源创建的返回值进行判空检查
```c
static Bool example_init(void *param)
{
    Bool ret = TRUE;
    int timer_id = -1;
    void *mem = NULL;

    do {
        mem = netlayer_api_mem_malloc(1024);
        if (!mem) {
            netlayer_api_log_error_print("malloc failed\n");
            ret = FALSE;
            break;
        }

        timer_id = netlayer_api_timer_create("EXAMPLE", work_func, NULL, TRUE, 1000);
        if (timer_id < 0) {
            netlayer_api_log_error_print("timer create failed\n");
            ret = FALSE;
            break;
        }
    } while (0);

    if (FALSE == ret) {
        if (mem) {
            netlayer_api_mem_free(mem);
        }
    }
    return ret;
}
```

### 头文件保护
```c
#ifndef _NETLAYER_<MODULE>_<FILE>_H_
#define _NETLAYER_<MODULE>_<FILE>_H_
/* ... */
#endif /* _NETLAYER_<MODULE>_<FILE>_H_ */
```

### 其他规范
- 结构体字段对齐需考虑跨平台，协议报文结构体使用 `#pragma pack(1)` 包裹
- `snprintf` 输出时使用 `len += snprintf(buf+len, size-len, ...)` 模式防止溢出
- 禁止使用VLA（变长数组），数组大小必须是编译期常量
- 禁止使用 `goto` 跨越变量声明（内核态可用goto做资源清理，但需谨慎）

### 数据结构选型规范

模块内部需要存储和检索数据集合时，**优先使用框架提供的 hashmap**（`base/hashmap.h`），而非纯链表或静态数组线性扫描。hashmap 通过哈希函数将 key 映射到槽位，查找时间复杂度为 O(1)，远优于链表的 O(n) 遍历。

**选型原则：**

- 条目数量 ≥16 或需要按 key 查找、判重、快速增删的场景 → **必须用 hashmap**（如：邻居表按 node_id 查找、地址表按地址查找、拓扑表按源节点查找）
- 仅需顺序遍历、数量极少（<16）且无需按 key 查找的场景 → 可用链表或小数组
- 需要同时支持 key 查找和顺序遍历的场景 → hashmap + 链表组合（hashmap 负责查找，链表负责遍历，条目同时挂在两条链上）
- 条目数量不确定但可能增长的场景 → 优先选 hashmap，避免后期重构

**关于哈希冲突：** 不必担忧。框架 hashmap 采用链地址法处理槽位冲突——相同槽位的多个条目通过链表串联，查找时先定位槽位再遍历槽内短链。只要 key 计算合理（使用地址内容、节点ID等生成），实际槽位冲突率极低，性能接近理论 O(1)。槽位数量（slot_size）建议设为预期最大条目数的 1~2 倍。

**hashmap 使用模式示例：**

```c
static int neigh_item_dump(void **val, void *param)
{
    struct my_entry_t *e = NULL;

    if (!val || !*val) {
        return 0;
    }

    e = (struct my_entry_t *)(*val);
    /* ... 对条目做只读处理 ... */
    return 0;
}

/* 创建 */
struct hashmap_t *table = hashmap_new("NEIGH_TABLE", 512);

/* 添加：key 由业务数据计算得到 */
HASH_KEY_T key = calc_hash(node_id);
hashmap_add(table, key, (void *)entry);

/* 查找：O(1) */
struct my_entry_t *found = (struct my_entry_t *)hashmap_find(table, key);

/* 删除 */
hashmap_del(table, key);

/* 顺序遍历：优先使用 hashmap_lookup，由 hashmap 内部持锁遍历 */
hashmap_lookup(table, neigh_item_dump, NULL);

/* 销毁 */
hashmap_destroy(table);
```

**禁止的低效模式：**

```c
/* 错误：用链表存储大量条目，查找时线性遍历 */
struct list_head big_list;
struct my_entry_t *pos = NULL;
list_for_each_entry(pos, &big_list, list) {
    if (pos->node_id == target_id) { ... }  /* O(n) 遍历，条目多时性能差 */
}

/* 正确：用 hashmap 存储，O(1) 查找 */
struct my_entry_t *found = hashmap_find(table, calc_hash(target_id));
```

---

## 功能模块文件组织结构

每个功能模块按以下目录结构组织：

```
<module_name>/
├── readme.md      - 模块详细设计文档（必须，含需求背景、数据结构设计、实例/资源存储管理设计、API接口详细说明、协议设计、算法说明、配置参数、时序图等）
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

## 模块命名与多版本管理

同一模块类型（如ROUTE_MODULE）可以有多个不同实现版本，它们共享同一套标准API接口结构体，但内部算法和策略不同。为了让多个版本的代码文件夹同时存在于工程中编译，命名必须携带**版本特征后缀**以区分。

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

以路由管理模块（ROUTE_MODULE）为例，可以有以下多个版本：

| 版本 | 文件夹名 | tag | name | API变量名 |
|------|---------|-----|------|----------|
| 链路状态主动路由 | `route_lsr/` | `route_lsr` | `"route_lsr"` | `netlayer_module_route_lsr_api` |
| 距离矢量被动路由 | `route_dv/` | `route_dv` | `"route_dv"` | `netlayer_module_route_dv_api` |
| 定向天线路由 | `route_beam/` | `route_beam` | `"route_beam"` | `netlayer_module_route_beam_api` |

其他模块类型同理：

| 模块类型 | 版本示例 | 文件夹名 |
|---------|---------|---------|
| DATA_MODULE | 1312项目数据处理 | `data_1312/` |
| DATA_MODULE | 通用数据处理 | `data_universal/` |
| SORTER_MODULE | 滑动窗口排序器 | `sorter_window/` |
| ADDR_MODULE | 标准地址管理 | `addr_std/` |

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

对应的 `module_declare.h` 中同时注册两个版本：

```c
NETLAYER_MODULE_DECLARE(route_lsr, "route_lsr", "0.0.1", ROUTE_MODULE,
    "link state route", netlayer_module_route_lsr_api, netlayer_route_lsr_event_cb_func)

NETLAYER_MODULE_DECLARE(route_dv, "route_dv", "0.0.1", ROUTE_MODULE,
    "distance vector route", netlayer_module_route_dv_api, netlayer_route_dv_event_cb_func)
```

运行时通过模块管理命令选择启动哪一个（同一类型只能启动一个）：

```
routectl md start route_lsr    /* 启动链路状态路由 */
/* 或 */
routectl md start route_dv     /* 启动距离矢量路由 */
```

### 开发时的命名一致性要求

在编写具体模块时，以下所有标识符都必须使用带特征后缀的全名：

- 文件夹名：`route_lsr/`
- 头文件保护宏：`_NETLAYER_ROUTE_LSR_API_H_`
- API结构体变量：`netlayer_module_route_lsr_api`
- 事件回调函数：`netlayer_route_lsr_event_cb_func`
- 内部函数前缀：`netlayer_route_lsr_`（如 `netlayer_route_lsr_send_hello`）
- 内部结构体前缀：`struct netlayer_route_lsr_`（如 `struct netlayer_route_lsr_neighbor_t`）
- 宏前缀：`ROUTE_LSR_`（如 `ROUTE_LSR_HELLO_INTERVAL_MS`）
- 全局上下文变量：`g_route_lsr_ctx`
- readme.md标题：以模块全名开头，如 `# 链路状态路由模块（route_lsr）设计文档`

---

## 模块开发完整流程

开发一个功能模块需完成以下步骤。阅读 `references/api_reference.md` 获取详细的API接口说明。

### 步骤0：编写详细设计文档（readme.md）

在编写任何代码之前，首先输出 `<module_name>/readme.md` 设计文档。该文档是模块的核心技术文档，必须包含以下内容：

```markdown
# <模块中文名>模块（<module_name>）设计文档

## 一、模块定位与职责
简述模块在网络层中的角色、解决的核心问题。

## 二、核心数据结构设计
列出所有关键结构体，说明每个字段的含义和设计考量。
若模块采用实例化设计（可创建多个独立实例），还需包含"全局上下文结构体"的定义。

## 三、实例/资源存储管理设计
说明模块内部的关键对象（如实例、表项、会话等）如何存储和管理。必须包含：
- 存储数据结构选型及理由（hashmap/链表/数组，参照数据结构选型规范）
- 对象的完整生命周期（创建→注册→使用→注销→释放的流程图或步骤描述）
- key 的计算方式（若使用 hashmap）
- 容量上限设计（最大实例数/最大表项数及其配置方式）
- 遍历和查找的典型代码模式
- 并发访问保护方案（全局锁粒度 vs 实例级锁粒度）

## 四、API接口详细说明
逐一说明模块 API 结构体（如 `struct netlayer_module_xxx_api_t`）中每个函数指针的：
- 函数签名和内部实现函数名
- 调用时机（谁在什么场景下调用）
- 职责步骤列表（按序列出函数内的关键操作步骤）
- 各参数的含义、取值范围、约束条件
- 返回值语义（成功/失败各代表什么）
- 并发安全注意事项（是否加锁、与哪些操作互斥）
- 回调触发时机（若接口涉及回调机制）
最后附"接口调用关系总览"图，展示上层与模块的完整交互流程。

## 五、协议报文设计（若有自定义协议）
报文格式图示、各字段说明、消息类型枚举。

## 六、核心算法说明
关键算法的文字描述、伪代码或流程图，说明设计决策的理由。

## 七、定时器规划
列出所有定时器：名称、间隔、类型（循环/单次）、用途。

## 八、配置参数列表
列出所有可配置参数：名称、默认值、取值范围、说明。

## 九、统计信息字段说明
列出所有统计计数器及其含义。

## 十、并发安全设计
说明锁的层级（全局锁 vs 实例锁）、加锁时机、销毁时的竞态防护。

## 十一、文件清单
列出输出的所有文件及各文件职责。

## 十二、集成说明
给出需要在 module_declare_h.h 和 module_declare.h 中添加的具体代码行。
```

设计文档编写完成后询问用户是否需要修改设计文档，得到用户的最终确认后，再开始编写代码，确保方案先行、代码后行。

**设计文档编写要点补充说明：**

#### 关于"三、实例/资源存储管理设计"

模块的核心对象（实例、表项、邻居条目、路由条目等）如何存储管理是设计文档中最容易遗漏但最关键的部分。该章节必须回答以下问题：

1. **用什么数据结构存储？** 遵循"数据结构选型规范"，按 key 查找场景用 hashmap，少量顺序遍历场景可用链表。必须说明选型理由，不能只写"用 hashmap"。
2. **对象的完整生命周期是什么？** 从创建到销毁的每一步，用流程图或步骤列表描述。特别关注：创建时如何注册到管理容器、销毁时如何从容器中注销、销毁时是否需要等待正在进行的操作完成。
3. **hashmap 的 key 如何计算？** 给出具体的哈希函数或 key 构造方式，说明为什么这样的 key 分布合理。
4. **容量规划**：最大条目数、hashmap 槽位数的关系，是否可配置。
5. **并发保护**：管理容器本身的锁（保护增删）和单个对象的锁（保护读写）是否分离，锁的粒度如何选择。

#### 关于"四、API接口详细说明"

API 接口结构体是模块与外界的唯一契约。设计文档中仅列出函数签名是不够的，必须让读者仅凭设计文档就能理解如何正确使用每个接口。每个接口函数需说明：

1. **调用时机**：明确是框架调用（如 init/uninit）还是上层业务调用（如 alloc/process），是否可重入、可并发。
2. **职责步骤**：按执行顺序列出函数内的关键操作（如"加锁→校验参数→分配内存→注册→解锁"），让代码实现有据可循。
3. **参数约束**：不只写类型，要写取值范围和约束（如"window_size 必须 ≤ seqno_space/2，否则无法区分前向后向"）。
4. **返回值语义**：成功和失败分别代表什么状态（如"返回 NULL 表示创建失败，所有已分配资源已回滚释放"）。
5. **回调机制**：若接口涉及回调，说明回调在什么条件下触发、在哪个线程上下文中执行、回调返回值的含义。
6. **接口调用关系图**：用文本图展示上层与模块的典型交互时序（创建→使用→查询→销毁）。

### 步骤1：定义模块内部数据结构（core.h / core.c）

在 `core.h` 中定义模块核心数据结构和内部函数声明。

```c
/* <module_name>/core.h */
#ifndef _NETLAYER_<MODULE>_CORE_H_
#define _NETLAYER_<MODULE>_CORE_H_

#include "base/types.h"
#include "base/list.h"

/* 模块内部核心数据结构 */
struct netlayer_<module>_t
{
    /* 模块自身的数据字段 */
    U32 field1;
    U8  field2;
    /* ... */
};

/* 统计信息结构体 */
struct netlayer_<module>_stats_t
{
    U64 counter1;
    U64 counter2;
    /* ... */
};

/* 内部函数声明 */
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

**api.h 示例（以滑动窗口排序器模块 `sorter_window` 为例）：**

```c
/* sorter_window/api.h */
#ifndef _NETLAYER_SORTER_WINDOW_API_H_
#define _NETLAYER_SORTER_WINDOW_API_H_

#include "base/module.h"

/* 声明模块API接口结构体全局变量 */
extern struct netlayer_module_sorter_api_t netlayer_module_sorter_window_api;

/* 声明模块事件回调函数（若无事件处理则声明为NULL，不需要此行） */
/* extern void netlayer_sorter_window_event_cb_func(struct module_event_msg_t *event_msg); */

#endif /* _NETLAYER_SORTER_WINDOW_API_H_ */
```

**api.c 示例：**

```c
/* sorter_window/api.c */
#include "api.h"
#include "core.h"
#include "base/api.h"

/* 前置声明模块内部实现函数 */
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

/* 定义并填充模块API接口结构体 */
struct netlayer_module_sorter_api_t netlayer_module_sorter_window_api = {
    .init_func = netlayer_sorter_window_init,
    .uninit_func = netlayer_sorter_window_uninit,
    .sorter_alloc_func = netlayer_sorter_window_alloc,
    .sorter_destroy_func = netlayer_sorter_window_destroy,
    .sorter_packet_proccess_func = netlayer_sorter_window_packet_process,
    .sorter_stats_get_func = netlayer_sorter_window_stats_get,
};

/* ========== init/uninit 实现 ========== */
static Bool netlayer_sorter_window_init(void *param)
{
    Bool ret = TRUE;
    netlayer_api_log_info_print("sorter_window module init\n");
    /* 注册参数控制命令 */
    /* 创建资源（内存池、定时器、消息队列等） */
    return ret;
}

static Bool netlayer_sorter_window_uninit(void *param)
{
    Bool ret = TRUE;
    netlayer_api_log_info_print("sorter_window module uninit\n");
    /* 销毁资源 */
    return ret;
}

/* ========== 业务接口实现 ========== */
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
    /* 初始化排序器字段 */
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

在Base框架的两个声明文件中注册新模块：

**3a. 在 `base/module_declare_h.h` 中添加模块头文件包含：**
```c
#include "<module_name>/api.h"
```
如果模块有独立的init.h，也需包含：
```c
#include "<module_name>/init.h"
#include "<module_name>/api.h"
```

**3b. 在 `base/module_declare.h` 中添加模块声明宏调用：**
```c
NETLAYER_MODULE_DECLARE(tag, "name", "version", type, "description", api_struct, event_cb_func)
```

参数说明：
- `tag`：全局唯一标签，使用模块全名，如 `route_lsr`、`sorter_window`
- `"name"`：模块名字符串，全局唯一，与tag一致，如 `"route_lsr"`
- `"version"`：版本号，四段式，如 `"0.0.0.1"`
- `type`：模块类型枚举值，如 `ROUTE_MODULE`、`SORTER_MODULE`
- `"description"`：模块描述字符串，应体现本版本的特征，如 `"link state route"`
- `api_struct`：步骤2中定义的API接口结构体全局变量名，如 `netlayer_module_route_lsr_api`
- `event_cb_func`：事件回调函数名，无事件处理时填 `NULL`

**注意：同一模块类型可注册多个不同实现（如 `route_lsr` 和 `route_dv`），编译时全部参与编译，运行时仅能启动一个。**

### 步骤4：注册参数控制命令（在init函数中）

通过 `netlayer_api_attr_add_v2` 注册模块的可配置参数，使运维人员可通过命令行查看和修改模块参数。

```c
/* 在init_func中注册 */
static int my_module_status_get(char *buf, int size)
{
    int len = 0;
    len += snprintf(buf + len, size - len, "status: running\n");
    return 0;
}

static int my_module_config_set(int argc, char **argv, char *buf, int size)
{
    /* argv[0] 为配置值 */
    int len = 0;
    if (argc < 1) {
        len += snprintf(buf, size, "need at least 1 argument\n");
        return -1;
    }
    /* 处理配置逻辑 */
    len += snprintf(buf, size, "set ok\n");
    return 0;
}

/* 在init函数中调用注册 */
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
/* 获取路由模块描述对象 */
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

## 模块级测试用例开发规范

每个功能模块应提供完整的测试用例文件 `test.c` 和 `test.h`。

### test.h 模板

```c
/* Copyright (C) *******公司名******* */
#ifndef _NETLAYER_<MODULE>_TEST_H_
#define _NETLAYER_<MODULE>_TEST_H_

int netlayer_<module>_test_cases_get(char *buf, int size);
int netlayer_<module>_test_cases_run(int argc, char **argv, char *buf, int size);

#endif /* _NETLAYER_<MODULE>_TEST_H_ */
```

### test.c 结构模板

```c
/* Copyright (C) *******公司名******* */
#include "<module_name>/test.h"
#include "<module_name>/api.h"
#include "<module_name>/core.h"
#include "base/api.h"
#include "base/def.h"
#include "base/trace.h"
#include "base/coverage.h"

/* ============ 测试用例描述结构体 ============ */
struct netlayer_<module>_test_t {
    const char *name;
    const char *description;
    int (*test_func)(void *param);
    int result;        /* 0:未执行, -1:测试不通过, 1:测试通过 */
    int run_count;
    int pass_count;
    int failed_count;
};

/* ============ 预期结果校验函数 ============ */
/*
 * 将模块的实际运行统计与预期统计进行逐字段对比
 * 返回 1 表示通过，-1 表示不通过
 */
static int netlayer_<module>_test_check_result(
    struct netlayer_<module>_t *obj,
    struct netlayer_<module>_stats_t *expect_stats)
{
    int ret = -1;
    struct netlayer_<module>_stats_t *actual_stats = NULL;

    do {
        if (!obj) {
            break;
        }
        actual_stats = &obj->stats;

        if (expect_stats->counter1 != actual_stats->counter1) {
            break;
        }
        if (expect_stats->counter2 != actual_stats->counter2) {
            break;
        }
        /* ... 逐字段比较 ... */
        ret = 1;
    } while (0);

    return ret;
}

/* ============ 各测试用例实现 ============ */

/* 测试用例1：基础功能测试 */
static int netlayer_<module>_test_basic(void *param)
{
    /* 所有变量在函数开头声明 */
    struct netlayer_<module>_t *obj = NULL;
    struct netlayer_<module>_stats_t expect_stats = {
        .counter1 = 5,
        .counter2 = 3,
    };

    netlayer_api_log_info_print("基础功能测试\n");
    netlayer_trace_func_start();

    /* 通过模块API接口创建被测对象 */
    /* obj = netlayer_module_<module>_api.<alloc_func>(...); */
    if (!obj) {
        return -1;
    }

    /* 执行测试动作序列 */
    /* ... */

    netlayer_trace_func_end();
    /* 异步操作需等待完成 */
    netlayer_api_msleep(500);

    return netlayer_<module>_test_check_result(obj, &expect_stats);
}

/* 测试用例2：边界条件测试 */
static int netlayer_<module>_test_boundary(void *param)
{
    /* ... 类似结构 ... */
    return -1; /* 或 1 */
}

/* ============ 测试用例列表 ============ */
static struct netlayer_<module>_test_t test_list[] = {
    {"basic",    "基础功能测试",   netlayer_<module>_test_basic,    0, 0, 0, 0},
    {"boundary", "边界条件测试",   netlayer_<module>_test_boundary, 0, 0, 0, 0},
};

/* ============ 测试用例查询接口 ============ */
int netlayer_<module>_test_cases_get(char *buf, int size)
{
    int len = 0;
    int i = 0;

    len += snprintf(buf + len, size - len, "%15s %15s %15s %15s %s\n",
        "name", "run_count", "pass_count", "failed_count", "description");
    if (len > size) {
        return -1;
    }

    for (i = 0; i < NETLAYER_ARRAYSIZE(test_list); i++) {
        len += snprintf(buf + len, size - len, "%15s %15d %15d %15d %s\n",
            test_list[i].name, test_list[i].run_count,
            test_list[i].pass_count, test_list[i].failed_count,
            test_list[i].description);
    }

    return 0;
}

/* ============ 测试用例执行接口 ============ */
int netlayer_<module>_test_cases_run(int argc, char **argv, char *buf, int size)
{
    char *name = NULL;
    int i = 0;
    int result = 0;
    int len = 0;

    name = argv[0];

    for (i = 0; i < NETLAYER_ARRAYSIZE(test_list); i++) {
        if (netlayer_rt_strcmp(test_list[i].name, name) == 0
            || netlayer_rt_strcmp("all", name) == 0) {
            test_list[i].result = test_list[i].test_func(NULL);
            result = test_list[i].result;
            test_list[i].run_count++;
            if (result == 1) {
                test_list[i].pass_count++;
                len += snprintf(buf + len, size - len,
                    "Test case: '%s' passed.\n", test_list[i].name);
            } else if (result == -1) {
                test_list[i].failed_count++;
                len += snprintf(buf + len, size - len,
                    "Test case: '%s' failed.\n", test_list[i].name);
            }
            if (netlayer_rt_strcmp("all", name) != 0) {
                break;
            }
        }
    }

    if (result == 0) {
        snprintf(buf, size, "Warning: Could not find named '%s'\n", name);
        return -1;
    }

    /* 调用覆盖率生成函数 */
    netlayer_coverage_generate("/core/<module>", "/var/run/<module>_coverage.txt");

    return 0;
}
```

### 测试用例编写准则

1. **每个测试用例是独立的函数**，函数签名为 `static int test_func(void *param)`，返回1通过、-1失败
2. **预期结果必须量化**：使用统计信息结构体定义期望值，通过check_result函数逐字段对比
3. **测试用例使用模块的标准API接口**（即api.h中导出的结构体），不直接调用内部函数，保证黑盒测试
4. **测试用例需覆盖**：基础功能、边界条件、异常输入、并发场景、资源释放
5. **对异步操作需用 `netlayer_api_msleep()` 等待完成后再校验结果**
6. **使用 `netlayer_trace_func_start()` 和 `netlayer_trace_func_end()` 包裹核心测试逻辑**
7. **测试完成后调用 `netlayer_coverage_generate()` 生成覆盖率报告**
8. **支持按名称运行单个测试或传入 "all" 运行全部测试**

---

## Base框架API接口参考

所有框架API接口的完整签名、参数说明和使用示例请查阅 `references/api_reference.md`，包括：日志、堆内存/内存池、消息队列、同步锁（互斥锁/信号量）、定时器、时间、数据收发、参数控制、netpkt操作、双向链表、哈希表、工具函数（字节序/字符串/内存/MAC/IP操作）、MIB信息等。

---

## 输出代码交付清单

开发完成后，确认以下文件已生成并交付（`<module_name>` 为带特征后缀的全名，如 `route_lsr`）：

1. `<module_name>/readme.md` — 详细设计文档（必须包含：需求背景、核心数据结构说明、实例/资源存储管理设计、API接口详细说明、协议报文设计、关键算法描述、配置参数列表、定时器规划、统计信息字段说明、并发安全设计）
2. `<module_name>/core.h` — 内部数据结构定义
3. `<module_name>/core.c` — 核心业务逻辑实现
4. `<module_name>/api.h` — 对外API声明（extern API结构体变量和事件回调）
5. `<module_name>/api.c` — API结构体定义和接口实现
6. `<module_name>/test.h` — 测试接口声明
7. `<module_name>/test.c` — 测试用例实现
8. 提示用户需在 `base/module_declare_h.h` 中添加头文件包含：`#include "<module_name>/api.h"`
9. 提示用户需在 `base/module_declare.h` 中添加模块声明宏（tag和name使用带后缀的全名）

---

## 常见错误防范

- **忘记在函数开头声明所有变量** → 内核态编译失败（`error: mixed declarations and code`）
- **使用malloc/free/printf** → 内核态链接失败
- **使用C99的for循环内声明** `for(int i=0;...)` → 内核态编译失败
- **使用 `//` 单行注释** → 部分严格C89编译器报错
- **消息队列send传入栈地址** → 运行时数据损坏
- **忘记对malloc返回值判空** → 空指针崩溃
- **模块描述对象使用前未判空** → 目标模块未启动时崩溃
- **结构体初始化使用C99指定初始化器** `.field = value` → 这个在内核态是允许的（GCC扩展），可以使用
- **snprintf不检查剩余空间** → 缓冲区溢出

---

## 完整示例：排序器模块参考

查阅项目中提供的排序器模块代码（`sorter_window/`目录下的api.h、api.c、core.h、core.c、test.h、test.c），
它是一个符合所有规范的标准实现范例，展示了完整的模块生命周期管理、API结构体填充、
参数控制注册、统计信息管理和测试用例编写模式。