# Result/DeferredResult.php 文件分析报告

## 文件概述

`DeferredResult.php` 实现了延迟结果模式，封装了结果转换器和原始结果，只有在实际访问结果时才执行转换。这提供了懒加载和灵活的结果处理能力。

**文件路径**: `src/platform/src/Result/DeferredResult.php`  
**命名空间**: `Symfony\AI\Platform\Result`  
**作者**: Christopher Hertel

---

## 类/接口/枚举定义

### `final class DeferredResult`

封装延迟结果转换逻辑的核心类。

---

## 方法/函数分析

### `__construct(ResultConverterInterface $resultConverter, RawResultInterface $rawResult, array $options = [])`

| 参数 | 类型 | 说明 |
|------|------|------|
| `$resultConverter` | `ResultConverterInterface` | 结果转换器 |
| `$rawResult` | `RawResultInterface` | 原始结果 |
| `$options` | `array<string, mixed>` | 选项 |

### `getResult(): ResultInterface`

执行实际的结果转换。
- 首次调用时执行转换，后续调用返回缓存结果
- 自动设置原始结果引用
- 注册 TokenUsage StreamListener（如果是流式结果）
- 合并元数据和 Token 使用信息

### 便捷访问方法

| 方法 | 返回类型 | 说明 |
|------|----------|------|
| `asText()` | `string` | 文本内容 |
| `asObject()` | `object` | 结构化对象 |
| `asBinary()` | `string` | 二进制数据 |
| `asFile(string $path)` | `void` | 保存为文件 |
| `asDataUri(?string $mimeType)` | `string` | Data URI |
| `asVectors()` | `Vector[]` | 向量数组 |
| `asReranking()` | `RerankingEntry[]` | 重排序结果 |
| `asStream()` | `Generator` | 流式生成器 |
| `asToolCalls()` | `ToolCall[]` | 工具调用数组 |

---

## 设计模式

### 延迟加载模式 (Lazy Loading)

```php
private bool $isConverted = false;
private ResultInterface $convertedResult;

public function getResult(): ResultInterface
{
    if (!$this->isConverted) {
        $this->convertedResult = $this->resultConverter->convert(...);
        $this->isConverted = true;
    }
    
    return $this->convertedResult;
}
```

---

## 使用场景

### 场景1：基本使用

```php
$result = $platform->invoke('gpt-4', $messages);

// 此时还未转换
$metadata = $result->getMetadata();

// 调用 asText() 时才执行转换
$text = $result->asText();
```

### 场景2：类型检查后访问

```php
$result = $platform->invoke('gpt-4', $messages, ['tools' => $tools]);

try {
    $toolCalls = $result->asToolCalls();
    // 处理工具调用
} catch (UnexpectedResultTypeException $e) {
    // 不是工具调用，获取文本
    $text = $result->asText();
}
```

### 场景3：访问原始数据

```php
$result = $platform->invoke('gpt-4', $messages);
$text = $result->asText();

// 访问原始 API 响应
$rawData = $result->getRawResult()->getData();
```
