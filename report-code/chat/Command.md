# Command 目录分析报告

## 目录基本信息

| 属性 | 值 |
|------|------|
| 目录路径 | `src/chat/src/Command/` |
| 命名空间 | `Symfony\AI\Chat\Command` |
| 文件数量 | 2 个 |
| 职责 | 提供消息存储管理的 CLI 命令 |

## 目录结构

```
Command/
├── SetupStoreCommand.php   # 初始化存储基础设施
└── DropStoreCommand.php    # 清除存储数据
```

## 功能概述

此目录包含两个 Symfony Console 命令，它们是 `ManagedStoreInterface` 两个方法（`setup()` 和 `drop()`）的 **CLI 入口点**。它们将存储基础设施管理能力暴露给运维人员和开发者。

## 设计思路

### 为什么需要 CLI 命令

1. **运维分离**: 存储基础设施的创建/销毁通常是运维操作，不应嵌入业务代码
2. **部署流程**: 在部署管道中可以执行 `ai:message-store:setup` 来准备基础设施
3. **开发便利**: 开发者可以快速重置测试数据（`ai:message-store:drop --force`）

### 命令对称性

两个命令构成了完整的**生命周期管理对**：

```
setup ←→ drop
创建      销毁
```

| 方面 | SetupStoreCommand | DropStoreCommand |
|------|-------------------|------------------|
| 安全级别 | 低风险（幂等） | 高风险（不可逆） |
| 安全机制 | 无 | 需要 `--force` |
| 调用方法 | `setup()` | `drop()` |
| 典型场景 | 部署时 | 维护/重置时 |

## 共同的设计模式

### 1. 服务定位器模式（Service Locator Pattern）

两个命令都注入 `ServiceLocator<ManagedStoreInterface>`，按名称获取 store 服务：

```php
public function __construct(
    private readonly ServiceLocator $stores,
)
```

**好处**:
- 懒加载：只实例化实际使用的 store
- 自动发现：新注册的 store 自动出现在可用列表中
- 自动补全：`complete()` 方法利用 ServiceLocator 提供 shell 补全

### 2. 早期验证模式（Fail Fast）

两个命令都在 `initialize()` 中验证参数，在 `execute()` 之前报告错误。

### 3. 异常包装模式（Exception Wrapping）

两个命令都在 `execute()` 中捕获底层异常，包装为 Chat 模块的 `RuntimeException`。

## 调用流程图

```
                           CLI 命令行
                               │
            ┌──────────────────┴──────────────────┐
            │                                      │
    ai:message-store:setup            ai:message-store:drop --force
            │                                      │
            ▼                                      ▼
    ┌──────────────────┐              ┌──────────────────┐
    │ initialize()     │              │ initialize()     │
    │ - 验证 store 存在 │              │ - 验证 store 存在 │
    │ - 验证支持 setup  │              │ - 验证支持 drop   │
    └────────┬─────────┘              └────────┬─────────┘
             │                                  │
             ▼                                  ▼
    ┌──────────────────┐              ┌──────────────────┐
    │ execute()        │              │ execute()        │
    │ - 获取 store 实例 │              │ - 检查 --force   │
    │ - 调用 setup()   │              │ - 获取 store 实例 │
    │ - 输出结果        │              │ - 调用 drop()    │
    └────────┬─────────┘              │ - 输出结果        │
             │                        └────────┬─────────┘
             ▼                                  ▼
    ManagedStoreInterface::setup()    ManagedStoreInterface::drop()
             │                                  │
             ▼                                  ▼
    具体的 Bridge 实现                  具体的 Bridge 实现
    (创建表/索引/命名空间等)            (清除数据/删除文档等)
```

## 与其他模块的关系

| 去向 | 模块/组件 | 说明 |
|------|-----------|------|
| `ServiceLocator` | `symfony/dependency-injection` | 服务定位和懒加载 |
| `Command/AsCommand` | `symfony/console` | Console 命令基础设施 |
| `ManagedStoreInterface` | Chat 本模块 | 存储管理契约 |
| 具体 Bridge | Chat Bridge/ | 实际的存储操作 |

## 可扩展性

1. **自动发现新 Store**: 在 DI 容器中注册新的 `ManagedStoreInterface` 实现，命令自动支持
2. **添加新命令**: 如 `ai:message-store:info` 查看存储状态
3. **添加新选项**: 如 `--dry-run` 预览操作效果
