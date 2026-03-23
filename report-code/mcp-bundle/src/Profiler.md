# 目录分析报告：src/Profiler/

## 目录基本信息

| 属性 | 值 |
|------|-----|
| 路径 | `src/mcp-bundle/src/Profiler/` |
| 文件数量 | 2 |
| 职责 | 为 Symfony Web Profiler 提供 MCP 服务器能力的调试和可视化支持 |

## 目录内容

| 文件 | 类 | 设计模式 | 职责 |
|------|-----|----------|------|
| `TraceableRegistry.php` | `TraceableRegistry` | 装饰器模式 | 透明代理 Registry，为 DataCollector 提供数据访问 |
| `DataCollector.php` | `DataCollector` | Data Collector 模式 | 收集并格式化 MCP 能力数据用于 Profiler 展示 |

## 在整体架构中的位置

```
仅在 debug 模式下激活：

Registry (原始)
    ↑ 被装饰
TraceableRegistry (装饰器)
    ↑ 被引用
DataCollector (数据收集器)
    ↑ 被 Profiler 调用
Web Profiler 面板
    ↑ 使用 Twig 模板
templates/data_collector.html.twig
```

## 内部调用流程

```
1. 容器编译阶段：
   McpBundle::loadExtension()
   ├─ 检查 kernel.debug = true
   ├─ 注册 TraceableRegistry 装饰 mcp.registry
   └─ 注册 DataCollector (标签: data_collector)

2. 请求处理阶段：
   所有 MCP 相关代码使用 mcp.registry
   └─ 实际调用的是 TraceableRegistry
       └─ 内部委托给原始 Registry

3. Profiler 数据收集阶段：
   Symfony Profiler
   ├─ 调用 DataCollector::collect() → 空操作
   └─ 调用 DataCollector::lateCollect()
       └─ 从 TraceableRegistry 读取所有能力数据
       └─ 规范化为 PHP 数组存入 $this->data

4. Profiler 面板渲染阶段：
   data_collector.html.twig
   ├─ collector.tools → DataCollector::getTools()
   ├─ collector.prompts → DataCollector::getPrompts()
   ├─ collector.resources → DataCollector::getResources()
   └─ collector.resourceTemplates → DataCollector::getResourceTemplates()

5. 请求结束阶段（长驻进程）：
   Symfony ServicesResetter
   └─ 调用 TraceableRegistry::reset()
       └─ 调用 Registry::clear()
```

## 设计模式

### 1. 装饰器 + 观察者组合（Decorator + Observer Combination）
TraceableRegistry 通过装饰器模式透明代理所有 Registry 操作，DataCollector 作为观察者从装饰器中提取状态数据。

**好处：**
- 零侵入原始 Registry 代码
- debug 模式关闭时完全不存在任何性能开销
- 两个类各自职责清晰

### 2. 条件装饰（Conditional Decoration）
```php
if ($builder->getParameter('kernel.debug')) {
    // 注册 TraceableRegistry 和 DataCollector
}
```

**好处：**
- 生产环境下没有任何额外的服务实例化
- 没有装饰器的代理开销
- 没有 DataCollector 的数据收集开销

## 与其他目录的关系

| 关联 | 说明 |
|------|------|
| `config/services.php` | 定义被装饰的 `mcp.registry` 服务 |
| `McpBundle.php` | 条件注册 Profiler 组件 |
| `templates/` | DataCollector 使用的 Twig 模板 |

## 与 Twig 模板的数据流

```
DataCollector.$data
├─ tools[] → tools.html.twig
│   └─ name, description, inputSchema
├─ prompts[] → prompts.html.twig
│   └─ name, description, arguments[]
├─ resources[] → resources.html.twig
│   └─ uri, name, description, mimeType
└─ resourceTemplates[] → resource_templates.html.twig
    └─ uriTemplate, name, description, mimeType
```

## 可扩展性

### 可扩展方向
- 在 TraceableRegistry 中添加方法调用追踪（计时、计数）
- 在 DataCollector 中添加运行时数据（MCP 请求/响应日志）
- 添加新的 Profiler 面板 tab（如 MCP 会话信息）

### 可替换
- 可以用自定义装饰器替换 TraceableRegistry
- 可以用自定义 DataCollector 替换，提供不同的数据展示
