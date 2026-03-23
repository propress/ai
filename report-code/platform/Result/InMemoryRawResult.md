# InMemoryRawResult 分析报告

## 文件概述
`InMemoryRawResult` 是 `RawResultInterface` 的内存实现，用于测试场景，无需真实 HTTP 请求即可模拟 AI 模型原始响应。

## 类定义
- **类型**: `final class`，实现 `RawResultInterface`
- **命名空间**: `Symfony\AI\Platform\Result`

## 方法分析

### `__construct(array $data, iterable $dataStream, object $object)`
- `$data`：模拟非流式响应 JSON 数据
- `$dataStream`：模拟流式响应的数据块序列
- `$object`：模拟原始对象（默认 `stdClass`）

### `getData(): array`
- 返回固定的 `$data` 数组（模拟 `toArray()` 响应）

### `getDataStream(): iterable`
- `yield from $dataStream`：逐一返回流式数据块

### `getObject(): object`
- 返回注入的对象

## 设计模式
**测试替身（Test Double / Fake）**：提供真实接口的轻量内存实现，便于单元测试不依赖网络。

## 扩展点
可在测试中构造具体格式的数据，模拟任意 AI 提供商的原始响应格式。

## 与其他文件的关系
- 实现 `RawResultInterface`
- 被 `InMemoryPlatform` 使用
- 被各 Bridge 的测试用例使用

## 使用示例
```php
$raw = new InMemoryRawResult(['choices' => [['message' => ['content' => 'Hello']]]]);
$result = $converter->convert($raw);
```
