# Bridge/Doctrine/DoctrineDbalMessageStore.php 分析报告

## 文件基本信息

| 属性 | 值 |
|------|------|
| 文件路径 | `src/chat/src/Bridge/Doctrine/DoctrineDbalMessageStore.php` |
| 命名空间 | `Symfony\AI\Chat\Bridge\Doctrine` |
| Composer 包名 | `symfony/ai-doctrine-message-store` |
| 类型 | 最终类（final class） |
| 实现接口 | `ManagedStoreInterface`, `MessageStoreInterface` |
| 作者 | Guillaume Loulier |
| 行数 | 146 行 |

## 功能描述

`DoctrineDbalMessageStore` 是 Chat 模块中**最复杂的 Bridge 实现**（146 行）。它通过 Doctrine DBAL（Database Abstraction Layer）将对话消息持久化到关系型数据库中。支持所有 Doctrine DBAL 兼容的数据库（MySQL、PostgreSQL、SQLite、Oracle 等），且包含了 Oracle 平台的特殊处理逻辑。

## 类定义

```php
final class DoctrineDbalMessageStore implements ManagedStoreInterface, MessageStoreInterface
{
    public function __construct(
        private readonly string $tableName,
        private readonly DBALConnection $dbalConnection,
        private readonly SerializerInterface $serializer = new Serializer([
            new ArrayDenormalizer(),
            new MessageNormalizer(),
        ], [new JsonEncoder()]),
        private readonly ClockInterface $clock = new MonotonicClock(),
    ) {
    }
}
```

### 构造函数参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `$tableName` | `string` | - | 数据库表名 |
| `$dbalConnection` | `DBALConnection` | - | Doctrine DBAL 数据库连接 |
| `$serializer` | `SerializerInterface` | 默认 Serializer | 消息序列化器 |
| `$clock` | `ClockInterface` | `MonotonicClock()` | 时钟接口，用于生成时间戳 |

**ClockInterface 注入技巧**:
使用 PSR-20 `ClockInterface` 而非直接使用 `time()` 或 `new \DateTime()`：
- **可测试性**: 测试中可以注入模拟时钟，控制时间
- **一致性**: 确保整个应用使用同一时间源
- 默认值 `MonotonicClock` 来自 Symfony Clock 组件

## 方法详解

### `setup(array $options = []): void`

```php
public function setup(array $options = []): void
{
    if ([] !== $options) {
        throw new InvalidArgumentException('No supported options.');
    }

    $schema = $this->dbalConnection->createSchemaManager()->introspectSchema();

    if ($schema->hasTable($this->tableName)) {
        return;
    }

    $this->addTableToSchema($schema);
}
```

| 输入 | 输出 | 异常 | 行为 |
|------|------|------|------|
| `array $options`（必须为空） | `void` | `InvalidArgumentException` | 创建数据库表（幂等） |

**流程**:
1. 验证 `$options` 为空
2. 获取当前数据库 Schema
3. 检查表是否已存在（幂等）
4. 不存在则调用 `addTableToSchema()` 创建

### `drop(): void`

```php
public function drop(): void
{
    $schema = $this->dbalConnection->createSchemaManager()->introspectSchema();

    if (!$schema->hasTable($this->tableName)) {
        return;
    }

    $queryBuilder = $this->dbalConnection->createQueryBuilder()
        ->delete($this->tableName);

    $this->dbalConnection->executeStatement($queryBuilder->getSQL());
}
```

| 输入 | 输出 | 行为 |
|------|------|------|
| 无 | `void` | 删除表中所有记录（保留表结构） |

**注意**: `drop()` 执行的是 `DELETE FROM table`（清空数据），而非 `DROP TABLE`（删除表结构）。表结构保留。

### `save(MessageBag $messages): void`

```php
public function save(MessageBag $messages): void
{
    $queryBuilder = $this->dbalConnection->createQueryBuilder()
        ->insert($this->tableName)
        ->values([
            'messages' => '?',
            'added_at' => '?',
        ]);

    $this->dbalConnection->executeStatement($queryBuilder->getSQL(), [
        $this->serializer->serialize($messages->getMessages(), 'json'),
        $this->clock->now()->getTimestamp(),
    ]);
}
```

| 输入 | 输出 | 行为 |
|------|------|------|
| `MessageBag $messages` | `void` | 插入新记录（不替换旧记录） |

**重要差异**: 与其他 Bridge 的 save 不同，Doctrine Bridge 执行的是 **INSERT**（追加），而非 **SET**（替换）。这意味着每次 save 都会新增一条记录，`load` 时需要合并所有记录。

**这种设计的意图**:
- 保留每次保存的快照，形成时间线
- 按 `added_at` 排序可以重建对话历史
- 但这也意味着会有冗余数据（每次保存包含完整历史）

### `load(): MessageBag`

```php
public function load(): MessageBag
{
    $queryBuilder = $this->dbalConnection->createQueryBuilder()
        ->select('messages')
        ->from($this->tableName)
        ->orderBy('added_at', 'ASC');

    $result = $this->dbalConnection->executeQuery($queryBuilder->getSQL());

    $messages = array_map(
        fn (array $payload): array => $this->serializer->deserialize(
            $payload['messages'], MessageInterface::class.'[]', 'json'
        ),
        $result->fetchAllAssociative(),
    );

    return new MessageBag(...array_merge(...$messages));
}
```

| 输入 | 输出 | 行为 |
|------|------|------|
| 无 | `MessageBag` | 从所有记录中加载并合并消息 |

**合并逻辑**:
```php
array_merge(...$messages)
```
将所有记录中的消息数组合并为一个扁平数组，然后构建 MessageBag。

### `addTableToSchema(Schema $schema): void`（私有方法）

```php
private function addTableToSchema(Schema $schema): void
```

**功能**: 创建数据库表，包含以下列：

| 列名 | 类型 | 约束 |
|------|------|------|
| `id` | `BIGINT` | 主键，自增 |
| `messages` | `TEXT` | 非空，存储 JSON |
| `added_at` | `INTEGER` | 非空，Unix 时间戳 |

**Oracle 平台特殊处理**:

```php
if ($this->dbalConnection->getDatabasePlatform() instanceof OraclePlatform) {
    $serverVersion = $this->dbalConnection->executeQuery(
        "SELECT version FROM product_component_version WHERE product LIKE 'Oracle Database%'"
    )->fetchOne();
    if (version_compare($serverVersion, '12.1.0', '>=')) {
        $idColumn->setAutoincrement(false);
        $idColumn->setDefault($this->tableName.'_seq.nextval');
        $schema->createSequence($this->tableName.'_seq');
    }
}
```

**为什么需要 Oracle 特殊处理**:
- Oracle 12.1+ 之前不原生支持自增列，需要使用 SEQUENCE + TRIGGER
- Doctrine DBAL 的默认行为是为 Oracle 创建 SEQUENCE 和 TRIGGER
- 但 Oracle 12.1+ 支持使用 `DEFAULT sequence.NEXTVAL` 语法
- 此代码手动创建 SEQUENCE 并设置列默认值，避免创建不必要的 TRIGGER

**Doctrine DBAL 版本兼容**:

```php
if (class_exists(PrimaryKeyConstraint::class)) {
    $table->addPrimaryKeyConstraint(new PrimaryKeyConstraint(null, [
        new UnqualifiedName(Identifier::unquoted('id')),
    ], true));
} else {
    $table->setPrimaryKey(['id']);
}
```

这段代码兼容 Doctrine DBAL 的不同版本：
- 新版 DBAL 使用 `PrimaryKeyConstraint` 类
- 旧版使用 `setPrimaryKey()` 方法

## 设计模式

### 1. 适配器模式（Adapter Pattern）

将 Doctrine DBAL 适配为 `MessageStoreInterface`。

### 2. Schema 构建器模式（Schema Builder Pattern）

通过 Doctrine Schema API 编程式地创建数据库表，而非使用原生 SQL。

**好处**:
- 数据库无关：同一代码适用于 MySQL、PostgreSQL、SQLite、Oracle
- 类型安全：编译时检查列类型
- 可维护：比原生 SQL 更容易修改

### 3. 查询构建器模式（Query Builder Pattern）

使用 Doctrine QueryBuilder 构建 SQL，而非拼接字符串。

### 4. 时钟抽象模式（Clock Abstraction Pattern）

注入 `ClockInterface` 而非直接获取系统时间，使时间逻辑可测试。

## 外部知识：Doctrine DBAL

### Doctrine DBAL 概述

Doctrine DBAL（Database Abstraction Layer）是 PHP 的数据库抽象层，提供：
- 统一的数据库连接 API
- 跨数据库的 Schema 管理
- 查询构建器
- 类型系统

### 支持的数据库

| 数据库 | Platform 类 |
|--------|------------|
| MySQL / MariaDB | `MySQLPlatform` |
| PostgreSQL | `PostgreSQLPlatform` |
| SQLite | `SQLitePlatform` |
| Oracle | `OraclePlatform` |
| SQL Server | `SQLServerPlatform` |

### 本代码使用的 DBAL API

| API | 用途 |
|-----|------|
| `createSchemaManager()->introspectSchema()` | 获取当前数据库 Schema |
| `createQueryBuilder()` | 创建查询构建器 |
| `executeStatement()` | 执行写操作（INSERT/UPDATE/DELETE） |
| `executeQuery()` | 执行读操作（SELECT） |
| `Schema::createTable()` | 创建表定义 |
| `Schema::toSql()` | 将 Schema 变更转为 SQL |

### 定价参考

| 数据库 | 价格 | 说明 |
|--------|------|------|
| SQLite | 免费 | 嵌入式，无需服务器 |
| MySQL Community | 免费 | 自托管 |
| PostgreSQL | 免费 | 自托管 |
| AWS RDS MySQL | ~$0.017/小时（db.t3.micro） | 托管 |
| AWS RDS PostgreSQL | ~$0.018/小时（db.t3.micro） | 托管 |
| Oracle SE | ~$5,000+/CPU/年 | 商业许可 |

## 依赖关系

| 依赖 | 来源 | 版本 |
|------|------|------|
| `doctrine/dbal` | Composer require | `^4.0` |
| `symfony/clock` | Composer require | `^7.3\|^8.0` |
| `symfony/serializer` | Composer require | `^7.3\|^8.0` |
| `symfony/ai-chat` | 本模块 | `^0.6` |
| `psr/clock` | 间接依赖 | `ClockInterface` |

## 创建的数据库表结构

```sql
-- MySQL 示例
CREATE TABLE message_store (
    id BIGINT AUTO_INCREMENT NOT NULL,
    messages TEXT NOT NULL,
    added_at INT NOT NULL,
    PRIMARY KEY (id)
);

-- Oracle 12.1+ 示例
CREATE SEQUENCE message_store_seq;
CREATE TABLE message_store (
    id NUMBER(19) DEFAULT message_store_seq.NEXTVAL NOT NULL,
    messages CLOB NOT NULL,
    added_at NUMBER(10) NOT NULL,
    CONSTRAINT pk_message_store PRIMARY KEY (id)
);
```

## 可替换性与扩展性

### 可替换
- 替换 `DBALConnection` 可以切换数据库
- 替换 `ClockInterface` 可以控制时间戳生成
- 替换 `SerializerInterface` 可以改变序列化策略

### 可能的扩展方向
1. **分区表**: 按时间或用户分区，提升大规模数据的查询性能
2. **索引优化**: 在 `added_at` 上创建索引
3. **数据清理**: 添加按时间清理旧对话的功能
4. **加密存储**: 在存入数据库前加密消息内容
5. **审计日志**: 利用数据库的审计功能记录操作
6. **全文搜索**: 利用数据库的全文搜索能力搜索对话内容
