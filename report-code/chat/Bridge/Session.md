# Bridge/Session/MessageStore.php 分析报告

## 文件基本信息

| 属性 | 值 |
|------|------|
| 文件路径 | `src/chat/src/Bridge/Session/MessageStore.php` |
| 命名空间 | `Symfony\AI\Chat\Bridge\Session` |
| Composer 包名 | `symfony/ai-session-message-store` |
| 类型 | 最终类（final class） |
| 实现接口 | `ManagedStoreInterface`, `MessageStoreInterface` |
| 作者 | Christopher Hertel |
| 行数 | 53 行 |

## 功能描述

`Session\MessageStore` 通过 Symfony HttpFoundation 的 `SessionInterface` 实现消息持久化。它将对话消息存储在**当前用户的 HTTP 会话**中，非常适合 Web 应用中与特定用户绑定的对话场景。

这是**最适合 Web 应用的 Bridge**，因为它天然地将对话与用户会话关联，无需额外的用户标识管理。

## 类定义

```php
final class MessageStore implements ManagedStoreInterface, MessageStoreInterface
{
    private readonly SessionInterface $session;

    public function __construct(
        RequestStack $requestStack,
        private readonly string $sessionKey = 'messages',
    ) {
        $this->session = $requestStack->getSession();
    }
}
```

### 构造函数参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `$requestStack` | `RequestStack` | - | Symfony 请求栈，用于获取当前 Session |
| `$sessionKey` | `string` | `'messages'` | Session 中的存储键名 |

**关键设计**: 通过 `RequestStack` 获取 `Session`，而非直接注入 `SessionInterface`。这是因为：
- Session 的获取依赖于当前请求上下文
- `RequestStack` 是 Symfony 推荐的获取请求相关服务的方式
- 在 Symfony DI 容器中，`RequestStack` 是可注入的单例服务

## 方法详解

### `setup(array $options = []): void`

```php
public function setup(array $options = []): void
{
    $this->session->set($this->sessionKey, new MessageBag());
}
```

| 输入 | 输出 | 行为 |
|------|------|------|
| `array $options`（忽略） | `void` | 在 Session 中设置空 MessageBag |

### `save(MessageBag $messages): void`

```php
public function save(MessageBag $messages): void
{
    $this->session->set($this->sessionKey, $messages);
}
```

| 输入 | 输出 | 行为 |
|------|------|------|
| `MessageBag $messages` | `void` | 直接存入 Session |

### `load(): MessageBag`

```php
public function load(): MessageBag
{
    return $this->session->get($this->sessionKey, new MessageBag());
}
```

| 输入 | 输出 | 行为 |
|------|------|------|
| 无 | `MessageBag` | 从 Session 加载，不存在则返回空 MessageBag |

**技巧**: `Session::get()` 的第二个参数是默认值，优雅地处理了键不存在的情况。

### `drop(): void`

```php
public function drop(): void
{
    $this->session->remove($this->sessionKey);
}
```

| 输入 | 输出 | 行为 |
|------|------|------|
| 无 | `void` | 从 Session 移除键 |

## 设计模式

### 1. 适配器模式（Adapter Pattern）

将 Symfony Session API 适配为 `MessageStoreInterface`。

### 2. 整体序列化策略

与 Cache Bridge 相同，直接序列化整个 `MessageBag` 对象，不需要 `MessageNormalizer`。Session 的序列化由 PHP 的 session 处理器负责。

### 3. 作用域绑定模式（Scoped Binding）

对话自动与用户会话绑定，天然实现了多用户隔离：
- 用户 A 的 Session 只包含用户 A 的对话
- 用户 B 的 Session 只包含用户 B 的对话
- 无需额外的用户标识逻辑

## 与通用实现的差异

| 方面 | Session Bridge | 通用 Bridge |
|------|---------------|-------------|
| 用户隔离 | ✅ 自动（基于 Session ID） | ❌ 需要手动实现 |
| 持久化范围 | 会话级别 | 全局 |
| 适用场景 | Web 应用 | 通用 |
| 是否需要 Normalizer | ❌ | 大多数 ✅ |
| 序列化方式 | PHP Session 序列化 | JSON/PHP |

## 外部知识：Symfony Session

### Session 机制

Symfony Session 基于 PHP 的 Session 机制，提供了对象化的接口：

- **存储**: 默认使用文件系统（`/var/sessions/`），也支持数据库、Redis 等
- **生命周期**: 与 Session ID（通常存储在 Cookie 中）关联
- **过期**: 由 `session.gc_maxlifetime` PHP 配置控制（默认 1440 秒 = 24分钟）

### Session 存储选项

| 存储 | 适用场景 | 配置 |
|------|----------|------|
| 文件系统 | 开发/单机 | 默认 |
| Redis | 分布式/高性能 | `framework.session.handler_id: 'redis://...'` |
| 数据库 | 持久化 | PdoSessionHandler |
| Memcached | 高性能 | NativeSessionHandler |

### 限制

| 限制 | 说明 |
|------|------|
| Session 大小限制 | PHP 默认 Session 大小无硬限制，但大 Session 影响性能 |
| 并发写入 | PHP Session 默认锁定文件，同一用户的并发请求会等待 |
| 非 Web 环境 | CLI/API（无 Cookie）场景不适用 |
| Session 过期 | 长对话可能因 Session 过期丢失 |

## 依赖关系

| 依赖 | 来源 | 版本 |
|------|------|------|
| `symfony/http-foundation` | Composer require | `^7.3\|^8.0` |
| `symfony/ai-chat` | 本模块 | `^0.6` |

## 可替换性与扩展性

### 可替换

通过配置 Symfony 的 Session 存储后端，可以改变底层持久化：

```yaml
# config/packages/framework.yaml
framework:
    session:
        handler_id: 'redis://localhost'  # 使用 Redis 存储 Session
```

### 可能的扩展方向

1. **多对话支持**: 使用不同的 sessionKey 管理多个独立对话
2. **Session Flash**: 利用 Symfony Flash 消息实现一次性对话
3. **Session 大小监控**: 添加对大对话的警告
4. **自定义序列化**: 使用 JSON 而非 PHP 序列化存储，便于调试
