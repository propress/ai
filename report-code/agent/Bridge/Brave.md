# Bridge/Brave/Brave.php 分析报告

## 概述

`Brave` 是 Brave Search API 的工具实现，通过 `#[AsTool('brave_search', ...)]` 注册为工具，支持分页网页搜索，并通过 `HasSourcesTrait` 收集搜索结果来源。

## 关键方法分析

### __invoke(string $query, int $count, int $offset): array

- `$query`：搜索词（通过 `#[With(maxLength: 500)]` 限制最大长度）。
- `$count`：返回结果数量（默认 20）。
- `$offset`：跳过前 N 条结果（`#[With(minimum: 0, maximum: 9)]`，用于翻页）。
- 调用 `https://api.search.brave.com/res/v1/web/search`，使用 `X-Subscription-Token` 头认证。
- 将每条结果的 `title`、`url`、`description` 注册为 `Source` 对象。
- 返回 `array<int, {title, description, url}>` 数组。

## 安全特性

- `#[\SensitiveParameter]` 标注 `$apiKey`，防止 API 密钥在调试信息或日志中泄露。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Toolbox/Attribute/AsTool` | 工具注册注解 |
| `Toolbox/Source/HasSourcesInterface` + `HasSourcesTrait` | 来源收集 |
| `Platform/Contract/JsonSchema/Attribute/With` | 参数约束（maxLength、minimum、maximum） |
