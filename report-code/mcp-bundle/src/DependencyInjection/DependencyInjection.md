# 目录分析：src/DependencyInjection/

## 目录概述

| 属性 | 值 |
|------|-----|
| 路径 | `src/mcp-bundle/src/DependencyInjection/` |
| 文件数 | 1 |
| 职责 | DI 容器编译期的服务聚合处理 |

## 包含文件

| 文件 | 类 | 职责 | 详细报告 |
|------|-----|------|----------|
| `McpPass.php` | `McpPass` | 收集 MCP 标记服务创建 ServiceLocator | [McpPass.php.md](McpPass.php.md) |

## 设计模式

### 编译器传递模式（Compiler Pass）
这是 Symfony DI 的核心扩展机制。`McpPass` 在容器编译阶段运行，聚合所有带 MCP 标签的服务。

**为什么单独一个目录：**
- Symfony Bundle 约定将 DI 相关逻辑（Extension、Configuration、CompilerPass）放在 `DependencyInjection` 目录
- 与运行时代码（Controller、Command 等）分离
- 这些代码只在容器编译时执行，不影响运行时性能

## 内部调用流程

```
McpBundle::build()
    └── addCompilerPass(new McpPass())
        └── 容器编译时 McpPass::process()
            ├── findTaggedServiceIds('mcp.tool') → [服务列表]
            ├── findTaggedServiceIds('mcp.prompt') → [服务列表]
            ├── findTaggedServiceIds('mcp.resource') → [服务列表]
            ├── findTaggedServiceIds('mcp.resource_template') → [服务列表]
            ├── ServiceLocatorTagPass::register() → ServiceLocator
            └── builder->setContainer(ServiceLocator)
```

## 在哪些场景下被调用

- 容器编译时（应用启动/缓存预热/缓存清除后首次请求）

## 与其他模块的关系

- **输入**：依赖 McpBundle 中注册的属性自动配置产生的标签
- **输出**：将 ServiceLocator 注入到 `mcp.server.builder`，最终被 MCP SDK 的 Server 使用

## 可替换/可扩展点

1. 可以注册额外的 CompilerPass 在 McpPass 之后修改服务定义
2. 可以通过覆盖标签机制改变哪些服务被收集
