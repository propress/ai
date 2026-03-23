# ChoiceResult 分析报告

## 文件概述
`ChoiceResult` 封装多个候选结果（如生成 N 张图片、N 段文本），继承 `BaseResult`，要求至少包含两个结果。

## 类定义
- **类型**: `final class`，继承 `BaseResult`
- **命名空间**: `Symfony\AI\Platform\Result`

## 方法分析

### `__construct(ResultInterface[] $results)`
- 验证 `count($results) >= 2`，否则抛出 `InvalidArgumentException`
- **设计原因**：单个结果应使用其他 Result 类型，ChoiceResult 专表"多选一"场景

### `getContent(): ResultInterface[]`
- 返回所有候选结果数组

## 设计模式
**复合对象（Composite）**：一个 Result 内部包含多个 ResultInterface，允许批量调用返回多选项。

## 扩展点
可在消费端自行实现"最佳结果选择"逻辑（按 score/长度/特定关键词筛选）。

## 与其他文件的关系
- 继承 `BaseResult`，实现 `ResultInterface`
- 内部元素为 `ResultInterface`（可以是 TextResult 等）

## 使用示例
```php
$result = $platform->invoke('gpt-4o', $input, ['n' => 3])->getContent();
if ($result instanceof ChoiceResult) {
    foreach ($result->getContent() as $choice) {
        echo $choice->getContent(); // 三个候选回答
    }
}
```
