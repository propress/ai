# Bridge/SurrealDb/MessageStore.php 分析报告

## 文件基本信息

| 属性 | 值 |
|------|------|
| 文件路径 | `src/chat/src/Bridge/SurrealDb/MessageStore.php` |
| 命名空间 | `Symfony\AI\Chat\Bridge\SurrealDb` |
| Composer 包名 | `symfony/ai-surreal-db-message-store` |
| 类型 | 最终类（final class） |
| 实现接口 | `ManagedStoreInterface`, `MessageStoreInterface` |
| 作者 | Guillaume Loulier |
| 行数 | 145 行 |

## 功能描述

`SurrealDb\MessageStore` 使用 **SurrealDB** 多模型数据库作为消息存储后端。SurrealDB 是一个新型数据库，同时支持文档、图、关系等多种数据模型。

**独特之处**: 这是唯一需要**手动认证并管理认证 Token**的 Bridge，因为 SurrealDB 使用基于 JWT 的认证机制。

## 类定义

```php
final class MessageStore implements ManagedStoreInterface, MessageStoreInterface
{
    private string $authenticationToken = '';

    public function __construct(
        private readonly HttpClientInterface $httpClient,
        private readonly string $endpointUrl,
        private readonly string $user,
        #[\SensitiveParameter] private readonly string $password,
        private readonly string $namespace,
        private readonly string $database,
        private readonly SerializerInterface&NormalizerInterface&DenormalizerInterface $serializer = new Serializer([...]),
        private readonly string $table = '_message_store_surrealdb',
        private readonly bool $isNamespacedUser = false,
    ) {
    }
}
```

### 构造函数参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `$httpClient` | `HttpClientInterface` | - | Symfony HTTP 客户端 |
| `$endpointUrl` | `string` | - | SurrealDB 服务 URL |
| `$user` | `string` | - | 用户名 |
| `$password` | `string` | - | 密码（敏感参数） |
| `$namespace` | `string` | - | SurrealDB 命名空间 |
| `$database` | `string` | - | SurrealDB 数据库名 |
| `$serializer` | 交集类型 | 默认 Serializer | 消息序列化器 |
| `$table` | `string` | `'_message_store_surrealdb'` | 表名 |
| `$isNamespacedUser` | `bool` | `false` | 是否为命名空间级用户 |

**`$isNamespacedUser` 参数**: SurrealDB 支持不同级别的用户：
- 根级用户（Root User）：访问所有命名空间和数据库
- 命名空间级用户：仅访问特定命名空间和数据库
- 此参数决定认证时是否传递 `ns` 和 `db` 参数

### 实例变量

| 变量 | 类型 | 说明 |
|------|------|------|
| `$authenticationToken` | `string` | JWT Token，认证后缓存 |

## 方法详解

### `setup(array $options = []): void`

```php
public function setup(array $options = []): void
{
    if ([] !== $options) {
        throw new InvalidArgumentException('No supported options.');
    }
}
```

| 输入 | 输出 | 行为 |
|------|------|------|
| `array $options`（必须为空） | `void` | **空操作**——SurrealDB 不需要预创建表 |

**为什么是空实现**: SurrealDB 的 schemaless 模式下，表会在第一次写入时自动创建，不需要预先 setup。

### `drop(): void`

```php
public function drop(): void
{
    $this->request('DELETE', sprintf('key/%s', $this->table));
}
```

| 输入 | 输出 | 行为 |
|------|------|------|
| 无 | `void` | DELETE 请求删除表中所有记录 |

### `save(MessageBag $messages): void`

```php
public function save(MessageBag $messages): void
{
    foreach ($messages->getMessages() as $message) {
        $this->request('POST', sprintf('key/%s', $this->table), $this->serializer->normalize($message));
    }
}
```

| 输入 | 输出 | 行为 |
|------|------|------|
| `MessageBag $messages` | `void` | **逐条**插入消息 |

**注意**: 与其他 Bridge 不同，SurrealDB Bridge **逐条发送请求**而非批量操作。这在消息数量多时可能导致性能问题。每条消息一个 HTTP 请求。

### `load(): MessageBag`

```php
public function load(): MessageBag
{
    $messages = $this->request('GET', sprintf('key/%s', $this->table), []);
    return new MessageBag(...array_map(
        fn (array $message): MessageInterface => $this->serializer->denormalize($message, MessageInterface::class),
        $messages[0]['result'],
    ));
}
```

| 输入 | 输出 | 行为 |
|------|------|------|
| 无 | `MessageBag` | GET 请求获取所有记录并反序列化 |

**`$messages[0]['result']` 结构**: SurrealDB 的响应格式是嵌套的——结果在 `[0]['result']` 中。

### `request(string $method, string $endpoint, array $payload = []): array`（私有方法）

```php
private function request(string $method, string $endpoint, array $payload = []): array
{
    $this->authenticate();
    // ... 构建并发送请求
}
```

**请求头**:
```php
'headers' => [
    'Accept' => 'application/json',
    'Content-Type' => 'application/json',
    'Surreal-NS' => $this->namespace,
    'Surreal-DB' => $this->database,
    'Authorization' => sprintf('Bearer %s', $this->authenticationToken),
],
```

SurrealDB 使用自定义的 HTTP 头来指定命名空间和数据库。

### `authenticate(): void`（私有方法）

```php
private function authenticate(): void
{
    if ('' !== $this->authenticationToken) {
        return;
    }

    $authenticationPayload = [
        'user' => $this->user,
        'pass' => $this->password,
    ];

    if ($this->isNamespacedUser) {
        $authenticationPayload['ns'] = $this->namespace;
        $authenticationPayload['db'] = $this->database;
    }

    $authenticationResponse = $this->httpClient->request('POST', sprintf('%s/signin', $this->endpointUrl), [
        'headers' => ['Accept' => 'application/json'],
        'json' => $authenticationPayload,
    ]);

    $payload = $authenticationResponse->toArray();

    if (!\array_key_exists('token', $payload)) {
        throw new RuntimeException('The SurrealDB authentication response does not contain a token.');
    }

    $this->authenticationToken = $payload['token'];
}
```

**认证流程**:
1. 检查是否已有 Token（惰性认证）
2. 构建认证 payload（根级用户 vs 命名空间用户）
3. POST 请求到 `/signin` 端点
4. 从响应中提取 JWT Token
5. 缓存 Token 供后续请求使用

**惰性认证模式（Lazy Authentication）**: Token 仅在首次请求时获取，之后复用。这避免了在构造函数中就建立连接（可能尚不需要）。

**缺陷**: Token 没有过期处理机制。如果 Token 过期，后续请求会失败。

## 设计模式

### 1. 惰性认证模式（Lazy Authentication）

认证延迟到实际需要时执行，Token 被缓存复用。

### 2. 状态缓存模式（State Caching）

`$authenticationToken` 作为实例状态缓存，避免重复认证。

### 3. 条件装载模式（Conditional Payload）

根据 `$isNamespacedUser` 决定认证请求的 payload 内容。

## 外部知识：SurrealDB

### SurrealDB 概述

SurrealDB 是一个多模型数据库：
- 支持文档、图、关系等多种数据模型
- 支持实时查询和 WebSocket
- 内置认证和权限系统
- Schemaless 和 Schemafull 两种模式
- 使用 SurrealQL（类 SQL 查询语言）

### 使用的 API 端点

| 端点 | 方法 | 用途 |
|------|------|------|
| `/signin` | POST | 用户认证，获取 JWT Token |
| `/key/{table}` | GET | 获取表中所有记录 |
| `/key/{table}` | POST | 创建新记录 |
| `/key/{table}` | DELETE | 删除表中所有记录 |

### SurrealDB 认证体系

| 用户级别 | 范围 | 认证参数 |
|----------|------|----------|
| Root | 所有命名空间和数据库 | user, pass |
| Namespace | 特定命名空间的所有数据库 | user, pass, ns |
| Database | 特定数据库 | user, pass, ns, db |
| Scope | 自定义范围 | 自定义 |

### 定价

| 方案 | 价格 | 说明 |
|------|------|------|
| 自托管（开源） | 免费 | 需要自己运维 |
| SurrealDB Cloud | 测试阶段 | 价格待定 |

### 限制

| 限制 | 值 |
|------|------|
| 文档大小 | 无硬限制 |
| 并发连接 | 取决于配置 |
| JWT 过期 | 取决于服务器配置 |

### 本代码支持的 SurrealDB 功能

| 功能 | 是否支持 | 说明 |
|------|----------|------|
| 文档 CRUD | ✅ | 基本的增删改查 |
| JWT 认证 | ✅ | 支持 Root 和 Namespace 用户 |
| 命名空间 | ✅ | 通过请求头指定 |
| SurrealQL | ❌ | 使用 REST API 而非 SQL |
| WebSocket | ❌ | 仅 HTTP |
| 实时查询 | ❌ | 未使用 |
| 图查询 | ❌ | 未使用 |

## 依赖关系

| 依赖 | 来源 | 版本 |
|------|------|------|
| `symfony/http-client` | Composer require | `^7.3\|^8.0` |
| `symfony/serializer` | Composer require | `^7.3\|^8.0` |
| `symfony/ai-chat` | 本模块 | `^0.6` |

## 可替换性与扩展性

### 可能的扩展方向
1. **Token 刷新**: 检测 Token 过期并自动重新认证
2. **批量写入**: 使用 SurrealQL 批量插入替代逐条 POST
3. **SurrealQL 查询**: 使用 `/sql` 端点执行更复杂的查询
4. **WebSocket**: 使用 WebSocket 连接替代 HTTP，减少开销
5. **图关系**: 利用 SurrealDB 的图特性建立消息之间的关系
6. **权限控制**: 利用 SurrealDB 的权限系统实现细粒度访问控制
