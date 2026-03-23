# 目录分析：config/

## 目录概述

| 属性 | 值 |
|------|-----|
| 路径 | `src/mcp-bundle/config/` |
| 文件数 | 2 |
| 职责 | 定义 Bundle 的配置树和核心服务 |

## 包含文件

| 文件 | 职责 | 详细报告 |
|------|------|----------|
| `options.php` | 定义所有可配置选项及默认值 | [options.php.md](options.php.md) |
| `services.php` | 注册 MCP 核心服务（Registry, Builder, Server） | [services.php.md](services.php.md) |

## 设计模式

### 配置与服务分离模式
将配置定义（`options.php`）和服务注册（`services.php`）分成两个独立文件。

**为什么这么做：**
- **单一职责**：`options.php` 只关心"可以配置什么"，`services.php` 只关心"注册什么服务"
- **加载顺序清晰**：`configure()` 先加载 options（定义配置树），`loadExtension()` 再加载 services（使用已验证的配置值）
- **可维护性**：修改配置选项不会影响服务定义，反之亦然

### PHP DSL 模式
两个文件都使用 Symfony 的 PHP DSL（而非 YAML/XML），以获得完整的 IDE 支持和类型安全。

## 内部调用流程

```
McpBundle::configure()
    └── import('../config/options.php')
        └── 构建配置树

McpBundle::loadExtension()
    └── import('../config/services.php')
        └── 注册 mcp.registry, mcp.server.builder, mcp.server
```

## 在哪些场景下被调用

- `options.php`：Symfony 处理 Bundle 配置时（每次容器编译）
- `services.php`：Bundle 加载扩展时（每次容器编译）

## 可替换/可扩展点

1. **服务覆盖**：应用配置中可以覆盖 `services.php` 中定义的任何服务
2. **PrependExtension**：其他 Bundle 可以修改 MCP 配置
