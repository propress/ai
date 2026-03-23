# Supabase/Store

Supabase（PostgreSQL + pgvector）的向量存储适配器，通过 Supabase REST API 调用 RPC 函数。

## 关键方法/属性
- `__construct(HttpClientInterface, string $url, string $apiKey, string $matchFunction)`
- `add/remove/query` — 通过 REST API 操作
- `supports()` — 仅支持 `VectorQuery`

## 设计模式
- 适配器模式：封装 Supabase REST API

## 与其他文件的关系
- 仅实现 `StoreInterface`（无 `ManagedStoreInterface`）
