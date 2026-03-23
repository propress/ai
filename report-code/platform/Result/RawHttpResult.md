# RawHttpResult 分析报告

## 文件概述
`RawHttpResult` 封装 Symfony HTTP Client 的 `ResponseInterface`，实现 `RawResultInterface`，是真实 HTTP 调用下的原始结果载体，支持流式和非流式两种读取方式。

## 类定义
- **类型**: `final class`，实现 `RawResultInterface`

## 方法分析

### `getData(): array`
- 调用 `$this->response->toArray(false)`（false = 不抛出 HTTP 错误，错误处理交给 ResultConverter）

### `getDataStream(): iterable`
- 使用 `EventSourceHttpClient` 处理 SSE（Server-Sent Events）流
- **核心技巧**：
  1. 跳过 first/last chunk 和 `[DONE]` 标记
  2. 去除前后方括号处理 JSON 数组格式的流（如 Gemini）
  3. 按 `,\r\n` 分割支持多 JSON 对象同行返回
  4. 跳过注释行（`:`开头）
  5. 每个有效 JSON 块 `yield` 出去

### `getObject(): ResponseInterface`
- 返回原始 HTTP 响应对象，供需要访问 HTTP 头等信息的场景使用

## 设计模式
**适配器（Adapter）**：将 Symfony HTTP Client 的 `ResponseInterface` 适配为平台统一的 `RawResultInterface`。

## 技巧亮点
SSE 流处理逻辑包含多种边界情况处理，覆盖 OpenAI、Gemini 等不同厂商的流格式差异。这一差异通常各 Bridge 需要自己处理，但基础解析逻辑集中在此。

## 与其他文件的关系
- 实现 `RawResultInterface`
- 由各 Bridge 的 `ModelClient::request()` 返回
- 被 `DeferredResult` / `ResultConverter` 消费

## 使用示例
```php
// 在 Bridge 的 ModelClient 中:
return new RawHttpResult($this->httpClient->request('POST', $url, $options));
```
