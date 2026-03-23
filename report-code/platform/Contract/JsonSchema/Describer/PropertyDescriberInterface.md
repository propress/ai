# PropertyDescriberInterface 分析报告

## 文件概述
`PropertyDescriberInterface` 定义了"如何将单个属性描述为 JSON Schema 片段"的契约，是 Schema 生成链中属性级别的核心接口。

## 接口定义
```php
interface PropertyDescriberInterface {
    public function describeProperty(PropertySubject $subject, ?array &$schema): void;
}
```

## 方法分析

### `describeProperty(PropertySubject $subject, ?array &$schema): void`
- **参数**：
  - `$subject`：被描述的属性主题（名称+反射器）
  - `&$schema`：引用传递，描述器叠加/修改属性 Schema 片段
- **返回**：`void`，通过 by-ref 修改 `$schema`

**设计技巧**：`$schema` 初始为 `null`，第一个描述器设置基本类型（如 `['type' => 'string']`），后续描述器叠加约束（如 `minLength`）。这是"渐进式构建"模式，每个描述器只关注自己知道的部分。

## 设计模式
**责任链（Chain of Responsibility）**：`Describer` 按顺序调用所有 `PropertyDescriberInterface`，每个描述器处理自己能处理的部分，无法处理的直接 return。

## 与其他文件的关系
- 实现类：`Describer`、`TypeInfoDescriber`、`PropertyInfoDescriber`、`MethodDescriber`、`ValidatorConstraintsDescriber`、`WithAttributeDescriber`
- 被 `Describer::describeProperty()` 驱动调用
