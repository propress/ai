# MockAgent.php 分析报告

## 概述

`MockAgent` 是 `AgentInterface` 的测试专用实现，不发出任何真实 HTTP 请求，通过预配置的输入→响应映射提供可预测的结果，并内置了调用记录和断言方法，设计灵感来自 Symfony 的 `MockHttpClient`。

## 关键方法分析

### 构造函数

```php
public function __construct(
    array $responses = [],   // ['用户输入文本' => '响应字符串 | MockResponse | Closure']
    private string $name = 'mock',
)
```

预加载响应映射，响应值可以是：
- **字符串**：直接作为文本响应。
- **`MockResponse` 对象**：可附带元数据（如 `sources`）。
- **`Closure`**：动态响应，签名为 `function(MessageBag, array $options, string $input)`。

### call(MessageBag $messages, array $options = []): ResultInterface

1. 提取最后一条 `UserMessage` 中的文本内容作为查找键。
2. 在 `$responses` 映射中查找，若不存在则抛出 `RuntimeException`。
3. 若为 `Closure`，则调用并将结果转换为 `MockResponse`。
4. 调用 `MockResponse::toResult()` 返回 `TextResult`。
5. 记录本次调用到 `$calls` 数组，递增 `$callCount`。

### 断言方法

| 方法 | 说明 |
|---|---|
| `assertCallCount(int $n)` | 断言恰好被调用 n 次 |
| `assertCalled()` | 断言至少被调用过一次 |
| `assertNotCalled()` | 断言从未被调用 |
| `assertCalledWith(string $input)` | 断言某次调用使用了指定输入 |

### 辅助方法

| 方法 | 说明 |
|---|---|
| `addResponse()` | 链式添加单条响应 |
| `clearResponses()` | 清空所有响应 |
| `getCalls()` / `getCall(int $i)` / `getLastCall()` | 检索调用记录 |
| `reset()` | 重置调用计数和记录（保留响应配置） |

## 设计模式

- **测试替身（Test Double）/ 桩（Stub）**：以静态映射代替真实 AI 调用，确保测试的确定性与速度。
- **流畅接口（Fluent Interface）**：`addResponse()`、`clearResponses()`、`reset()` 均返回 `$this`，支持链式调用。
- **仿 MockHttpClient 设计**：与 Symfony HTTP Client 测试工具保持一致的使用习惯。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `AgentInterface` | 实现此接口，可无差别替换真实 `Agent` |
| `MockResponse` | 封装测试响应内容，支持附带元数据 |
| `Exception/RuntimeException` | 输入无匹配响应时抛出 |
| `Exception/OutOfBoundsException` | `getCall(int $i)` 索引越界时抛出 |
| `Exception/LogicException` | `getLastCall()` 在无调用记录时抛出 |
