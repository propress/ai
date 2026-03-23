# ModelCatalogTestCase 分析报告

## 文件概述
`ModelCatalogTestCase` 是所有 Bridge ModelCatalog 测试的**抽象基类测试用例**，提供标准化的测试方法（数据驱动测试所有模型、测试未知模型异常、验证所有模型类存在），确保每个 Bridge 的模型目录都经过完整测试。

## 类定义
- **类型**: `abstract class`，继承 `TestCase`

## 方法分析

### `modelsProvider(): iterable` *(abstract static)*
子类提供测试数据：`[modelName, expectedClass, expectedCapabilities]` 三元组列表。

### `testGetModel(string $modelName, string $expectedClass, array $expectedCapabilities)` #[DataProvider]
- 验证 `getModel($name)` 返回正确的类实例
- 验证模型名称一致
- **排序后比较 Capability**（避免顺序导致测试失败）

### `testGetModelThrowsExceptionForUnknownModel()`
- 对 `FallbackModelCatalog` 跳过（它接受任意名称）
- 其他目录验证抛出 `ModelNotFoundException`

### `testGetModels()`
- 验证 `getModels()` 返回有效的模型定义数组结构

### `testAllModelsHaveValidClass()`
- 验证每个注册模型的 class-string 对应的类实际存在且继承 `Model`

### `createModelCatalog(): ModelCatalogInterface` *(abstract protected)*
子类返回被测目录实例。

## 设计模式
**抽象测试用例（Abstract TestCase）**：将公共测试逻辑提取为父类，子类只需提供"什么模型/期望什么"的数据，消除跨 34 个 Bridge 的测试代码重复。

## 扩展点
子类可以覆盖 `testGetModel()` 添加 Bridge 特定的额外断言（如验证特定选项被正确传递）。

## 与其他文件的关系
- 被每个 Bridge 的 `ModelCatalogTest` 继承
- 测试 `AbstractModelCatalog` 的所有功能
