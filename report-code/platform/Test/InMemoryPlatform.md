# InMemoryPlatform 分析报告

## 文件概述
`InMemoryPlatform` 是 `PlatformInterface` 的**测试假实现（Fake）**，无需真实 API 密钥和网络连接，通过固定字符串或 Closure 返回预设响应，专为单元测试和集成测试设计。

## 类定义
- **类型**: `class`（非 final，可子类化扩展）
- **实现**: `PlatformInterface`

## 方法分析

### `__construct(\Closure|string $mockResult)`
- `string`：所有调用返回相同字符串响应
- `\Closure`：闭包接收 `(Model $model, array|string|object $input, array $options)` 参数，动态生成响应

**闭包技巧**：闭包版本使测试可以根据不同调用参数返回不同响应，模拟真实 API 的多样性行为（如根据 input 返回不同内容、模拟错误等）。

### `invoke(string $model, ..., array $options): DeferredResult`
1. 创建匿名 `Model` 子类（内联 anonymous class，避免需要注册具体模型类）
2. 执行 mockResult（字符串或 Closure）
3. 若结果是 `ResultInterface`（Closure 返回完整 Result），直接包装
4. 否则包装为 `TextResult`
5. 返回 `DeferredResult`（使用 `PlainConverter`）

### `createDeferredResult(ResultInterface $result, array $options): DeferredResult` *(private)*
- 若 `$result` 已有 `rawResult`，复用
- 否则创建 `InMemoryRawResult`（包含 content 的简单字典）

## 设计模式
**假对象（Fake）**：有完整工作逻辑（不是 Mock），但绕过外部依赖。与 Mock（只验证调用）不同，Fake 提供真实可运行的替代实现。

**策略（Strategy）**：通过构造时传入字符串或 Closure，在不继承的情况下实现行为定制。

## 与其他文件的关系
- 实现 `PlatformInterface`
- 使用 `FallbackModelCatalog`（接受任何模型名）
- 使用 `PlainConverter`（直接传递 Result）
- 在 agent、chat 等模块的测试中广泛使用

## 使用示例
```php
// 固定响应
$platform = new InMemoryPlatform('Hello from AI!');

// 动态响应
$platform = new InMemoryPlatform(function(Model $model, $input, $options) {
    if (str_contains($input, '天气')) {
        return '北京今天晴，25°C';
    }
    return '我不知道';
});
```
