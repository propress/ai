# 目录分析报告：config/

## 目录基本信息

| 属性 | 值 |
|------|-----|
| 路径 | `src/mcp-bundle/config/` |
| 文件数量 | 2 |
| 职责 | 定义 MCP Bundle 的配置选项和核心服务 |

## 目录内容

| 文件 | 类型 | 职责 |
|------|------|------|
| `options.php` | 配置定义 | 定义所有可用的配置选项、类型、默认值和验证规则 |
| `services.php` | 服务定义 | 定义核心 MCP 服务（Registry、Builder、Server） |

## 职责分离

这两个文件共同构成 MCP Bundle 的声明式基础设施层：

```
options.php  → 定义"用户可以配置什么"（配置 schema）
services.php → 定义"系统需要哪些服务"（服务拓扑）
```

### 调用流程

```
McpBundle::configure()
    └─→ options.php                    ← 定义配置 schema
        └─→ Symfony 验证用户配置
            └─→ 生成 $config 数组

McpBundle::loadExtension($config)
    └─→ services.php                   ← 注册核心服务
        └─→ Registry (mcp.registry)
        └─→ Builder  (mcp.server.builder)
        └─→ Server   (mcp.server)
    └─→ 根据 $config 注册额外服务       ← 由 McpBundle 处理
        └─→ Transport 相关服务
        └─→ Session 相关服务
        └─→ Debug 相关服务
```

## 设计模式

### 配置与服务分离（Separation of Concerns）
- `options.php` 专注于配置验证，不涉及任何服务创建
- `services.php` 专注于服务定义，不涉及配置验证
- 两者通过 Symfony 容器参数（parameters）进行数据传递

**好处：**
- 修改配置 schema 不影响服务定义
- 修改服务拓扑不影响配置验证
- 每个文件职责清晰，便于维护

### PHP 配置格式（而非 YAML/XML）
两个文件都使用 PHP 闭包格式定义配置和服务。

**好处：**
- 类型安全，IDE 可以提供自动补全
- 支持编程逻辑（条件判断、循环等）
- 编译时可以直接执行，无需解析器
- 与 `AbstractBundle` 的现代 Symfony 风格一致

## 可扩展性

### 添加新配置选项
1. 在 `options.php` 中添加新的节点定义
2. 在 `McpBundle::loadExtension()` 中读取新配置并设置参数或注册服务

### 添加新核心服务
1. 在 `services.php` 中添加新的服务定义
2. 如果需要用户配置，在 `options.php` 中添加对应选项

### 替换核心服务
- 使用 Symfony 的服务装饰器（`decorate`）包装现有服务
- 使用 `CompilerPass` 替换服务定义
- 例如：`TraceableRegistry` 就是在 debug 模式下装饰 `mcp.registry` 的

## 与其他模块的关系

- `options.php` 被 `McpBundle::configure()` 导入
- `services.php` 被 `McpBundle::loadExtension()` 导入
- 两个文件定义的参数和服务在整个 Bundle 的其他组件中被使用
