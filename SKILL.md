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

### 补充要求

- 原子计数器、状态位统一使用 `netlayer_api_atomic_*`，句柄类型为 `NETLAYER_ATOMIC_T`
- 极短临界区的忙等保护统一使用 `netlayer_api_spinlock_*`，句柄类型为 `NETLAYER_SPINLOCK_T`
- `NETLAYER_ATOMIC_T`、`mutex`、`msgqueue` 都是平台无关的不透明句柄，功能模块只保存和传递句柄，不直接访问平台内部对象
- `hashmap` 内部已用 spinlock 保护基础增删查遍历，但"查找后插入""遍历时联动修改其他状态"等复合操作仍须在模块层加 `netlayer_api_mutex_*` 互斥锁
- `hashmap` 的重复添加、逻辑删除、销毁释放语义都不是普通容器语义；写代码前务必结合 `references/data_structure_guide.md` 一起看
- 发送接口 `netlayer_api_send_netpkt_lower` / `netlayer_api_send_netpkt_higher` 的返回值必须检查，返回 `TRUE` 才表示发送成功

---

## 文档索引

本技能的详细规范分布在以下文档中，请按需查阅：

| 文档 | 内容说明 |
|------|---------|
| `references/coding_standards.md` | C语言编码规范：变量声明、类型使用、命名规范、注释规范、错误处理模式等 |
| `references/module_development.md` | 模块开发指南：文件组织、命名与多版本管理、开发完整流程（含设计文档模板）、交付清单、常见错误 |
| `references/testing_guide.md` | 测试用例开发规范：test.h/test.c模板、编写准则 |
| `references/data_structure_guide.md` | 数据结构选型规范与hashmap完整使用指南（选型原则、API行为、推荐写法、反例） |
| `references/api_reference.md` | Base框架API接口完整参考：所有函数签名、结构体定义、使用示例 |

---

## 完整示例

查阅项目中提供的排序器模块代码（`sorter_window/`目录下的api.h、api.c、core.h、core.c、test.h、test.c），它是一个符合所有规范的标准实现范例，展示了完整的模块生命周期管理、API结构体填充、参数控制注册、统计信息管理和测试用例编写模式。
