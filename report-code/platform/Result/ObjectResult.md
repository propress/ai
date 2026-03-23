# ObjectResult 分析报告

## 文件概述
`ObjectResult` 封装结构化输出（PHP 对象或数组），是结构化输出功能的最终结果类型，由 `StructuredOutput\ResultConverter` 生成。

## 类定义
- **类型**: `final class`，继承 `BaseResult`

## 方法分析

### `__construct(object|array $structuredOutput)`
- 存储反序列化后的 PHP 对象或关联数组

### `getContent(): object|array`
- 返回结构化数据，调用方可直接当作 DTO 使用

## 设计模式
**值对象**：不可变结构化数据容器。与 `TextResult` 不同，此类明确表示"已反序列化的强类型结果"。

## 与其他文件的关系
- 被 `StructuredOutput\ResultConverter` 创建
- 继承 `BaseResult`

## 使用示例
```php
$result = $platform->invoke('gpt-4o', $messages, ['response_format' => MyDTO::class])->getContent();
if ($result instanceof ObjectResult) {
    $dto = $result->getContent(); // 直接是 MyDTO 实例
}
```
