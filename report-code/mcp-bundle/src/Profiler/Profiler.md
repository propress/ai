# 目录分析：src/Profiler/

## 目录概述

| 属性 | 值 |
|------|-----|
| 路径 | `src/mcp-bundle/src/Profiler/` |
| 文件数 | 2 |
| 职责 | Symfony Web Profiler 集成，提供 MCP 能力的调试面板 |

## 包含文件

| 文件 | 类 | 职责 | 详细报告 |
|------|-----|------|----------|
| `TraceableRegistry.php` | `TraceableRegistry` | Registry 装饰器，为 Profiler 提供数据访问 | [TraceableRegistry.php.md](TraceableRegistry.php.md) |
| `DataCollector.php` | `DataCollector` | 收集 MCP 能力数据供 Profiler 展示 | [DataCollector.php.md](DataCollector.php.md) |

## 设计模式

### 装饰器 + 数据收集器协作模式

```
TraceableRegistry [装饰器]
    │ 包装真实 Registry
    │ 提供数据访问接口
    ▼
DataCollector [数据收集器]
    │ 持有 TraceableRegistry 引用
    │ lateCollect() 读取数据
    │ 序列化存储
    ▼
Twig 模板 [展示层]
    │ 从 DataCollector 读取数据
    │ 渲染 Profiler 面板
```

**为什么需要两个类配合：**
1. `TraceableRegistry` 必须实现 `RegistryInterface`——它是 Registry 的替身，被注入到 MCP Server 中
2. `DataCollector` 必须继承 `AbstractDataCollector`——它是 Profiler 系统的标准接口
3. 两者职责不同，不能合并

**为什么单独一个目录：**
- Profiler 功能是开发调试功能，与核心业务逻辑分离
- 这两个类只在 `kernel.debug = true` 时注册
- 便于整体禁用或替换调试功能

## 内部调用流程

```
McpBundle::loadExtension() [debug=true]
    ├── TraceableRegistry 装饰 mcp.registry
    └── DataCollector 注入 TraceableRegistry
        │
MCP 请求处理
    └── Server 使用 Registry（实际是 TraceableRegistry）
        │
请求结束 → Profiler 收集
    ├── DataCollector::collect() [空]
    └── DataCollector::lateCollect()
        ├── TraceableRegistry::getTools() → Page
        ├── TraceableRegistry::getPrompts() → Page
        ├── TraceableRegistry::getResources() → Page
        └── TraceableRegistry::getResourceTemplates() → Page
        │
Profiler 面板
    └── Twig 模板渲染数据
        │
请求重置
    └── TraceableRegistry::reset() → clear()
```

## 在哪些场景下被调用

- 仅在开发环境（`kernel.debug = true`）
- 每次 HTTP 请求处理时（数据收集）
- 用户查看 Profiler 面板时（数据展示）

## 与其他模块的关系

- **装饰**：TraceableRegistry 装饰 `mcp.registry`（核心服务）
- **展示**：DataCollector 使用 `templates/` 目录下的 Twig 模板
- **仅影响 debug 模式**：生产环境完全绕过

## 可替换/可扩展点

1. 可以在 TraceableRegistry 中添加调用追踪（计时、计数）
2. 可以自定义 DataCollector 收集更多信息
3. 可以覆盖 Twig 模板自定义面板外观
