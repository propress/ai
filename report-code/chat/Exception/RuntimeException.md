# RuntimeException.php 分析报告

## 文件基本信息

| 属性 | 值 |
|------|------|
| 文件路径 | `src/chat/src/Exception/RuntimeException.php` |
| 命名空间 | `Symfony\AI\Chat\Exception` |
| 类型 | 类（Class） |
| 继承 | `\RuntimeException` |
| 实现 | `ExceptionInterface` |
| 作者 | Guillaume Loulier |
| 行数 | 19 行 |

## 功能描述

Chat 模块专用的 `RuntimeException`，用于表示**运行时错误**——只有在运行时才能检测到的错误，如数据库连接失败、外部 API 不可达、存储操作失败等。

## 类定义

```php
class RuntimeException extends \RuntimeException implements ExceptionInterface
{
}
```

## 设计模式

与其他异常类相同，采用**双重继承模式**。

## 使用场景

| 使用位置 | 场景 |
|----------|------|
| `Command/SetupStoreCommand::initialize()` | 指定的 store 不存在或不支持 setup 操作 |
| `Command/SetupStoreCommand::execute()` | 执行 setup 过程中发生错误，包装原始异常 |
| `Command/DropStoreCommand::initialize()` | 指定的 store 不存在或不支持 drop 操作 |
| `Command/DropStoreCommand::execute()` | 执行 drop 过程中发生错误，包装原始异常 |
| `Bridge/Meilisearch/MessageStore::__construct()` | 缺少 `symfony/clock` 依赖时 |
| `Bridge/SurrealDb/MessageStore::authenticate()` | SurrealDB 认证响应缺少 token 时 |

**关键用途**: `RuntimeException` 是 Chat 模块中使用最广泛的异常类型，主要用于：

1. **命令层错误处理**: 两个 CLI 命令在验证和执行阶段都依赖此异常。
2. **外部服务交互失败**: 如 SurrealDB 认证失败。
3. **依赖检查失败**: 如缺少必要的 Composer 包。

## 异常链（Exception Chaining）

在 Command 层中，`RuntimeException` 的使用展示了**异常链模式**：

```php
throw new RuntimeException(
    sprintf('An error occurred while setting up the "%s" message store: ', $storeName) . $e->getMessage(),
    previous: $e
);
```

**为什么要这么做**:
- `previous: $e` 保留了原始异常的完整堆栈信息，便于调试。
- 外层消息提供了业务上下文（"正在 setup 哪个 store"），而内层消息保留了技术细节。
- 这样在生产环境中可以展示友好的错误信息，同时在日志中保留完整的调试信息。

## 可替换性与扩展性

- **可扩展**: 可以创建特定的运行时异常子类，如 `ConnectionException`、`AuthenticationException` 等。
- **可在 Bridge 中替换**: 各 Bridge 实现可以抛出自己的 `RuntimeException` 子类来提供更精确的错误信息。
