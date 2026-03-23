# Exception 目录总结报告

> 目录路径：`src/store/src/Exception/`
> 命名空间：`Symfony\AI\Store\Exception`

## 概述

Store 组件的 `Exception` 目录定义了完整的异常层次结构，遵循 Symfony 组件的标准异常设计约定。该体系包含 1 个标记接口和 4 个具体异常类，覆盖了参数验证、运行时错误、功能不支持、查询类型不支持等核心错误场景。

## 目录结构

```
src/store/src/Exception/
├── ExceptionInterface.php           # 标记接口（根）
├── InvalidArgumentException.php     # 无效参数
├── RuntimeException.php             # 运行时错误
├── UnsupportedFeatureException.php  # 功能不支持
└── UnsupportedQueryTypeException.php # 查询类型不支持（特殊）
```

## 异常层次结构

```
\Throwable
├── ExceptionInterface（标记接口）
│   ├── InvalidArgumentException  ← extends \InvalidArgumentException
│   ├── RuntimeException          ← extends \RuntimeException
│   └── UnsupportedFeatureException ← extends \LogicException
│
└── UnsupportedQueryTypeException（独立）← extends \RuntimeException（final）
     ⚠️ 不实现 ExceptionInterface
```

## 设计原则

### 1. Symfony 组件异常约定

每个 Symfony 组件都定义自己的 `ExceptionInterface`，这是标准做法。核心优势：

| 特性 | 说明 |
|---|---|
| **组件隔离** | 可区分异常来源组件 |
| **统一捕获** | 一个 `catch` 块捕获全部组件异常 |
| **SPL 兼容** | 同时兼容 PHP 原生异常体系 |
| **精细处理** | 支持按具体类型分别处理 |

### 2. 双重继承策略

每个具体异常同时具有两条继承路径：

```php
class InvalidArgumentException
    extends \InvalidArgumentException   // ← SPL 兼容路径
    implements ExceptionInterface        // ← 组件统一路径
```

这使得以下三种捕获方式均可工作：

```php
catch (\InvalidArgumentException $e)  // SPL 通用
catch (ExceptionInterface $e)         // Store 组件统一
catch (InvalidArgumentException $e)   // Store 精确匹配
```

### 3. SPL 异常语义分类

| Store 异常 | SPL 基类 | 语义 |
|---|---|---|
| `InvalidArgumentException` | `\InvalidArgumentException` → `\LogicException` | 代码错误：参数值非法 |
| `UnsupportedFeatureException` | `\LogicException` | 代码错误：选择了错误的 Store |
| `RuntimeException` | `\RuntimeException` | 环境错误：运行时不可控因素 |
| `UnsupportedQueryTypeException` | `\RuntimeException` | 协议违反：未预检查即调用 |

## 异常使用指南

### 何时使用哪个异常

| 场景 | 应使用的异常 | 示例 |
|---|---|---|
| 参数值不合法 | `InvalidArgumentException` | `semanticRatio` 超出 [0,1] 范围 |
| 参数类型不匹配 | `InvalidArgumentException` | 传入非 `EmbeddableDocumentInterface` 的文档 |
| 必需配置项缺失 | `InvalidArgumentException` | 未提供 `dimension` 选项 |
| 空内容/空字符串 | `InvalidArgumentException` | `TextDocument` 内容为空 |
| 文件不存在/不可读 | `RuntimeException` | 文档加载器找不到文件 |
| 外部服务通信失败 | `RuntimeException` | API 请求超时、连接断开 |
| 向量化过程出错 | `RuntimeException` | 嵌入模型返回错误 |
| Store 不支持某操作 | `UnsupportedFeatureException` | `remove()` 未实现 |
| Store 不支持某查询类型 | `UnsupportedQueryTypeException` | 传入不支持的 Query |

### 最佳捕获模式

```php
use Symfony\AI\Store\Exception\ExceptionInterface;
use Symfony\AI\Store\Exception\InvalidArgumentException;
use Symfony\AI\Store\Exception\RuntimeException;
use Symfony\AI\Store\Exception\UnsupportedFeatureException;
use Symfony\AI\Store\Exception\UnsupportedQueryTypeException;

try {
    $store->query($query, $options);
} catch (UnsupportedQueryTypeException $e) {
    // 1. 最具体：查询类型不支持（不在 ExceptionInterface 体系中！）
    $logger->warning('请使用 supports() 预检查查询类型', ['error' => $e->getMessage()]);
} catch (InvalidArgumentException $e) {
    // 2. 参数错误：调用方需修正输入
    $logger->error('参数错误', ['error' => $e->getMessage()]);
} catch (UnsupportedFeatureException $e) {
    // 3. 功能不支持：需要更换 Store 实现
    $logger->warning('功能不支持', ['error' => $e->getMessage()]);
} catch (RuntimeException $e) {
    // 4. 运行时错误：可考虑重试
    $logger->error('运行时错误', ['error' => $e->getMessage()]);
} catch (ExceptionInterface $e) {
    // 5. 兜底：未来可能新增的 Store 异常
    $logger->error('Store 操作失败', ['error' => $e->getMessage()]);
}
```

## UnsupportedQueryTypeException 的特殊性

这是异常体系中最特殊的成员，需要特别关注：

| 特性 | 其他异常 | `UnsupportedQueryTypeException` |
|---|---|---|
| 实现 `ExceptionInterface` | ✅ | ❌ |
| 可被 `catch (ExceptionInterface)` 捕获 | ✅ | ❌ |
| `final` 修饰 | ❌ | ✅ |
| 自定义构造函数 | ❌ | ✅ |
| 依赖 `StoreInterface` | ❌ | ✅ |

**使用时的关键提醒**：如果你用 `catch (ExceptionInterface $e)` 来捕获所有 Store 异常，`UnsupportedQueryTypeException` 会逃逸！必须单独处理。

## 使用频率统计

基于代码库分析，各异常的使用频率：

| 异常 | 使用频率 | 主要使用者 |
|---|---|---|
| `InvalidArgumentException` | ⭐⭐⭐⭐⭐ | 所有加载器、转换器、索引器、25+ Bridge |
| `UnsupportedQueryTypeException` | ⭐⭐⭐⭐⭐ | 所有 25+ Store 实现（`query()` 默认分支） |
| `RuntimeException` | ⭐⭐⭐⭐ | 加载器、向量化器、命令行、部分 Bridge |
| `UnsupportedFeatureException` | ⭐⭐ | Supabase、MongoDb、集成测试基类 |

## 可扩展点

### 创建自定义异常

如需为自定义 Bridge 添加特殊异常类型：

```php
namespace App\Store\Exception;

use Symfony\AI\Store\Exception\ExceptionInterface;

// 方式1：通用模式 — 继承 SPL 异常 + 实现 ExceptionInterface
class ConnectionPoolExhaustedException extends \RuntimeException implements ExceptionInterface
{
    public function __construct(string $storeName, int $poolSize)
    {
        parent::__construct(\sprintf(
            'Store "%s" 的连接池已耗尽（大小：%d）',
            $storeName,
            $poolSize
        ));
    }
}

// 方式2：特殊模式 — 类似 UnsupportedQueryTypeException 的自描述异常
final class QueryTimeoutException extends \RuntimeException implements ExceptionInterface
{
    public function __construct(QueryInterface $query, float $timeout)
    {
        parent::__construct(\sprintf(
            '查询 "%s" 超时（%.1f秒）',
            $query::class,
            $timeout
        ));
    }
}
```

## 与其他组件异常体系的对比

| 组件 | ExceptionInterface | 典型异常 |
|---|---|---|
| **Store** | `Symfony\AI\Store\Exception\ExceptionInterface` | `InvalidArgumentException`、`RuntimeException`、`UnsupportedFeatureException` |
| **Platform** | `Symfony\AI\Platform\Exception\ExceptionInterface` | `InvalidArgumentException`、`RuntimeException` |
| **Agent** | 同 Platform/Store | 复用上游异常 |

各组件的异常互不干扰，`catch (Store\Exception\ExceptionInterface)` 不会误捕获 `Platform` 组件的异常。
