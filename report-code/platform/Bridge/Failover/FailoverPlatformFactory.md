# FailoverPlatformFactory 分析报告

## 文件概述
`FailoverPlatformFactory` 是 `FailoverPlatform` 的静态工厂，签名与 `FailoverPlatform` 构造函数一致，仅作便捷包装。

## 方法
```php
static function create(iterable $platforms, RateLimiterFactoryInterface $rateLimiterFactory, ClockInterface $clock, LoggerInterface $logger): PlatformInterface
```
