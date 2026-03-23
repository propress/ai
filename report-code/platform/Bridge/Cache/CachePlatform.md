# CachePlatform 分析报告

## 文件概述
见 `Cache.md` 中的 CachePlatform 分析节。

## 关键方法摘要

### `invoke(string $model, ...): DeferredResult`
核心逻辑：检查缓存 → 命中时反序列化恢复 → 未命中时调用真实 Platform 并存储结果。

两个特殊 options（消费后删除，不传给真实 Platform）：
- `prompt_cache_key`：缓存键标识符
- `prompt_cache_ttl`：此次调用的 TTL 覆盖值

### `getModelCatalog()`
完全委托给被包装的 Platform。
