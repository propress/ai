# TokenUsage 目录分析报告

## 目录职责

`TokenUsage/` 目录包含 Token 使用量追踪系统，用于记录和聚合 AI 模型调用中消耗的 Token 数量。这对于成本监控和配额管理非常重要。

**目录路径**: `src/platform/src/TokenUsage/`

---

## 包含的文件清单

| 文件 | 说明 |
|------|------|
| `TokenUsageInterface.php` | Token 使用量接口 |
| `TokenUsage.php` | Token 使用量数据类 |
| `TokenUsageAggregation.php` | Token 使用量聚合类 |
| `TokenUsageExtractorInterface.php` | Token 提取器接口 |
| `StreamListener.php` | 流式响应的 Token 监听器 |

---

## TokenUsageInterface

```php
interface TokenUsageInterface
{
    public function getPromptTokens(): ?int;
    public function getCompletionTokens(): ?int;
    public function getThinkingTokens(): ?int;
    public function getToolTokens(): ?int;
    public function getCachedTokens(): ?int;
    public function getCacheCreationTokens(): ?int;
    public function getCacheReadTokens(): ?int;
    public function getRemainingTokens(): ?int;
    public function getRemainingTokensMinute(): ?int;
    public function getRemainingTokensMonth(): ?int;
    public function getTotalTokens(): ?int;
}
```

---

## TokenUsage

单次调用的 Token 使用量。

```php
$usage = new TokenUsage(
    promptTokens: 150,
    completionTokens: 50,
    totalTokens: 200
);
```

---

## TokenUsageAggregation

多次调用的 Token 使用量聚合。

```php
$aggregation = new TokenUsageAggregation([
    $usage1,
    $usage2
]);

// 自动合并
$newAggregation = $usage1->merge($usage2);
```

**特性**:
- 求和: `promptTokens`, `completionTokens`, `totalTokens` 等
- 取最小值: `remainingTokens`, `remainingTokensMinute` 等

---

## 典型使用场景

### 场景1：获取 Token 使用量

```php
$result = $platform->invoke('gpt-4', $messages);
$text = $result->asText();

$tokenUsage = $result->getMetadata()->get('token_usage');

if ($tokenUsage) {
    echo "Prompt tokens: {$tokenUsage->getPromptTokens()}\n";
    echo "Completion tokens: {$tokenUsage->getCompletionTokens()}\n";
    echo "Total tokens: {$tokenUsage->getTotalTokens()}\n";
}
```

### 场景2：流式响应 Token 追踪

```php
$result = $platform->invoke('gpt-4', $messages, ['stream' => true]);

foreach ($result->asStream() as $chunk) {
    echo $chunk;
}

// 流结束后获取聚合的 Token 使用量
$tokenUsage = $result->getMetadata()->get('token_usage');
```

### 场景3：多轮对话 Token 累计

```php
$totalUsage = null;

foreach ($rounds as $input) {
    $result = $platform->invoke('gpt-4', $input);
    $text = $result->asText();
    
    $usage = $result->getMetadata()->get('token_usage');
    if ($usage) {
        $totalUsage = $totalUsage ? $totalUsage->merge($usage) : $usage;
    }
}

echo "Total tokens used: {$totalUsage->getTotalTokens()}";
```

### 场景4：成本估算

```php
$tokenUsage = $result->getMetadata()->get('token_usage');

$promptCost = $tokenUsage->getPromptTokens() * 0.00003; // $0.03/1K
$completionCost = $tokenUsage->getCompletionTokens() * 0.00006; // $0.06/1K
$totalCost = $promptCost + $completionCost;

echo "Estimated cost: \${$totalCost}";
```
