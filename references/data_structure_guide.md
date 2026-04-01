# 数据结构选型规范与 hashmap 使用指南

模块内部需要存储和检索数据集合时，**优先使用框架提供的 hashmap**（`base/hashmap.h`），而非纯链表或静态数组线性扫描。

## 一、选型原则

| 场景 | 推荐数据结构 |
|------|-------------|
| 条目数量 >= 16 或需要按 key 查找、判重、快速增删 | **hashmap** |
| 仅需顺序遍历、数量极少（<16）且无需按 key 查找 | 链表或小数组 |
| 需要同时支持 key 查找和顺序遍历 | hashmap + 链表组合 |
| 条目数量不确定但可能增长 | 优先选 hashmap |

**关于哈希冲突：** 框架 hashmap 采用链地址法处理槽位冲突。只要 key 计算合理（使用地址内容、节点ID等），实际冲突率极低，性能接近 O(1)。槽位数量（slot_size）建议设为预期最大条目数的 1~2 倍。

**禁止的低效模式：**

```c
/* 错误：用链表存储大量条目，查找时线性遍历 */
struct list_head big_list;
struct my_entry_t *pos = NULL;
list_for_each_entry(pos, &big_list, list) {
    if (pos->node_id == target_id) { ... }  /* O(n)，条目多时性能差 */
}

/* 正确：用 hashmap，O(1) 查找 */
struct my_entry_t *found = hashmap_find(table, calc_hash(target_id));
```

---

## 二、hashmap API 行为详解

框架 hashmap 内部已对基础增删查遍历做了锁保护，但它的删除与重复添加语义和常见"覆盖式容器"不同，必须严格理解。

### 2.1 hashmap_add

```c
void *hashmap_add(struct hashmap_t *hashmap, HASH_KEY_T key, void *val);
```

**行为规则：**
- **key 不存在**：将 val 添加到 hashmap 中，返回 val 本身
- **key 已存在**：**释放传入的新 val**，返回旧的 val 指针，供调用者直接修改旧条目字段
- **key 已存在但此前被 `hashmap_del` 逻辑删除**：复用旧条目，恢复 `live_flag`，返回旧的 val 指针

### 2.2 hashmap_del

```c
Bool hashmap_del(struct hashmap_t *hashmap, HASH_KEY_T key);
```

**行为规则：**
- **不释放内存**，仅将 `live_flag` 置为 0
- 条目仍留在 hashmap 的链表结构中
- `hashmap_find` 会跳过 `live_flag == 0` 的条目
- `live_cnt` 减1，但 `total_cnt` 不变
- 后续对同一 key 调用 `hashmap_add` 会复用旧条目

**严禁在 hashmap_del 后 free val：**

```c
/* 错误 */
entry = hashmap_find(table, key);
hashmap_del(table, key);
netlayer_api_mem_free(entry);     /* 严禁！val 仍在 hashmap 内部管理 */

/* 正确 */
hashmap_del(table, key);
/* val 的内存由 hashmap_destroy 时统一回收 */
```

### 2.3 hashmap_destroy

```c
Bool hashmap_destroy(struct hashmap_t *hashmap);
```

- 销毁整个 hashmap，释放所有内部结构和所有条目（包括 `live_flag == 0` 的）
- 条目中 `val` 指向的用户数据也会被释放
- uninit 时不需要手动遍历 free 每个条目，直接调用 `hashmap_destroy` 即可

### 2.4 hashmap_find

```c
void *hashmap_find(struct hashmap_t *hashmap, HASH_KEY_T key);
```

- 仅返回 `live_flag == 1` 的条目的 val 指针
- 返回 NULL 表示未找到

### 2.5 hashmap_lookup（遍历）

```c
Bool hashmap_lookup(struct hashmap_t *hashmap, hashmap_lookup_func item_deal, void *param);
```

- 遍历所有 `live_flag == 1` 且 `val != NULL` 的条目
- 回调参数是 `void **val`
- 回调返回 `-1` 时提前停止遍历

**注意：** 不要直接遍历 `hashmap->head` 或 `slot[i].head`，内部链表可能包含已删除条目。优先使用 `hashmap_find` / `hashmap_lookup` 等公开接口。

### 2.6 live_cnt 与 total_cnt

- `live_cnt`：`live_flag == 1` 的条目数（有效条目）
- `total_cnt`：所有条目数（含已标记删除的）
- 容量判断应使用 `live_cnt`

---

## 三、推荐使用模式（find-first）

先 `find` 判断是否存在，存在则直接改字段（零开销），不存在才申请内存并 `add`。在模块级 mutex 保护下，find 和 add 之间状态不会被其他线程改变，因此 find-first 是安全且高效的。

```c
struct my_entry_t *entry = NULL;
struct my_entry_t *new_entry = NULL;

entry = (struct my_entry_t *)hashmap_find(table, key);

if (entry) {
    /* 已存在，直接修改字段，不需要 malloc */
    entry->field1 = new_value;
    entry->last_update_ms = netlayer_api_time_now();
} else {
    /* 不存在，申请内存后 add */
    new_entry = (struct my_entry_t *)netlayer_api_mem_malloc(sizeof(struct my_entry_t));
    if (!new_entry) {
        return;
    }
    netlayer_rt_memset(new_entry, 0, sizeof(struct my_entry_t));

    entry = (struct my_entry_t *)hashmap_add(table, key, (void *)new_entry);
    entry->node_addr = addr;
    entry->field1 = value;
    entry->last_update_ms = netlayer_api_time_now();
}
```

**禁止的错误模式：**

```c
/* 错误1：不 find 直接申请内存 + add（已存在时白白浪费申请/释放） */
new_entry = netlayer_api_mem_malloc(...);
hashmap_add(table, key, new_entry);

/* 错误2：add 前先完整初始化 val 的所有字段 */
new_entry = netlayer_api_mem_malloc(...);
new_entry->field1 = complex_calc();  /* 若 key 已存在，new_entry 被释放，计算浪费 */
hashmap_add(table, key, new_entry);
/* 此后访问 new_entry->field1 是野指针！ */
```

---

## 四、典型完整使用模式

```c
/* ======== 创建 ======== */
struct hashmap_t *table = hashmap_new("MY_TABLE", 512);

/* ======== 新增/更新条目（find-first 模式） ======== */
static void upsert_entry(struct hashmap_t *table, HASH_KEY_T key,
    NODE_ADDR_T addr, U8 hop_type)
{
    struct my_entry_t *entry = NULL;
    struct my_entry_t *new_entry = NULL;

    entry = (struct my_entry_t *)hashmap_find(table, key);
    if (entry) {
        entry->hop_type = hop_type;
        entry->last_update_ms = netlayer_api_time_now();
        return;
    }

    new_entry = (struct my_entry_t *)netlayer_api_mem_malloc(sizeof(struct my_entry_t));
    if (!new_entry) {
        return;
    }
    netlayer_rt_memset(new_entry, 0, sizeof(struct my_entry_t));

    entry = (struct my_entry_t *)hashmap_add(table, key, (void *)new_entry);
    entry->node_addr = addr;
    entry->hop_type = hop_type;
    entry->last_update_ms = netlayer_api_time_now();
}

/* ======== 删除条目 ======== */
hashmap_del(table, key);
/* 不要 free，内存由 hashmap 管理 */

/* ======== 查找 ======== */
struct my_entry_t *found = (struct my_entry_t *)hashmap_find(table, key);
if (found) {
    /* 使用 found */
}

/* ======== 遍历 ======== */
static int my_dump_cb(void **val, void *param)
{
    struct my_entry_t *e = NULL;

    if (!val || !*val) {
        return 0;
    }
    e = (struct my_entry_t *)(*val);
    /* 处理条目 e */
    return 0;
}
hashmap_lookup(table, my_dump_cb, NULL);

/* ======== 销毁（自动释放所有条目内存） ======== */
hashmap_destroy(table);
table = NULL;
```

---

## 五、并发注意事项

- hashmap 内部已对基础增删查和 `hashmap_lookup` 做了锁保护，使用者无需直接操作内部锁
- 若模块需要"查找后插入""遍历同时联动更新其他对象"等复合操作，应使用 `netlayer_api_mutex_*` 在模块层面加锁
- 不建议在并发路径里直接遍历 `hashmap->head` 或 `slot[i].head`
