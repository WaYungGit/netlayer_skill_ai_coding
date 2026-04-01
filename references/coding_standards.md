# C语言编码规范（内核态兼容）

由于Linux内核态要求使用C89/C90风格，以下规范必须严格遵守。

## 1. 变量声明规则

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

## 2. 循环变量规则

for循环的迭代变量必须在函数开头声明，循环中仅赋值：

```c
int i = 0;  /* 在函数开头声明 */
for (i = 0; i < count; i++) { ... }
```

## 3. 类型使用规则

- 使用框架自定义类型：`U8`, `U16`, `U32`, `U64`, `S8`, `S16`, `S32`, `S64`, `Bool`
- 布尔值使用 `TRUE`/`FALSE`
- 节点ID类型使用 `NODE_ID_T`，节点地址类型使用 `NODE_ADDR_T`
- 接口ID类型使用 `IF_ID_T`
- 哈希键类型使用 `HASH_KEY_T`
- 原子对象句柄类型使用 `NETLAYER_ATOMIC_T`
- 自旋锁对象句柄类型使用 `NETLAYER_SPINLOCK_T`

## 4. 注释规范

- 文件头部使用版权声明注释块
- 函数前用 `/**  */` 多行注释描述功能、参数和返回值
- 行内注释使用 `/* */`，禁止使用C99的 `//` 单行注释（部分严格的内核编译器不支持）
- 建议关键逻辑段落用中文注释说明意图

## 5. 命名规范

### 函数命名

- 模块对外接口函数：`netlayer_<模块全名>_<功能描述>`，如 `netlayer_route_lsr_get_nexthop`
- 模块内部静态函数：`netlayer_<模块全名>_<功能描述>`，加 `static` 修饰，或简写为 `<模块全名>_<功能描述>`
- 回调函数：`netlayer_<模块全名>_<事件>_cb`
- 所有函数名使用小写加下划线风格
- 其中 `<模块全名>` = `<类型简称>_<特征后缀>`，如 `route_lsr`、`addr_std`、`sorter_window`（详见 `module_development.md` 中"模块命名与多版本管理"章节）

### 结构体和宏命名

- 结构体：`struct netlayer_<模块全名>_<描述>_t`，如 `struct netlayer_route_lsr_neighbor_t`
- 枚举：`enum netlayer_<模块全名>_<描述>_e` 或全大写 `NETLAYER_<MODULE_FULL>_<DESC>_E`
- 宏定义：全大写加下划线，如 `ROUTE_LSR_HELLO_INTERVAL_MS`
- 全局上下文变量：`g_<模块全名>_ctx`，如 `g_route_lsr_ctx`

## 6. 错误处理模式

使用 `do { ... } while(0)` + `break` 实现资源安全释放的错误退出，对所有内存申请、资源创建的返回值进行判空检查：

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

## 7. 头文件保护

```c
#ifndef _NETLAYER_<MODULE>_<FILE>_H_
#define _NETLAYER_<MODULE>_<FILE>_H_
/* ... */
#endif /* _NETLAYER_<MODULE>_<FILE>_H_ */
```

## 8. 其他规范

- 结构体字段对齐需考虑跨平台，协议报文结构体使用 `#pragma pack(1)` 包裹
- `snprintf` 输出时使用 `len += snprintf(buf+len, size-len, ...)` 模式防止溢出
- 禁止使用VLA（变长数组），数组大小必须是编译期常量
- 禁止使用 `goto` 跨越变量声明（内核态可用goto做资源清理，但需谨慎）
- 结构体初始化使用C99指定初始化器 `.field = value` 是允许的（GCC扩展）
