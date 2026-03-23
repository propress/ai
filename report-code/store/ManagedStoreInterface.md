# ManagedStoreInterface 源码分析报告

## 基本信息

| 项目 | 内容 |
|------|------|
| **文件路径** | `src/store/src/ManagedStoreInterface.php` |
| **命名空间** | `Symfony\AI\Store` |
| **类型** | 接口（Interface） |
| **作者** | Oskar Stark |
| **依赖** | 无外部依赖 |

---

## 职责说明

`ManagedStoreInterface` 定义了存储后端的**生命周期管理**能力，即基础设施的创建和销毁操作。它与 `StoreInterface` 是**正交关系**——一个存储后端可以同时实现两个接口（自管理存储），也可以只实现 `StoreInterface`（外部管理存储）。

典型场景：
- **需要实现的后端**：PostgreSQL（需要创建表和索引）、SQLite（需要建表）、Elasticsearch（需要创建索引映射）、Redis（需要创建搜索索引）
- **不需要实现的后端**：Pinecone（云端全托管）、部分 SaaS 服务（基础设施由服务商管理）

---

## 方法签名详解

### 1. `setup(array $options = []): void`

```php
/**
 * @param array<mixed> $options
 */
public function setup(array $options = []): void;
```

**功能**：创建/初始化存储所需的基础设施。

| 参数 | 类型 | 说明 |
|------|------|------|
| `$options` | `array<mixed>` | 可选配置参数，因后端而异 |

**返回值**：`void`

**典型操作**：
| 存储后端 | `setup()` 执行的操作 |
|----------|---------------------|
| PostgreSQL | `CREATE TABLE IF NOT EXISTS ...`; 创建向量索引（如 pgvector 的 `ivfflat` 或 `hnsw` 索引） |
| SQLite | `CREATE TABLE ...`; 创建 FTS5 虚拟表 |
| Elasticsearch | 创建索引（Index）并设置映射（Mapping） |
| Redis | 使用 `FT.CREATE` 创建 RediSearch 索引 |
| ChromaDb | 创建集合（Collection） |
| Milvus | 创建集合并定义 Schema |

**幂等性要求**：实现通常应该是幂等的（Idempotent），即重复调用不应报错。常见做法是使用 `IF NOT EXISTS` 语句或先检查再创建。

---

### 2. `drop(array $options = []): void`

```php
/**
 * @param array<mixed> $options
 */
public function drop(array $options = []): void;
```

**功能**：销毁存储基础设施，删除所有数据。

| 参数 | 类型 | 说明 |
|------|------|------|
| `$options` | `array<mixed>` | 可选配置参数 |

**返回值**：`void`

**典型操作**：
| 存储后端 | `drop()` 执行的操作 |
|----------|---------------------|
| PostgreSQL | `DROP TABLE IF EXISTS ...` |
| SQLite | `DROP TABLE ...` |
| Elasticsearch | 删除索引 |
| Redis | `FT.DROPINDEX` |
| ChromaDb | 删除集合 |

**⚠️ 危险操作**：此方法会**永久删除所有数据**，应谨慎使用。在生产环境中通常需要额外的确认机制。

---

## 设计模式

### 1. 接口分离原则（ISP — Interface Segregation Principle）

`ManagedStoreInterface` 是接口分离原则的典型实践：

```
                   StoreInterface          ManagedStoreInterface
                        │                         │
                        │                         │
            ┌───────────┴───────────┐             │
            ▼                       ▼             │
      PineconeStore          PostgresStore ◀──────┘
     (仅 StoreInterface)    (同时实现两个接口)
```

**为什么不将 `setup()` / `drop()` 放入 `StoreInterface`？**
- 并非所有存储后端都需要生命周期管理
- 云托管服务（如 Pinecone）的基础设施由服务商管理
- 强制实现会导致空方法（`setup() {}` — 无操作），违反里氏替换原则
- 调用方可通过 `instanceof` 检查是否支持管理操作

### 2. 模板方法的准备阶段

`setup()` 和 `drop()` 可以看作是存储后端使用前后的准备和清理阶段，类似于数据库迁移（Migration）的 `up()` 和 `down()`。

---

## 在架构中的位置

```
                  CLI 命令层
          ┌──────────┴──────────┐
          ▼                     ▼
   SetupStoreCommand     DropStoreCommand
          │                     │
          ▼                     ▼
   ManagedStoreInterface::setup()  ManagedStoreInterface::drop()
          │                     │
          ▼                     ▼
   ┌──────────────────────────────┐
   │    具体存储后端实现           │
   │  (PostgresStore, RedisStore  │
   │   ElasticsearchStore, ...)   │
   └──────────────────────────────┘
```

**调用者**：
- `SetupStoreCommand`：Symfony Console 命令，在应用部署时调用 `setup()` 初始化存储基础设施
- `DropStoreCommand`：Symfony Console 命令，调用 `drop()` 清理存储
- 测试代码：在测试 setUp/tearDown 中创建和销毁测试存储
- 部署脚本：作为 CI/CD 流程的一部分

**与 StoreInterface 的关系**：
- 两个接口完全独立，没有继承关系
- 一个类可以同时实现两者：`class PostgresStore implements StoreInterface, ManagedStoreInterface`
- 调用方通过类型检查判断是否支持：`if ($store instanceof ManagedStoreInterface)`

---

## 可替换/可扩展部分

### 实现新的受管理存储

同时实现两个接口的完整示例：

```php
final class MyDatabaseStore implements StoreInterface, ManagedStoreInterface
{
    public function __construct(
        private \PDO $connection,
        private string $tableName = 'vector_documents',
    ) {}

    // StoreInterface 方法
    public function add(VectorDocument|array $documents): void { /* ... */ }
    public function remove(string|array $ids, array $options = []): void { /* ... */ }
    public function query(QueryInterface $query, array $options = []): iterable { /* ... */ }
    public function supports(string $queryClass): bool { /* ... */ }

    // ManagedStoreInterface 方法
    public function setup(array $options = []): void
    {
        $this->connection->exec(sprintf(
            'CREATE TABLE IF NOT EXISTS %s (
                id VARCHAR(255) PRIMARY KEY,
                vector BLOB NOT NULL,
                metadata JSON
            )', $this->tableName
        ));
    }

    public function drop(array $options = []): void
    {
        $this->connection->exec(sprintf(
            'DROP TABLE IF EXISTS %s', $this->tableName
        ));
    }
}
```

### 在 AI Bundle 中的配置

```yaml
# config/packages/ai.yaml
ai:
    stores:
        my_store:
            type: postgres
            dsn: '%env(DATABASE_URL)%'
```

配置后可通过 CLI 管理：
```bash
# 初始化存储
php bin/console ai:store:setup my_store

# 销毁存储
php bin/console ai:store:drop my_store
```

---

## 设计技巧与原因

1. **独立于 `StoreInterface` 的原因**：这是遵循 SOLID 的接口分离原则。存储的"数据操作能力"和"生命周期管理能力"是两个独立关注点。有些云服务只需要数据操作，基础设施由平台自动管理。将两者分开让类型系统更精确。

2. **`$options` 使用 `array<mixed>` 而非 `array<string, mixed>`**：注意这里与 `StoreInterface` 的 `$options` 类型注解不同。`setup()` 和 `drop()` 的选项可能包含非字符串键的数组元素（如数字索引的列定义列表），因此使用了更宽松的类型。

3. **`void` 返回值**：这两个方法是纯命令操作（Command），不需要返回结果。如果需要了解操作是否成功，应通过异常传达失败信息。

4. **`$options` 默认值为空数组**：这使得大多数场景下调用方无需传递参数，降低了使用门槛。后端的特定配置（如表名、索引类型）通常通过构造函数注入，而非 `setup()` 的参数。

5. **不强制幂等性但建议实现**：接口本身不强制要求幂等，但最佳实践是使用 `IF NOT EXISTS` 等机制确保幂等。这让 `setup()` 可以安全地在多次部署中调用。
