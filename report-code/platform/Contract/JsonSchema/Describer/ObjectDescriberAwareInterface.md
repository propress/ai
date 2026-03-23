# ObjectDescriberAwareInterface 分析报告

## 文件概述
`ObjectDescriberAwareInterface` 是 Aware 接口，允许属性描述器获得对"根 ObjectDescriber"的引用，从而实现嵌套对象的递归 Schema 生成。

## 接口定义
```php
interface ObjectDescriberAwareInterface {
    public function setObjectDescriber(ObjectDescriberInterface $describer): void;
}
```

## 设计模式
**Aware 接口（服务定位器变体）**：与 Symfony 的 `LoggerAwareInterface`、`ContainerAwareInterface` 模式完全一致。在 `Describer` 构造时，对所有实现了此接口的描述器注入 `$this`（即 `Describer` 自身），形成"根描述器知道所有子描述器，子描述器也知道根描述器"的双向引用。

**为什么需要这个**：`TypeInfoDescriber` 在描述 `ObjectType` 属性时，需要递归调用根描述器来生成嵌套对象的 Schema。如果直接引用 `TypeInfoDescriber` 自己，只能描述类型，不能触发完整的描述链（包括验证、With 属性等）。

## 与其他文件的关系
- 由 `Describer::__construct()` 检测并注入
- 实现类：`TypeInfoDescriber`、`SerializerDescriber`
