# Metadata 目录分析报告

## 目录职责

`Metadata/` 目录包含元数据系统，用于在消息、结果和其他对象上附加额外信息。元数据支持合并操作，特别适合累积性数据如 Token 使用量。

**目录路径**: `src/platform/src/Metadata/`

---

## 包含的文件清单

| 文件 | 说明 |
|------|------|
| `Metadata.php` | 元数据容器类 |
| `MetadataAwareInterface.php` | 元数据感知接口 |
| `MetadataAwareTrait.php` | 元数据感知特征 |
| `MergeableMetadataInterface.php` | 可合并元数据接口 |

---

## Metadata 类

```php
class Metadata implements JsonSerializable, Countable, IteratorAggregate, ArrayAccess
{
    public function all(): array;
    public function set(array $metadata): void;
    public function merge(self $metadata): void;
    public function add(string $key, mixed $value): void;
    public function has(string $key): bool;
    public function get(string $key, mixed $default = null): mixed;
    public function remove(string $key): void;
}
```

**特性**:
- 实现 `ArrayAccess`，支持数组语法
- 实现 `IteratorAggregate`，支持遍历
- 支持 JSON 序列化
- 支持值合并（`MergeableMetadataInterface`）

---

## MergeableMetadataInterface

```php
interface MergeableMetadataInterface
{
    public function merge(self $metadata): self;
}
```

**用途**: 当添加同键值时，如果旧值实现此接口，会调用 `merge()` 方法合并。

**典型实现**: `TokenUsage`, `TokenUsageAggregation`

---

## 典型使用场景

### 场景1：访问结果元数据

```php
$result = $platform->invoke('gpt-4', $messages);
$text = $result->asText();

$metadata = $result->getMetadata();

// 获取 Token 使用信息
$tokenUsage = $metadata->get('token_usage');
if ($tokenUsage) {
    echo "Total tokens: {$tokenUsage->getTotalTokens()}";
}
```

### 场景2：添加自定义元数据

```php
$result->getMetadata()->add('request_id', 'abc123');
$result->getMetadata()->add('latency_ms', 150);
```

### 场景3：数组语法访问

```php
$metadata = $result->getMetadata();

// 设置
$metadata['custom_key'] = 'value';

// 获取
$value = $metadata['custom_key'];

// 检查
if (isset($metadata['custom_key'])) {
    // ...
}
```
