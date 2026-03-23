# 目录分析报告：src/DependencyInjection/

## 目录基本信息

| 属性 | 值 |
|------|-----|
| 路径 | `src/mcp-bundle/src/DependencyInjection/` |
| 文件数量 | 1 |
| 职责 | Symfony 依赖注入容器的编译阶段处理 |

## 目录内容

| 文件 | 类 | 职责 |
|------|-----|------|
| `McpPass.php` | `McpPass` | 收集所有标记的 MCP 服务，创建 ServiceLocator 注入到 Builder |

## 在整体架构中的位置

```
Symfony 容器编译流程：

1. McpBundle::loadExtension()
   └─ 注册核心服务（Registry, Builder, Server）
   └─ 注册属性自动配置（McpTool, McpPrompt 等）
   └─ 注册接口自动配置（LoaderInterface 等）

2. 其他 Bundle 注册服务
   └─ 应用代码中带有 MCP 属性的类被扫描和注册

3. McpPass::process()  ← 本目录的核心
   └─ 收集所有 MCP 标签服务
   └─ 创建 ServiceLocator
   └─ 注入到 Builder
```

DependencyInjection 目录在编译流程中扮演"最后一公里"的角色，将分散在各处注册的 MCP 服务汇总并注入到构建器中。

## 设计模式

### Compiler Pass 模式
这是 Symfony DI 组件的核心扩展机制。McpPass 在容器编译阶段运行，对已注册的服务进行后处理。

```
Bundle Extension (注册服务)
    ↓
User Config (自动发现/自动配置)
    ↓
Compiler Passes (后处理)  ← McpPass 在此执行
    ↓
Compiled Container (最终产物)
```

## 与其他目录的关系

| 关联 | 说明 |
|------|------|
| `McpBundle.php` | 注册 McpPass 为 Compiler Pass |
| `config/services.php` | 定义 McpPass 操作的目标服务（`mcp.server.builder`） |
| 应用代码 | 带 MCP 标签的服务被 McpPass 收集 |

## 可扩展性

- 可以添加更多 Compiler Pass 处理不同的编译阶段需求
- 可以在 McpPass 中添加服务验证、排序等逻辑
- 其他 Bundle 可以注册自己的 Compiler Pass 来扩展 MCP 服务集合
