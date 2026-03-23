# FailoverPlatform 分析报告

## 文件概述
见 `Failover.md`。

## 核心方法摘要
- `invoke()` → 委托 `do(fn($p) => $p->invoke(...))`
- `getModelCatalog()` → 委托 `do(fn($p) => $p->getModelCatalog())`
- `do(Closure)` → 迭代尝试所有 Platform，catch 所有 Throwable，用 RateLimiter 追踪恢复
