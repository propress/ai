# Failover Bridge 分析报告

## 文件概述
`Failover` Bridge 实现**多平台故障转移**，按顺序尝试多个 Platform，任一失败后自动切换到下一个，所有平台都失败时抛出 `RuntimeException`。适用于高可用场景（如主用 OpenAI，备用 Anthropic）。

## 目录结构
```
Failover/
├── FailoverPlatform.php        # 核心故障转移逻辑
└── FailoverPlatformFactory.php # 静态工厂（简化创建）
```

## FailoverPlatform 分析

### 关键设计：WeakMap 追踪失败平台
```php
private readonly \WeakMap $failedPlatforms;
```
使用 PHP 8.0+ 的 `WeakMap`：
- Key: `PlatformInterface` 对象
- Value: 失败时的时间戳（int）
- **弱引用**：Platform 对象被 GC 时自动从 Map 中移除（不阻止 GC）

### `invoke()` 和 `getModelCatalog()` 的统一处理
两个方法都委托给私有的 `do(Closure $func)` 方法，避免重复的循环+错误处理逻辑。

### `do(Closure $func)` 核心逻辑
```php
foreach ($platforms as $platform) {
    $limiter = $rateLimiterFactory->create($platform::class);
    try {
        if ($limiter->consume()->isAccepted()) {
            $this->failedPlatforms->offsetUnset($platform); // 恢复：从失败列表移除
        }
        return $func($platform);
    } catch (\Throwable) {
        $limiter->consume();                    // 消耗一次限速令牌
        $this->failedPlatforms[$platform] = now; // 记录失败时间
        $this->logger->error(...);
        continue; // 尝试下一个
    }
}
throw new RuntimeException('All platforms failed.');
```

### RateLimiter 集成
`$rateLimiterFactory->create($platform::class)` 为**每个 Platform 类**创建独立限速器：
- 限速器令牌耗尽 → `consume()->isAccepted()` 返回 false → 跳过该 Platform（不触发新的真实请求）
- 限速器恢复 → 下次迭代中 `isAccepted()` 返回 true → 从 `failedPlatforms` 移除（重新启用）

**效果**：实现了"错误率限流 + 自动恢复"的熔断器（Circuit Breaker）简化版。

## 设计模式
**装饰器（Decorator）+ 组合模式（Composite）**：将多个 Platform 包装为单一 Platform，对调用方透明。

**熔断器（Circuit Breaker）简化版**：通过 RateLimiter 实现故障隔离和自动恢复。

## 使用示例
```php
$platform = FailoverPlatformFactory::create(
    platforms: [$openaiPlatform, $anthropicPlatform, $localOllamaPlatform],
    rateLimiterFactory: $rateLimiterFactory,
);
// 调用方完全不知道有 Failover 逻辑
$result = $platform->invoke('gpt-4o', $messages);
```

## 与其他文件的关系
- 实现 `PlatformInterface`，完全兼容任何使用 Platform 的代码
- 依赖 `symfony/rate-limiter` 和 `psr/log`
