# Describer 分析报告

## 文件概述
`Describer` 是 JSON Schema 生成链的**组合根（Composition Root）**，同时实现 `ObjectDescriberInterface` 和 `PropertyDescriberInterface`，内部聚合多个子描述器并协调它们的执行顺序，是整个 Schema 生成系统的中枢。

## 类定义
- **类型**: `final class`，实现 `ObjectDescriberInterface` + `PropertyDescriberInterface`
- **命名空间**: `Symfony\AI\Platform\Contract\JsonSchema\Describer`

## 方法分析

### `__construct(?iterable $describers = null)`
- 默认描述器链（按顺序）：
  1. `SerializerDescriber` — 处理 Symfony Serializer 元数据（鉴别器映射、DateTime 等）
  2. `TypeInfoDescriber` — 将 PHP 类型映射为 JSON Schema 类型
  3. `MethodDescriber` — 从方法 PHPDoc 提取参数说明
  4. `PropertyInfoDescriber` — 通过 PropertyInfo 组件提取属性列表和说明
  5. `ValidatorConstraintsDescriber`（条件加载，仅当 symfony/validator 存在）— 将验证约束转为 Schema 约束
  6. `WithAttributeDescriber` — 处理 `#[With]` 属性（**最后执行，优先级最高**）
- 构造时分离 `objectDescribers` 和 `propertyDescribers` 两个列表
- 对实现 `ObjectDescriberAwareInterface` 的描述器注入 `$this`

**技巧**：使用 `class_exists(ValidatorInterface::class)` 条件判断是否加载验证约束描述器，实现"可选依赖"——安装了 symfony/validator 才自动激活，无需额外配置。

### `describeObject(ObjectSubject $subject, ?array &$schema): iterable`
- 依次调用所有 `objectDescribers`
- 对每个 `yield` 出来的 `PropertySubject` 调用 `describeProperty()`
- 收集 `required` 字段列表
- **边界处理**：若 Schema 仅为 `['type' => 'object']`（没有任何属性），则置为 `null`（表示无法描述）
- 若有必须属性：设置 `required` 数组并添加 `additionalProperties: false`（OpenAI strict mode 要求）

### `describeProperty(PropertySubject $subject, ?array &$schema): void`
- 依次调用所有 `propertyDescribers`，每个描述器叠加自己知道的部分

## 设计模式
**组合模式（Composite）**：`Describer` 自身是 `ObjectDescriberInterface` 的实现，但其行为是聚合多个子描述器。对外表现为单一描述器，对内是有序描述器链。

**职责链（Chain of Responsibility）**：属性描述阶段，每个 `PropertyDescriberInterface` 独立处理自己关注的方面，互不干扰。

## 扩展点
通过构造函数注入自定义 `$describers`，可以：
- 移除不需要的描述器（如去掉 `ValidatorConstraintsDescriber` 减少依赖）
- 添加自定义描述器（如从数据库元数据、OpenAPI 注解读取约束）
- 改变执行顺序（后执行的描述器优先级更高，因为使用 `array_replace_recursive`）

## 与其他文件的关系
- 被 `Factory` 创建并使用
- 包含所有具体描述器
- 向所有 `ObjectDescriberAwareInterface` 注入自身
