# 文件分析报告：config/options.php

## 文件基本信息

| 属性 | 值 |
|------|-----|
| 文件路径 | `src/mcp-bundle/config/options.php` |
| 命名空间 | `Symfony\Component\Config\Definition\Configurator` |
| 类型 | Symfony Bundle 配置定义文件 |
| 职责 | 定义 MCP Bundle 的所有可配置选项及其默认值 |

## 功能描述

此文件使用 Symfony 的 `DefinitionConfigurator` 定义了 MCP Bundle 的完整配置树（Configuration Tree）。它声明了所有可以在 `config/packages/mcp.yaml` 或 `config/packages/mcp.php` 中使用的配置键，包括数据类型、默认值和验证规则。

## 设计模式

### Configuration Tree Builder（配置树构建器模式）
这是 Symfony DependencyInjection 组件中标准的配置定义方式，属于 **Builder 模式** 的变体。

**好处：**
- 提供强类型的配置定义，确保配置值的类型安全
- 自动验证用户输入的配置值
- 支持默认值，用户只需覆盖需要自定义的部分
- 通过 TreeBuilder 生成的配置 schema 可用于 IDE 自动补全

### 闭包返回模式（Closure Return Pattern）
文件使用 `return static function (DefinitionConfigurator $configurator): void` 这种闭包返回的形式，而非类文件。

**好处：**
- 更轻量级，不需要创建完整的 Configuration 类
- 与 `AbstractBundle` 的 `configure()` 方法配合使用时，代码更简洁
- 避免了额外的类加载开销

## 完整配置选项定义

### 顶级选项

| 配置键 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `app` | scalar (string) | `'app'` | MCP 服务器应用名称 |
| `version` | scalar (string) | `'0.0.1'` | MCP 服务器版本号 |
| `description` | scalar (string\|null) | `null` | MCP 服务器描述 |
| `website_url` | scalar (string\|null) | `null` | MCP 服务器网站 URL |
| `pagination_limit` | integer | `50` | 分页限制数量 |
| `instructions` | scalar (string\|null) | `null` | MCP 服务器使用说明 |

### icons 配置节点

```yaml
mcp:
  icons:
    - src: 'https://example.com/icon.png'
      mime_type: 'image/png'
      sizes: ['64x64', '128x128']
```

| 子键 | 类型 | 默认值 | 是否必须 | 说明 |
|------|------|--------|----------|------|
| `src` | scalar (string) | - | **是** | 图标 URL 或路径 |
| `mime_type` | scalar (string\|null) | `null` | 否 | MIME 类型 |
| `sizes` | scalar[] | `['any']` | 否 | 尺寸列表 |

### client_transports 配置节点

```yaml
mcp:
  client_transports:
    stdio: false
    http: false
```

| 子键 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `stdio` | boolean | `false` | 是否启用 STDIO 传输 |
| `http` | boolean | `false` | 是否启用 HTTP 传输 |

**重要：** `client_transports` 节点是可选的。如果未定义，则不会注册任何传输相关的服务（Command、Controller、RouteLoader）。

### discovery 配置节点

```yaml
mcp:
  discovery:
    scan_dirs: ['src']
    exclude_dirs: []
```

| 子键 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `scan_dirs` | scalar[] | `['src']` | 自动发现扫描目录列表 |
| `exclude_dirs` | scalar[] | `[]` | 排除目录列表 |

`addDefaultsIfNotSet()` 确保 discovery 节点即使未配置也有默认值。

### http 配置节点

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

| 子键 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `path` | scalar (string) | `'/_mcp'` | HTTP 端点路径 |
| `session.store` | enum | `'file'` | 会话存储类型 (`file`\|`memory`\|`cache`) |
| `session.directory` | scalar (string) | `'%kernel.cache_dir%/mcp-sessions'` | 文件存储目录（仅 file 模式） |
| `session.cache_pool` | scalar (string) | `'cache.mcp.sessions'` | 缓存池服务 ID（仅 cache 模式） |
| `session.prefix` | scalar (string) | `'mcp-'` | 缓存键前缀（仅 cache 模式） |
| `session.ttl` | integer (min: 1) | `3600` | 会话 TTL（秒） |

## 技巧分析

### 1. `addDefaultsIfNotSet()` 的使用
```php
->arrayNode('discovery')
    ->addDefaultsIfNotSet()
```
**为什么这么做：** 用户可以完全不配置 `discovery` 节点，但系统仍然可以获取到默认值 `['src']`。这是 Symfony 配置系统的标准做法，保证了零配置即可运行。

### 2. `enumNode` 约束
```php
->enumNode('store')->values(['file', 'memory', 'cache'])->defaultValue('file')->end()
```
**为什么这么做：** 限制 `store` 只能是三个有效值之一，在配置加载阶段就做验证，而非运行时。提前失败（Fail Fast），有利于开发者快速定位配置错误。

### 3. `integerNode` 的 `min(1)` 约束
```php
->integerNode('ttl')->min(1)->defaultValue(3600)->end()
```
**为什么这么做：** TTL 不允许为 0 或负数，避免出现会话立即过期或语义不明确的情况。

### 4. 层级默认值设计
HTTP 和 Session 配置都有 `addDefaultsIfNotSet()`，这意味着：
- 用户只需声明 `client_transports.http: true` 即可启用 HTTP，所有 HTTP 子配置自动使用默认值
- 可以只覆盖部分配置，其余使用默认值
- 减少了配置的模板代码（boilerplate）

## 被调用场景

1. **`McpBundle::configure()`** → 通过 `$definition->import('../config/options.php')` 导入
2. Symfony 框架在编译容器时自动调用，用于验证 `config/packages/mcp.yaml` 中的配置
3. 生成的配置数组传递给 `McpBundle::loadExtension()` 的 `$config` 参数

## 可扩展性

- 可以通过 Symfony 的 `prepend` 机制在其他 Bundle 中修改默认值
- 配置选项的增减直接影响 `McpBundle::loadExtension()` 中对应的处理逻辑
- 新增配置选项只需在此文件中添加节点定义，并在 `McpBundle` 中处理对应的值
