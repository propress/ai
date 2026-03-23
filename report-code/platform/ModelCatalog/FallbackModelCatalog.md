# FallbackModelCatalog 分析报告

## 文件概述
`FallbackModelCatalog` 是"接受任意模型名"的兜底目录实现，接受所有模型名称并为每个模型赋予全部 Capability，适用于动态模型平台（如 Ollama、Docker Model Runner）。

## 类定义
- **类型**: `class`，继承 `AbstractModelCatalog`

## 方法分析

### `__construct()`
- 初始化 `$this->models = []`（空注册表）

### `getModel(string $modelName): Model`
- **覆盖父类**：不查表，直接解析名称后创建 `new Model($name, Capability::cases(), $options)`
- `Capability::cases()` 返回所有已知能力，因为无法预知动态模型的实际能力

## 设计模式
**空对象（Null Object）变体**：与其在没有目录时报错，提供一个"总是成功"的实现，让使用者无需特殊处理。

**为什么赋予全部能力**：对于 Ollama 等本地平台，模型列表是动态的，运行时才知道，但调用者可能需要能力检查（如 `supports(Capability::INPUT_VISION)`），赋予全部能力等于"乐观假设"，让调用者自行尝试。

## 扩展点
可以覆盖此类，连接到动态 API（如 Ollama 的 `/api/tags`）动态获取真实能力列表。

## 与其他文件的关系
- 被 `InMemoryPlatform` 使用（测试场景）
- 被 Ollama、DockerModelRunner 等 Bridge 使用
