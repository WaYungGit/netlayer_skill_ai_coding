# hashmap 使用规范补充

本文档是 `../SKILL.md` 中“数据结构选型规范”的专项补充，也是 `api_reference.md` 中哈希表章节的行为说明。建议按以下分工查阅：

- `../SKILL.md`：什么时候选 `hashmap`、key 如何设计、何时需要模块层互斥锁
- `hashmap_usage_spec.md`：`hashmap_add` / `hashmap_del` / `hashmap_destroy` / `hashmap_lookup` 的真实行为、内存所有权和推荐写法
- `api_reference.md`：结构体定义、函数签名和其他 Base 框架 API 速查

## hashmap 关键行为补充

框架 `hashmap` 内部已经对基础增删查遍历做了锁保护，但它的删除与重复添加语义和常见“覆盖式容器”不同，写模块代码时必须严格按当前实现理解。

### 1. hashmap_add 的行为与返回值

```c
void *hashmap_add(struct hashmap_t *hashmap, HASH_KEY_T key, void *val);
```

**行为规则：**
- **key 不存在**：将 val 添加到 hashmap 中，返回 val 本身（即返回值 == 传入的 val）
- **key 已存在**：**释放传入的新 val**，返回旧的 val 指针，供调用者直接修改旧条目字段
- **key 已存在但条目此前被 `hashmap_del` 逻辑删除**：仍然复用旧条目，恢复 `live_flag`，并返回旧的 val 指针

**推荐使用模式（find-first）：**

先 `find` 判断是否存在，存在则直接改字段（零开销），不存在才申请内存并 `add`。
在模块级 mutex 保护下，find 和 add 之间状态不会被其他线程改变，因此 find-first 是安全且高效的。

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
    /* add 后通过 entry（即返回值）填充字段 */
    entry->node_addr = addr;
    entry->field1 = value;
    entry->last_update_ms = netlayer_api_time_now();
}
```

**禁止的错误模式：**

```c
/* 错误1：不 find 直接申请内存 + add（已存在时白白浪费申请/释放） */
new_entry = netlayer_api_mem_malloc(...);
hashmap_add(table, key, new_entry);  /* 若 key 已存在，new_entry 被内部释放，浪费 */

/* 错误2：add 前先完整初始化 val 的所有字段 */
new_entry = netlayer_api_mem_malloc(...);
new_entry->field1 = complex_calc();  /* 若 key 已存在，new_entry 被释放，计算浪费 */
hashmap_add(table, key, new_entry);
/* 此后访问 new_entry->field1 是野指针！ */
```

### 2. hashmap_del 的行为

```c
Bool hashmap_del(struct hashmap_t *hashmap, HASH_KEY_T key);
```

**行为规则：**
- **不释放内存**，仅将对应 `hashmap_item_t` 的 `live_flag` 置为 0
- 条目仍然留在 hashmap 的链表结构中，占用内存
- `hashmap_find` 查找时会跳过 `live_flag == 0` 的条目
- `live_cnt` 计数减1，但 `total_cnt` 不变
- 若后续再次对同一 key 调用 `hashmap_add`，会复用旧条目，而不是重新创建新条目

**严禁在 hashmap_del 后 free val：**

```c
/* 错误：hashmap_del 后释放 val */
entry = hashmap_find(table, key);
hashmap_del(table, key);
netlayer_api_mem_free(entry);     /* 严禁！val 仍在 hashmap 内部管理 */

/* 正确：只调用 hashmap_del，不管内存 */
hashmap_del(table, key);
/* val 的内存由 hashmap_destroy 时统一回收 */
```

### 3. hashmap_destroy 的行为

```c
Bool hashmap_destroy(struct hashmap_t *hashmap);
```

**行为规则：**
- 销毁整个 hashmap，释放所有内部结构和所有条目（包括 `live_flag == 0` 的条目）
- 条目中 `val` 指向的用户数据也会被释放
- 调用后 hashmap 指针不可再使用

**因此 uninit 时不需要手动遍历 free 每个条目**，直接调用 `hashmap_destroy` 即可。

### 4. hashmap_find 的行为

```c
void *hashmap_find(struct hashmap_t *hashmap, HASH_KEY_T key);
```

- 仅返回 `live_flag == 1` 的条目的 val 指针
- `live_flag == 0` 的条目不可见（等价于已删除）
- 返回 NULL 表示未找到

### 5. hashmap_lookup 遍历的行为

```c
Bool hashmap_lookup(struct hashmap_t *hashmap, hashmap_lookup_func item_deal, void *param);
```

- 遍历所有 **`live_flag == 1` 且 `val != NULL`** 的条目
- 回调参数是 `void **val`，用于支持直接操作条目指针或 `container_of` 场景
- 回调返回 `-1` 时提前停止遍历

**注意：** `hashmap_lookup` 本身会跳过已删除条目；但如果直接遍历 `hashmap->head` 或 `slot[i].head`，仍然可能看到 `live_flag == 0` 的条目，因此必须自行检查 `item->live_flag`。

### 6. live_cnt 与 total_cnt

- `live_cnt`：`live_flag == 1` 的条目数（即"有效"条目数）
- `total_cnt`：所有条目数（含已标记删除的）
- 容量判断应使用 `live_cnt`（如 `live_cnt >= max_num` 则表满）

### 7. 典型完整使用模式

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
        /* 已存在，直接修改 */
        entry->hop_type = hop_type;
        entry->last_update_ms = netlayer_api_time_now();
        return;
    }

    /* 不存在，申请内存后 add */
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
