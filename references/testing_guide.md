# 模块级测试用例开发规范

每个功能模块应提供完整的测试用例文件 `test.c` 和 `test.h`。

## test.h 模板

```c
/* Copyright (C) *******公司名******* */
#ifndef _NETLAYER_<MODULE>_TEST_H_
#define _NETLAYER_<MODULE>_TEST_H_

int netlayer_<module>_test_cases_get(char *buf, int size);
int netlayer_<module>_test_cases_run(int argc, char **argv, char *buf, int size);

#endif /* _NETLAYER_<MODULE>_TEST_H_ */
```

## test.c 结构模板

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

static int netlayer_<module>_test_basic(void *param)
{
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
    netlayer_api_msleep(500);

    return netlayer_<module>_test_check_result(obj, &expect_stats);
}

static int netlayer_<module>_test_boundary(void *param)
{
    /* ... 类似结构 ... */
    return -1;
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

    netlayer_coverage_generate("/core/<module>", "/var/run/<module>_coverage.txt");

    return 0;
}
```

## 编写准则

1. **每个测试用例是独立的函数**，签名为 `static int test_func(void *param)`，返回1通过、-1失败
2. **预期结果必须量化**：使用统计信息结构体定义期望值，通过check_result函数逐字段对比
3. **使用模块的标准API接口**（即api.h中导出的结构体），不直接调用内部函数，保证黑盒测试
4. **测试用例需覆盖**：基础功能、边界条件、异常输入、并发场景、资源释放
5. **异步操作需用 `netlayer_api_msleep()` 等待完成后再校验结果**
6. **使用 `netlayer_trace_func_start()` / `netlayer_trace_func_end()` 包裹核心测试逻辑**
7. **测试完成后调用 `netlayer_coverage_generate()` 生成覆盖率报告**
8. **支持按名称运行单个测试或传入 "all" 运行全部测试**
