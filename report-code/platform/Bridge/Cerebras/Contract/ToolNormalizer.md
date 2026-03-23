# Contract/ToolNormalizer.php 分析报告

## 概述

`ToolNormalizer` 继承自平台基础 `ToolNormalizer`，专门修复 Cerebras API 的工具定义兼容性问题：当工具函数未声明任何参数时，Cerebras 要求 `parameters` 字段必须存在（即使为空对象），而默认归一化器可能不会输出此字段。

## 关键方法分析

### `normalize(mixed $data, ?string $format, array $context): array`
调用父类 `normalize()` 获取基础工具定义数组，随后通过 null 合并赋值操作符确保 `function.parameters` 字段存在：

```php
$array['function']['parameters'] ??= ['type' => 'object'];
```

若父类已输出 `parameters` 字段，则保持不变；若缺失，则补充默认的 `{'type': 'object'}` 空对象结构。

## 设计模式

- **装饰器模式（Decorator）**：在父类输出结果上进行最小化修补，不改变整体序列化流程
- **防御性补全**：使用 `??=` 确保幂等性，不覆盖已有合法数据

## 与其他类的关系

- 由 `Cerebras\PlatformFactory` 在构建 `Contract` 时注入：`Contract::create(new ToolNormalizer())`
- 父类为 `Symfony\AI\Platform\Contract\Normalizer\ToolNormalizer`
- 仅影响工具调用场景，对无工具的请求序列化无影响
