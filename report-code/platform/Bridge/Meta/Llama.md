# Llama.php 分析报告

## 概述

`Llama` 是 Meta Bridge 的模型标记类，继承自平台基础 `Model` 类，不添加任何新成员，仅作为类型标识符，用于 `MessageBagNormalizer::supportsModel()` 中的 `instanceof` 类型检查。

## 关键设计分析

### 标记类（Marker Class）
类体仅有类定义，无任何属性或方法。通过 PHP 的类型系统而非字符串匹配或枚举来识别 Llama 模型：
```php
protected function supportsModel(Model $model): bool
{
    return $model instanceof Llama; // 关键路由判断
}
```

### 与基础 Model 的关系
继承 `Symfony\AI\Platform\Model`，获得以下基础功能：
- `getName()`：返回模型标识符（如 `llama-3.3-70B-Instruct`）
- `getCapabilities()`：返回能力列表（由 `ModelCatalog` 在注册时注入）
- `supports(Capability $capability)`：检查特定能力

## 设计模式

- **标记类模式（Marker Class）**：零代码开销的类型标识机制
- **开放封闭原则**：新增对 Llama 模型的特殊处理只需检查此类型，无需修改平台核心

## 与其他类的关系

- `ModelCatalog` 将所有 Llama 模型的 `class` 字段指定为此类
- `MessageBagNormalizer::supportsModel()` 通过 `instanceof Llama` 检查激活 Llama 格式序列化
- 与 `Cerebras\Model` 的设计模式完全一致
