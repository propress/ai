# Result/Exception 目录分析报告

## 目录职责

`Result/Exception/` 目录包含结果处理相关的特定异常类。

**目录路径**: `src/platform/src/Result/Exception/`

---

## 包含的文件清单

| 文件 | 说明 |
|------|------|
| `RawResultAlreadySetException.php` | 原始结果已设置异常 |

---

## RawResultAlreadySetException

```php
class RawResultAlreadySetException extends RuntimeException
{
    public function __construct()
    {
        parent::__construct('The raw result was already set.');
    }
}
```

**触发条件**: 当尝试对同一个 `ResultInterface` 对象多次调用 `setRawResult()` 时抛出。

**设计目的**: 确保原始结果的不可变性，防止意外覆盖。

---

## 使用场景

```php
use Symfony\AI\Platform\Result\TextResult;
use Symfony\AI\Platform\Result\RawHttpResult;

$result = new TextResult('Hello');
$result->setRawResult($rawResult1);

try {
    $result->setRawResult($rawResult2); // 抛出异常
} catch (RawResultAlreadySetException $e) {
    // 原始结果已经设置
}
```
