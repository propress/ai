# ObjectDescriberInterface 分析报告

## 文件概述
`ObjectDescriberInterface` 定义了"如何将一个对象/类/方法描述为 JSON Schema"的契约，是 Schema 生成链中对象级别的核心接口。

## 接口定义
```php
interface ObjectDescriberInterface {
    public function describeObject(ObjectSubject $subject, ?array &$schema): iterable;
}
```

## 方法分析

### `describeObject(ObjectSubject $subject, ?array &$schema): iterable<PropertySubject>`
- **参数**：
  - `$subject`：被描述的对象主题（包含类名和反射器）
  - `&$schema`：引用传递的 Schema 数组（by-reference，描述器直接修改）
- **返回值**：`iterable<PropertySubject>`，描述器发现的需要进一步描述的属性列表
- **设计技巧**：by-reference 参数允许描述器在不创建新数组的情况下直接修改 Schema，避免值拷贝开销；返回 `iterable` 允许使用 `yield` 懒生成属性列表

## 设计模式
**策略模式（Strategy）+ 组合模式（Composite）**：`Describer` 类同时实现此接口，并持有多个 `ObjectDescriberInterface` 实例，形成递归组合。每个实现可以专注于一个方面（类型、序列化、鉴别器等）。

## 与其他文件的关系
- 实现类：`Describer`、`TypeInfoDescriber`、`SerializerDescriber`、`PropertyInfoDescriber`、`MethodDescriber`
- 被 `Factory::buildProperties()` 和 `Factory::buildParameters()` 调用
- 与 `PropertyDescriberInterface` 配对使用
