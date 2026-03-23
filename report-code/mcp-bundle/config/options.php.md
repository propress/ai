# 文件分析：config/options.php

## 文件基本信息

| 属性 | 值 |
|------|-----|
| 文件路径 | `src/mcp-bundle/config/options.php` |
| 命名空间 | `Symfony\Component\Config\Definition\Configurator` |
| 文件类型 | Symfony Bundle 配置定义文件 |
| 行数 | 73 行 |
| 职责 | 定义 McpBundle 的所有可配置选项及其默认值 |

## 功能概述

该文件使用 Symfony 的 `DefinitionConfigurator` 定义了 `mcp-bundle` 的完整配置树（Configuration Tree）。它描述了用户在 `config/packages/mcp.yaml` 中可以设置的所有选项、类型约束、默认值。

## 设计模式

### Tree Builder 模式（配置树构建）
Symfony 框架的标准配置机制，采用 **流式接口（Fluent Interface）+ 树形结构构建器（Tree Builder）** 的组合模式。

**为什么这么做：**
- 类型安全：每个节点有严格的类型约束（`scalarNode`, `integerNode`, `booleanNode`, `enumNode`, `arrayNode`）
- 默认值兜底：即使用户不配置任何选项，系统也能以合理默认值正常运行
- 验证前置：在容器编译阶段就能发现配置错误，而非运行时
- 文档化配置：配置树本身就是最准确的配置文档

## 配置项详细解析

### 顶层配置

| 配置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `app` | string | `'app'` | MCP 服务器应用名称，用于 `initialize` 响应中的 `serverInfo.name` |
| `version` | string | `'0.0.1'` | MCP 服务器版本号，用于 `initialize` 响应中的 `serverInfo.version` |
| `description` | string\|null | `null` | 服务器描述，MCP 2025-06-18 新增字段 |
| `website_url` | string\|null | `null` | 服务器网站 URL |
| `pagination_limit` | integer | `50` | 列表接口（tools/list, prompts/list 等）的分页大小 |
| `instructions` | string\|null | `null` | 服务器指令，告知 AI 客户端如何使用该服务器的提示 |

### icons 配置

```yaml
mcp:
  icons:
    - src: 'https://example.com/icon.png'
      mime_type: 'image/png'
      sizes: ['64x64', '128x128']
```

| 字段 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `src` | string | 是 | - | 图标 URL 或路径 |
| `mime_type` | string\|null | 否 | `null` | 图标 MIME 类型 |
| `sizes` | string[] | 否 | `['any']` | 图标尺寸列表 |

### client_transports 配置

```yaml
mcp:
  client_transports:
    stdio: false
    http: false
```

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `stdio` | boolean | `false` | 是否启用 STDIO 传输（CLI 模式） |
| `http` | boolean | `false` | 是否启用 HTTP 传输（Web 模式） |

**技巧说明：** `client_transports` 节点默认不存在（无 `addDefaultsIfNotSet()`），这意味着如果用户不配置此节点，bundle 不会注册任何传输相关服务（Command、Controller、RouteLoader 等）。这是一种 **延迟加载/按需注册** 策略——只有明确启用传输时才注册对应的服务，减少不必要的开销。

### discovery 配置

```yaml
mcp:
  discovery:
    scan_dirs: ['src']
    exclude_dirs: []
```

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `scan_dirs` | string[] | `['src']` | 自动发现 MCP 能力（Tool、Prompt 等）的扫描目录 |
| `exclude_dirs` | string[] | `[]` | 排除的目录 |

**技巧说明：** 使用 `addDefaultsIfNotSet()` 确保即使用户不配置 `discovery`，也会有合理的默认扫描路径 `['src']`。

### http 配置

```yaml
mcp:
  http:
    path: '/_mcp'
    session:
      store: 'file'
      directory: '%kernel.cache_dir%/mcp-sessions'
      cache_pool: 'cache.mcp.sessions'
      prefix: 'mcp-'
      ttl: 3600
```

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `path` | string | `'/_mcp'` | HTTP 端点路由路径 |
| `session.store` | enum | `'file'` | 会话存储类型：`file`/`memory`/`cache` |
| `session.directory` | string | `'%kernel.cache_dir%/mcp-sessions'` | 文件存储目录 |
| `session.cache_pool` | string | `'cache.mcp.sessions'` | 缓存池服务 ID |
| `session.prefix` | string | `'mcp-'` | 缓存键前缀 |
| `session.ttl` | integer | `3600` | 会话过期时间（秒），最小值 1 |

**技巧说明：**
1. `session.store` 使用 `enumNode` 限制只能是三个合法值，编译时即可验证
2. `session.ttl` 使用 `->min(1)` 防止设置为 0 或负数
3. `session.directory` 默认使用 Symfony 参数 `%kernel.cache_dir%`，与框架缓存管理保持一致

## 调用流程

```
McpBundle::configure()
    └── $definition->import('../config/options.php')
        └── DefinitionConfigurator 执行闭包
            └── 构建配置树（TreeBuilder 内部）
                └── Symfony 配置处理器合并用户配置与默认值
                    └── 验证后的配置数组传递给 McpBundle::loadExtension()
```

## 可替换/可扩展点

1. **整个配置树**：可以在另一个 Bundle 中使用 `PrependExtensionInterface` 来预设或修改 MCP 配置
2. **新增配置项**：如果需要扩展 MCP SDK 的功能，可以 fork 此文件新增配置节点
3. **会话存储类型**：`enumNode` 目前限制为 `file`/`memory`/`cache`，可扩展支持更多存储后端（如 Redis 直连、数据库等）

## 外部知识

### Symfony Configuration 机制
- Symfony 从 7.1 开始支持 `AbstractBundle` 模式，使用 `configure()` + `loadExtension()` 替代传统的 `Extension` + `Configuration` 类
- 配置树在容器编译阶段（`compile()`）被处理，此时会验证类型、应用默认值、检查必填项
- 详细文档：https://symfony.com/doc/current/bundles/configuration.html

### MCP 协议 ServerInfo
- `app` 和 `version` 对应 MCP `initialize` 响应中的 `serverInfo` 字段
- `instructions` 字段是 MCP 2025-03-26 规范中新增的，用于在 `initialize` 响应中告知客户端（AI）如何使用该服务器
- `icons` 字段用于在支持的客户端中显示服务器图标
