# 网络层Base框架API接口完整参考

本文档包含所有Base框架对外提供的API接口详细说明，是功能模块开发的权威接口参考。

## 如何查阅

- 需要确认函数签名、头文件包含、同类 API 有哪些时，优先查本文档
- 需要确认功能模块设计、数据结构选型、开发约束时，查 `../SKILL.md` 及其引用的子文档
- 编写 hashmap 增删改查逻辑时，查阅 `data_structure_guide.md` 获取完整的行为说明和推荐写法

---

## 1. 头文件包含约定

模块代码中统一使用 `base/` 前缀包含框架头文件：
```c
#include "base/api.h"        /* 统一API入口，包含以下所有子头文件 */
#include "base/module.h"     /* 模块管理、模块类型定义、API结构体定义 */
#include "base/netpkt.h"     /* netpkt数据包结构和操作 */
#include "base/mib.h"        /* 网络层基础管理信息库 */
#include "base/def.h"        /* 工具函数、字节序转换、字符串/内存操作 */
#include "base/list.h"       /* 双向链表 */
#include "base/hashmap.h"    /* 哈希表 */
#include "base/types.h"      /* 基础类型定义 */
#include "base/trace.h"      /* 追踪桩函数 */
#include "base/coverage.h"   /* 覆盖率生成 */
```

---

## 2. 基础类型定义（base/types.h）

```c
typedef unsigned char       U8;
typedef signed char         S8;
typedef unsigned short      U16;
typedef signed short        S16;
typedef unsigned int        U32;
typedef signed int          S32;
typedef unsigned long long  U64;
typedef signed long long    S64;
typedef unsigned char       Bool;

#define FALSE 0
#define TRUE  1

typedef unsigned int  HASH_KEY_T;    /* 哈希键类型 */
typedef U32           IF_ID_T;       /* 接口ID类型 */
typedef void         *NETLAYER_ATOMIC_T; /* 原子对象句柄 */
typedef void         *NETLAYER_SPINLOCK_T; /* 自旋锁句柄 */
typedef unsigned char NODE_ID_T;     /* 节点网络ID（入网后分配） */
typedef unsigned short NODE_ADDR_T;  /* 节点地址（固定不变） */
typedef unsigned short CLIENT_ADDR_ID_T; /* 客户端地址ID */

#define MAX_NODE_ID ((NODE_ID_T)((1ULL << (NODE_ID_LEN * 8)) - 1))
```

---

## 3. 网络层基础管理信息库（base/mib.h）

```c
#define BYNAME_MAX_LEN 255
#define ETH_ALEN 6

struct netlayer_mib
{
    NODE_ID_T node_id;                 /* 本节点ID */
    NODE_ADDR_T node_addr;             /* 本节点地址 */
    U16 dev_type;                      /* 设备类型 */
    Bool run_switch;                   /* 网络层启动开关 */
    char byname[BYNAME_MAX_LEN];      /* 节点别名 */
    U8 macaddr[ETH_ALEN];             /* MAC地址 */
};

extern struct netlayer_mib global_mib;  /* 全局MIB实例 */
```

使用示例：
```c
NODE_ADDR_T my_addr = global_mib.node_addr;
NODE_ID_T my_id = global_mib.node_id;
```

---

## 4. 日志接口集（base/api.h）

### 4.1 调试日志模块注册
```c
Bool netlayer_api_log_debug_add_module(
    const char *module_name,   /* 模块名称 */
    Bool print_flag,           /* 初始打印开关，默认FALSE */
    const char *description,   /* 描述说明 */
    U8 *dbg_module_id          /* [out] 返回的日志模块ID */
);
/* 返回：TRUE成功，FALSE失败 */
/* 示例：netlayer_api_log_debug_add_module("route", FALSE, "route debug", &module_id); */
```

### 4.2 日志打印宏
```c
/* 信息日志（始终打印） */
netlayer_api_log_info_print(fmt, args...)

/* 调试日志（受开关控制） */
netlayer_api_log_debug_print(module_id, fmt, args...)

/* 错误日志（始终打印） */
netlayer_api_log_error_print(fmt, args...)
```

所有日志宏自动附加函数名前缀。使用示例：
```c
netlayer_api_log_info_print("node %d joined\n", node_id);
netlayer_api_log_error_print("malloc failed, size=%u\n", size);
```

---

## 5. 堆内存接口集（base/api.h）

```c
void *netlayer_api_mem_malloc(unsigned int size);
/* 申请size字节堆内存，失败返回NULL */

void netlayer_api_mem_free(void *mem);
/* 释放堆内存 */
```

---

## 6. 内存池接口集（base/api.h）

```c
int netlayer_api_memp_create(const char *name, int ele_num, int ele_size);
/* 创建内存池，返回ID句柄，失败返回-1 */
/* name: 标识名称, ele_num: 节点总数, ele_size: 单节点字节大小 */

void *netlayer_api_memp_malloc(int id);
/* 从内存池申请一个节点，返回节点地址 */

void netlayer_api_memp_free(int id, void *mem);
/* 归还节点到内存池 */

Bool netlayer_api_memp_stats(int id, int *total, int *used, int *max, int *err);
/* 获取内存池统计：总数、已用、峰值、失败次数 */

Bool netlayer_api_memp_destroy(int id);
/* 销毁内存池 */
```

---

## 7. 消息队列接口集（base/api.h）

```c
int netlayer_api_msgqueue_create(const char *name, int size);
/* 创建消息队列，size为容量个数，返回ID句柄，失败返回-1 */

Bool netlayer_api_msgqueue_send(int id, void *msg);
/* 发送消息。注意：msg不能指向栈数据！ */

Bool netlayer_api_msgqueue_recv(int id, void **msg, unsigned int timeout_ms);
/* 接收消息。timeout_ms=0为阻塞等待 */

Bool netlayer_api_msgqueue_stats(int id, int *total, int *used, int *max, int *err);
/* 获取队列统计信息 */

Bool netlayer_api_msgqueue_destroy(int id);
/* 销毁消息队列 */
```

---

## 8. 同步锁接口集（base/api.h）

### 8.1 互斥锁
```c
int  netlayer_api_mutex_create(void);      /* 创建，返回ID句柄 */
Bool netlayer_api_mutex_lock(int id);      /* 加锁 */
Bool netlayer_api_mutex_unlock(int id);    /* 解锁 */
Bool netlayer_api_mutex_destroy(int id);   /* 销毁 */
```

### 8.2 自旋锁
```c
NETLAYER_SPINLOCK_T netlayer_api_spinlock_create(const char *name);
Bool netlayer_api_spinlock_destroy(NETLAYER_SPINLOCK_T spinlock);
Bool netlayer_api_spinlock_trylock(NETLAYER_SPINLOCK_T spinlock);
Bool netlayer_api_spinlock_lock(NETLAYER_SPINLOCK_T spinlock);
Bool netlayer_api_spinlock_unlock(NETLAYER_SPINLOCK_T spinlock);
```

说明：
- `NETLAYER_SPINLOCK_T` 当前基于 atomic 封装，对上层仍保持平台无关
- 适合极短临界区的忙等保护，不适合长耗时逻辑
- 推荐使用场景：保护几个字段的一致性、短链表/状态位切换
- 若临界区可能阻塞、可能睡眠或执行时间不确定，优先使用 `netlayer_api_mutex_*`

### 8.3 原子操作
```c
NETLAYER_ATOMIC_T netlayer_api_atomic_create(const char *name, S32 init_value);
Bool netlayer_api_atomic_destroy(NETLAYER_ATOMIC_T atomic);
S32  netlayer_api_atomic_read(NETLAYER_ATOMIC_T atomic);
Bool netlayer_api_atomic_set(NETLAYER_ATOMIC_T atomic, S32 new_value);
Bool netlayer_api_atomic_add(NETLAYER_ATOMIC_T atomic, S32 add_value);
Bool netlayer_api_atomic_sub(NETLAYER_ATOMIC_T atomic, S32 sub_value);
Bool netlayer_api_atomic_inc(NETLAYER_ATOMIC_T atomic);
Bool netlayer_api_atomic_dec(NETLAYER_ATOMIC_T atomic);
S32  netlayer_api_atomic_add_return(NETLAYER_ATOMIC_T atomic, S32 add_value);
S32  netlayer_api_atomic_sub_return(NETLAYER_ATOMIC_T atomic, S32 sub_value);
S32  netlayer_api_atomic_inc_return(NETLAYER_ATOMIC_T atomic);
S32  netlayer_api_atomic_dec_return(NETLAYER_ATOMIC_T atomic);
S32  netlayer_api_atomic_cmpxchg(NETLAYER_ATOMIC_T atomic, S32 old_value, S32 new_value);
```

### 8.4 信号量
```c
int  netlayer_api_sem_create(const char *name, unsigned char cnt);
/* 创建信号量，cnt为初始信号值 */

Bool netlayer_api_sem_signal(int id);      /* 信号发送（V操作） */

Bool netlayer_api_sem_wait(int id, unsigned int timeout_ms);
/* 信号等待（P操作），timeout_ms=0为阻塞等待 */

Bool netlayer_api_sem_destroy(int id);     /* 销毁 */
```

---

## 9. 时间与定时器接口集（base/api.h）

```c
U32  netlayer_api_time_now(void);          /* 获取毫秒级系统时间 */
U32  netlayer_api_time_us_now(void);       /* 获取微秒级系统时间 */
void netlayer_api_msleep(U32 ms);          /* 毫秒级休眠 */

typedef int (*netlayer_api_timer_work_func)(void *);

int netlayer_api_timer_create(
    const char *name,                       /* 定时器名称 */
    netlayer_api_timer_work_func work_func, /* 回调函数 */
    void *param,                            /* 回调参数 */
    int circle_flag,                        /* TRUE=循环定时, FALSE=单次 */
    U32 interval_ms                         /* 定时间隔(ms) */
);
/* 返回定时器ID句柄，失败返回-1 */

Bool netlayer_api_timer_destroy(int id);
Bool netlayer_api_timer_setinterval(int id, U32 interval_ms);
```

---

## 10. 数据收发接口集（base/api.h）

### 10.1 netpkt发送
```c
Bool netlayer_api_send_netpkt_lower(IF_ID_T if_id, struct netpkt *pkt);
/* 向低层（MAC层）发送netpkt */

Bool netlayer_api_send_netpkt_higher(IF_ID_T if_id, struct netpkt *pkt);
/* 向高层（应用层）发送netpkt */
```

### 10.2 控制消息收发
```c
Bool netlayer_api_send_ctl_msg(IF_ID_T if_id, U8 *data, U16 data_len, struct data_attr_t *attr);
/* 发送控制消息 */

typedef Bool (*netlayer_ctl_msg_handle_func)(IF_ID_T if_id, struct netpkt *pkt);
Bool netlayer_api_recv_ctl_msg_register(
    U8 dst_module_id,
    netlayer_ctl_msg_handle_func recv_lower_func,   /* 接收来自低层的控制消息 */
    netlayer_ctl_msg_handle_func recv_higher_func    /* 接收来自高层的控制消息 */
);
```

### 10.3 数据接收注册
```c
typedef Bool (*netlayer_recv_pdu_func)(IF_ID_T if_id, struct netpkt *pkt);
Bool netlayer_api_recv_pdu_func_register(
    char *name,
    netlayer_recv_pdu_func recv_func,
    pdu_type_t type,
    U8 default_flag     /* 1=设为默认处理函数 */
);

typedef Bool (*netlayer_recv_eth_func)(IF_ID_T if_id, struct netpkt *pkt);
Bool netlayer_api_recv_eth_func_register(
    char *name,
    netlayer_recv_eth_func recv_func,
    U16 eth_proto,
    U8 default_flag
);
```

### 10.4 控制消息发送属性结构体
```c
struct data_attr_t
{
    U8 dst_module_id;        /* 目的模块ID */
    U8 src_module_id;        /* 源模块ID */
    U8 direct_type;          /* 0=向低层发送, 1=向高层发送 */
    NODE_ADDR_T dst_node_addr; /* 目的节点地址, 0xffff=广播 */
    U8 data_type;            /* 0=控制报文, 1=数据报文 */
    U8 type;                 /* 数据报文类型 */
    U8 rtlx;                 /* 可靠性传输标志 */
    U8 pri;                  /* 优先级 */
    U8 ttl;                  /* 转发跳数 */
};
#define INVALID_VAL 0xFE     /* 字段无效值 */
```

---

## 11. 参数控制接口（base/api.h）

```c
typedef int (*netlayer_api_attr_get_func)(char *buf, int size);
typedef int (*netlayer_api_attr_set_func)(int argc, char **argv, char *buf, int size);

Bool netlayer_api_attr_add_v2(
    const char *attr_name,     /* 参数完整名称 */
    const char *short_name,    /* 参数缩写名称 */
    const char *validvalue,    /* 有效值描述，如 "[0|1]" */
    netlayer_api_attr_get_func get_func,
    netlayer_api_attr_set_func set_func,
    const char *description,   /* 英文描述 */
    const char *description_zh /* 中文描述 */
);

说明：
- `get_func` / `set_func` 的输出缓冲区由框架提供，回调内部需自行保证不越界
```

---

## 12. netpkt数据包操作（base/netpkt.h）

### 12.1 netpkt结构体
```c
#define NETPKT_RESERVER_HDR_LEN 200 /* 预留头长度 */
struct netpkt_attr_t
{
    U16 val;       /* 标记值 */
    U8  type;      /* 业务类型 */
    U8  pri;       /* 业务优先级 */
    U8  retx;      /* 是否重传 */
    U8  ttl;       /* 生存跳数 */
    U16 livetm;    /* 生存时间 */
    U16 strm_id;   /* 业务流ID */
    U8  bitmask;   /* 属性项掩码 */
};

struct netpkt
{
    U8  *data;              /* 数据指针 */
    U16  data_len;          /* 数据长度 */
    U8  *usr_data;          /* 用户数据/控制消息 */
    U16  usr_data_len;
    U8  *proto_data;        /* 网络层私有头 */
    U8  *eth_hdr;           /* 以太网头 */
    U8  *vlan_hdr;          /* VLAN头 */
    U8  *arp_hdr;           /* ARP头 */
    U8  *ip_hdr;            /* IP头 */
    U8  *icmp_hdr;          /* ICMP头 */
    U8  *igmp_hdr;          /* IGMP头 */
    U8  *tran_hdr;          /* TCP/UDP传输头 */
    U8  *intl_phdr;         /* 内部协议头 */
    U8   hook;              /* 数据所在hook点 */
    U32 data_send_time_us;  // 数据包发送处理时间
    U32 data_recv_time_us;  // 数据包接收处理时间
    struct netpkt_attr_t attr;
    NODE_ID_T src_node_id;  /* 源节点ID */
    netpkt_type type;       /* 申请方式类型 */
    U8   ref;               /* 引用计数 */
    U8   flags;
    IF_ID_T if_id;          /* 接收信道 */
    int  pool_id;           /* 内存池句柄 */
    void *ref_orig;         /* 原始数据对象 */
    netpkt_free_orig_func free_orig_func; /* 原始数据释放函数 */
};
```

### 12.2 netpkt操作接口
```c
typedef enum {
    NETPKT_RAM = 0,   /* RAM申请 */
    NETPKT_REF,       /* 壳申请（仅结构体，数据引用外部） */
    NETPKT_POOL,      /* 内存池申请（结构体+数据连续） */
} netpkt_type;

struct netpkt *netpkt_alloc(
    IF_ID_T if_id,
    U8 *orig_data,
    U16 orig_data_len,
    netpkt_free_orig_func free_orig_func,
    void *ref_orig,
    netpkt_type type
);
/* 申请netpkt，失败返回NULL */

void netpkt_free(struct netpkt *pkt);    /* 释放netpkt */
void netpkt_ref(struct netpkt *pkt);     /* 增加引用计数 */
```

---

## 13. 模块管理接口（base/module.h）

### 13.1 模块类型定义
```c
enum MODULE_TYPE_E {
    ROUTE_MODULE = 0,   /* 路由管理 */
    DATA_MODULE,        /* 中继转发 */
    ADDR_MODULE,        /* 地址管理 */
    SORTER_MODULE,      /* 重排序 */
    HETNET_MODULE,      /* 异构管理 */
    KPI_MODULE,         /* 维测管理 */
    PKTABLE_MODULE,     /* pktable */
    TOPO_MODULE,        /* 拓扑管理 */
    MODULE_TYPE_CNT,
};
```

### 13.2 模块事件
```c
enum MODULE_EVENT_TYPE_E {
    MODULE_EVENT_STARTED,    /* 模块启动事件 */
    MODULE_EVENT_STOPPED,    /* 模块停止事件 */
    MODULE_EVENT_MAX
};

struct module_event_msg_t
{
    enum MODULE_EVENT_TYPE_E event_type;
    char src_module_name[64];
    enum MODULE_TYPE_E src_module_type;
};
```

### 13.3 模块描述对象获取
```c
struct netlayer_module_desc *netlayer_module_get_by_type(U8 type);
/* 根据模块类型获取已启动的模块描述对象，未启动返回NULL */
```

### 13.4 模块描述对象结构体
```c
struct netlayer_module_desc
{
    const char *name;               // 模块名称
    const char *ver_info;           // 模块版本
    enum MODULE_TYPE_E type;        // 模块类型
    const char *description;        // 模块描述信息
    void *api_struct;               // 模块API接口结构体指针
    struct netlayer_module_route_api_t *route_api;
    struct netlayer_module_data_api_t *data_api;
    struct netlayer_module_addr_api_t *addr_api;
    struct netlayer_module_sorter_api_t *sorter_api;
    struct netlayer_module_hetnet_api_t *hetnet_api;
    struct netlayer_module_kpi_api_t *kpi_api;
    struct netlayer_module_pktable_api_t *pktable_api;
    struct netlayer_module_topo_api_t *topo_api;
    void (*event_cb_func)(struct module_event_msg_t *event_msg); // 事件通知函数
    U8 status;                      // 模块运行状态
};
```

跨模块调用示例：
```c
struct netlayer_module_desc *route_desc = netlayer_module_get_by_type(ROUTE_MODULE);
if (route_desc && route_desc->route_api && route_desc->route_api->get_nexthop_by_dstid_func) {
    /* 通过 route_api 指针调用路由模块接口 */
}
```

### 13.5 模块声明宏
```c
NETLAYER_MODULE_DECLARE(tag, "name", "version", type, "description", api_struct, event_cb_func)
```

### 13.6 各模块类型标准API接口结构体

#### 路由管理模块
```c
struct netlayer_module_route_api_t
{
    Bool (*init_func)(void *param);
    Bool (*uninit_func)(void *param);
    NODE_ADDR_T (*get_nodeaddr_by_nodeid_func)(NODE_ID_T node_id);
    NODE_ID_T   (*get_nodeid_by_nodeaddr_func)(NODE_ADDR_T node_addr);
    Bool   (*get_nexthop_by_dstid_func)(NODE_ID_T dst_id, NODE_ID_T *nexthop_id, U8 *hop_cnt);
    Bool (*get_nexthop_by_dstaddr_func)(NODE_ADDR_T dst_addr, NODE_ADDR_T *nexthop_addr, U8 *hop_cnt);
    Bool (*get_global_nodeid_func)(NODE_ID_T *node_list, unsigned int *node_cnt);
    Bool (*get_neigh_nodeid_func)(NODE_ID_T *node_list, unsigned int *node_cnt);
    netlayer_recv_handler_func lower_recv_handler_func;
    netlayer_tcpdump_handler_func tcpdump_handler_func;
};
```

#### 中继转发模块
```c
struct netlayer_module_data_api_t
{
    Bool (*init_func)(void *param);
    Bool (*uninit_func)(void *param);
    Bool (*node_data_service_create_func)(NODE_ID_T node_id);
    Bool (*data_send_func)(IF_ID_T if_id, U8 *data, U16 data_len, struct data_attr_t *attr);
    Bool (*netpkt_parse_func)(struct netpkt *pkt, U8 hook);
    Bool (*netpkt_set_func)(struct netpkt *pkt, U8 hook);
    netlayer_recv_handler_func higher_recv_handler_func;
    netlayer_recv_handler_func lower_recv_handler_func;
    netlayer_tcpdump_handler_func tcpdump_handler_func;
};
```

#### 地址管理模块
```c
struct netlayer_module_addr_api_t
{
    Bool (*init_func)(void *param);
    Bool (*uninit_func)(void *param);
    Bool (*get_orig_by_addr_func)(U8 *addr, U16 addr_len, struct netlayer_dst_node_info *dst_node);
    Bool (*addr_higher_recv_handler_func)(IF_ID_T if_id, U8 *data, U16 data_len, struct netlayer_dst_node_info *dst_node);
    Bool (*addr_lower_recv_handler_func)(IF_ID_T if_id, struct netpkt *pkt);
    netlayer_tcpdump_handler_func tcpdump_handler_func;
};
```

#### 重排序模块
```c
struct netlayer_module_sorter_api_t
{
    Bool  (*init_func)(void *param);
    Bool  (*uninit_func)(void *param);
    void *(*sorter_alloc_func)(const char *name, U32 timeout_detect_ms, U32 timeout_ms, U32 window_size, U32 max_seqno_num);
    Bool  (*sorter_destroy_func)(void *sorter);
    Bool  (*sorter_packet_proccess_func)(void *sorter, U32 seqno_num, Bool (*cb)(U32 seqno_num, void *param), void *param);
    Bool  (*sorter_stats_get_func)(void *sorter, U64 *total_recv, U64 *total_sorted, U64 *timeout, U64 *out_of_order, U64 *duplicate, U64 *expired, U64 *big_detected, U64 *restart_detected);
    netlayer_tcpdump_handler_func tcpdump_handler_func;
};
```

#### 异构管理模块
```c
struct netlayer_module_hetnet_api_t
{
    Bool (*init_func)(void *param);
    Bool (*uninit_func)(void *param);
    netlayer_recv_handler_func higher_recv_handler_func;
    Bool (*check_node_id_is_backbone_func)(NODE_ID_T node_id);
    netlayer_tcpdump_handler_func tcpdump_handler_func;
};
```

#### 维测管理模块
```c
struct netlayer_module_kpi_api_t
{
    Bool (*init_func)(void *param);
    Bool (*uninit_func)(void *param);
    Bool (*kpi_item_add_and_init_func)(U8 kpi_type, const U8 *kpi_name, Bool post_enable, U32 post_interval, Bool (*kpi_msg_data_generate_func)(int *row, int *col, struct list_head *head, U16 col_max_data_len));
    Bool (*kpi_item_post_func)(U8 kpi_type, int (*netlayer_kpi_post_func)(const char *data, U32 data_len));
    netlayer_tcpdump_handler_func tcpdump_handler_func;
};
```

#### pktable模块
```c
struct netlayer_module_pktable_api_t
{
    Bool (*init_func)(void *param);
    Bool (*uninit_func)(void *param);
    int  (*pkt_do_command_extern)(int argc, char **argv, char *buf, int size, netlayer_recv_match_func match_func, netlayer_recv_handler_func handler_func);
};
```

#### 拓扑管理模块
```c
struct netlayer_module_topo_api_t
{
    Bool (*init_func)(void *param);
    Bool (*uninit_func)(void *param);
    netlayer_recv_handler_func lower_recv_handler_func;
    int  (*netlayer_topo_info_collect_func)(void *data, U32 data_len);
    int  (*netlayer_topo_share_send_func)(IF_ID_T if_id, Bool (*topo_data_send_func)(IF_ID_T if_id, U8 *data, U16 data_len, struct data_attr_t *attr));
    int  (*netlayer_get_topo_info_func)(netlayer_plat_protocol_msg_report_func topo_info_push_fun);
    netlayer_tcpdump_handler_func tcpdump_handler_func;
};
```

---

## 14. 双向链表（base/list.h）

```c
struct list_head { struct list_head *next, *prev; };

void INIT_LIST_HEAD(struct list_head *list);
void list_add_tail(struct list_head *new, struct list_head *head);
void list_del(struct list_head *entry);
int  list_empty(const struct list_head *head);

/* 遍历宏 */
list_for_each_entry(pos, head, member)
```

---

## 15. 哈希表（base/hashmap.h）

> hashmap 的选型原则、API行为详解、推荐写法和反例请查阅 `data_structure_guide.md`。本章仅提供结构体定义和函数签名速查。

### 15.1 关键类型与结构体

```c
#define HASHMAP_MIN_SLOT_SIZE_LIMIT 260
#define HASHMAP_MAX_SLOT_SIZE_LIMIT 65535

typedef HASH_KEY_T (*hash_calc_func)(U8 *, U16);
typedef NETLAYER_SPINLOCK_T ATOMIC_LOCK_T;
typedef int (*hashmap_print_func)(void **val, char *buf, void *param);
typedef int (*hashmap_lookup_func)(void **val, void *param);

struct hashmap_item_t {
    HASH_KEY_T   key;                    // 用户数据计算的hash值
    void*        val;                    // 存储用户数据
    U32          slot_no;                // 所属槽编号
    Bool         live_flag;              // 用户数据存活状态
    struct list_head slot_list;          // 构建槽内链表
    struct list_head hashmap_list;       // 构建hashmap链表，用于快速遍历hashmap数据
};

struct hashmap_slot_t {
    U32 slot_no;                         // 槽编号
    U32 slot_len;                        // 槽下面存储的数据数量长度
    struct list_head head;               // 槽数据头
    ATOMIC_LOCK_T list_lock;             // 槽操作锁
};

struct hashmap_t {
    char         hashmap_name[HASHMAP_NAME_LEN];      // hashmap名
    U32          slot_size;                           // hashmap槽数目
    struct hashmap_slot_t *slot;                      // 槽数组，用于高效查找
    U32          total_cnt;                           // 存储的数据总数目
    U32          live_cnt;                            // live_flag == 1 的条目数
    struct list_head head;                            // hashmap链表头
    ATOMIC_LOCK_T list_lock;                          // hashmap链表操作锁
    hash_calc_func hash;                              // hash计算函数
    hashmap_print_func hashmap_print;
    U32 min_use_slot_no;
};
```

### 15.2 核心接口

```c
struct hashmap_t *hashmap_new(const char *name, U32 slot_size);
Bool  hashmap_destroy(struct hashmap_t *hashmap);
void *hashmap_get_by_index(struct hashmap_t *hashmap, U32 index);
void *hashmap_find(struct hashmap_t *hashmap, HASH_KEY_T key);
void *hashmap_add(struct hashmap_t *hashmap, HASH_KEY_T key, void *val);
Bool  hashmap_del(struct hashmap_t *hashmap, HASH_KEY_T key);
int   hashmap_show(struct hashmap_t *hashmap, const char *root_name, char *show_buf, hashmap_print_func print, void *param);
int   hashmap_show_by_slot(struct hashmap_t *hashmap, const char *root_name, char *show_buf, hashmap_print_func print, void *param);
Bool  hashmap_lookup(struct hashmap_t *hashmap, hashmap_lookup_func item_deal, void *param);
```

---

## 16. 工具函数（base/def.h）

### 字节序转换
```c
U16 netlayer_htons(U16 x);   U16 netlayer_ntohs(U16 x);
U32 netlayer_htonl(U32 x);   U32 netlayer_ntohl(U32 x);
```

### 节点判断
```c
Bool netlayer_is_my_nodeid(NODE_ID_T node_id);
Bool netlayer_is_my_nodeaddr(NODE_ADDR_T node_addr);
```

### MAC地址操作
```c
Bool  netlayer_macaddr_is_equal(const U8 *addr1, const U8 *addr2);
char *netlayer_macaddr_to_str(U8 *mac_addr, char *buf, int size);
void  netlayer_str_to_macaddr(char *strmac, U8 *mac_addr);
Bool  netlayer_is_broadcast_ether_addr(const U8 *mac_addr);
Bool  netlayer_is_multicast_ether_addr(const U8 *mac_addr);
Bool  netlayer_is_unicast_ether_addr(const U8 *mac_addr);
Bool netlayer_is_my_mac_addr_seg(const U8 *mac_addr, U8 seg_len);
```

### IP地址操作
```c
char *netlayer_ip_to_str(const U32 ipv4_addr, char *buf, int size);
int   netlayer_str_to_ip(const char *ip_str, U32 *ip_addr);
U32   netlayer_in_aton(const char *str);
Bool  netlayer_is_multicast_ip(U32 multi_addr);
```

### 字符串操作（跨平台安全版本）
```c
int   netlayer_rt_strlen(const char *str);
char *netlayer_rt_strcpy(char *dst, const char *src);
int   netlayer_rt_strcmp(const char *s1, const char *s2);
char *netlayer_rt_strdup(const char *str);
char *netlayer_rt_strtok(char *s, const char *delim);
```

### 内存操作（跨平台安全版本）
```c
void *netlayer_rt_memcpy(void *dst, const void *src, int cnt);
int   netlayer_rt_memcmp(const void *m1, const void *m2, int cnt);
void *netlayer_rt_memset(void *dst, int value, int cnt);
```

### 数值与格式化
```c
U32   netlayer_atoi(const char *str);
U16   netlayer_checksum(U16 *buf, U32 size); /* 计算校验和 */
char *netlayer_data_to_hex_str(const U8 *data, U16 datalen, char *buf, int size);
char *netlayer_bps_to_str(U32 bps, char *buf, int size);
char *netlayer_distance_to_str(U32 distance, char *buf, int size);
char *netlayer_time_to_str(const U32 time, char *strtime);
char *netlayer_reverse_string(char *str);
void netlayer_convert_string_to_argv(const char *str, int *argc, char ***argv);
void netlayer_convert_argv_free(int argc, char **argv);
struct netlayer_time_t {
    S32 tm_sec;   // 秒 [0, 59]
    S32 tm_min;   // 分 [0, 59]
    S32 tm_hour;  // 时 [0, 23]
    S32 tm_mday;  // 一个月中的日期 [1, 31]
    S32 tm_mon;   // 月份 [0, 11] (0表示一月)
    S32 tm_year;  // 年份，从1900年开始
};
void netlayer_timestamp_to_tm(U64 timep, struct netlayer_time_t *tm_info);
char netlayer_timestap_zone_to_verfmt(const char *timestap, char *tm_verfmt);
#define BPS_CALC_BASE 1000   // 比特速率转Mbps或kbps的取模基准值
#define GBIT_SIZE (BPS_CALC_BASE*BPS_CALC_BASE*BPS_CALC_BASE)
#define MBIT_SIZE (BPS_CALC_BASE*BPS_CALC_BASE)
#define KBIT_SIZE (BPS_CALC_BASE)
```

### netpkt用户数据解析
```c
Bool netlayer_usr_data_parse(struct netpkt *pkt) /* 解析用户态以太网帧数据，得到包括eth头位置，ip头位置，tcp/udp头位置等信息 */
```

### 实用宏
```c
#define NETLAYER_MAX(x, y)    (((x) > (y)) ? (x) : (y))
#define NETLAYER_MIN(x, y)    (((x) < (y)) ? (x) : (y))
#define NETLAYER_ARRAYSIZE(x) (sizeof(x) / sizeof((x)[0]))
```

---

## 17. 追踪与覆盖率（base/trace.h, base/coverage.h）

```c
/* 函数追踪桩 */
void netlayer_trace_func_start(void);
void netlayer_trace_func_end(void);

/* 覆盖率生成 */
typedef void (*netlayer_coverage_generate_func)(const char *dir_path, const char *coverage_file);
extern netlayer_coverage_generate_func netlayer_coverage_generate;
Bool netlayer_coverage_generate_register(netlayer_coverage_generate_func func);
```
