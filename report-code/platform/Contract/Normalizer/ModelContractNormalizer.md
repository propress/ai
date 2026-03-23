# ModelContractNormalizer 分析报告

## 文件概述
`ModelContractNormalizer` 是所有**模型相关的序列化器**的抽象基类，实现了基于"当前模型是否支持"的条件激活机制，是 Bridge 层 Normalizer 的核心模板。

## 类定义
- **类型**: `abstract class`，实现 `NormalizerInterface`

## 方法分析

### `supportsNormalization(mixed $data, ?string $format, array $context): bool`
双重检查：
1. 数据类型是否匹配（`is_a($data, $this->supportedDataClass())`）
2. context 中是否有当前模型，且当前模型是否被子类认可（`$this->supportsModel()`）

### `getSupportedTypes(?string $format): array`
返回 `[$this->supportedDataClass() => true]`，Symfony Serializer 用于缓存支持类型。

### `supportedDataClass(): string` *(abstract protected)*
子类声明自己处理的数据类（如 `AssistantMessage::class`）

### `supportsModel(Model $model): bool` *(abstract protected)*
子类声明自己支持哪些模型（如只支持 Claude 系列）

## 设计模式
**模板方法（Template Method）**：父类定义"检查数据类型 + 检查模型"的算法框架，子类只需实现"我处理什么类型"和"我支持什么模型"两个细节。这使 Bridge 层可以为特定模型提供特殊序列化逻辑。

## 扩展点
所有 Bridge 中需要对特定消息类型做特殊处理（如 Anthropic 的 AssistantMessage 格式与 OpenAI 不同），都通过继承此类实现。

## 与其他文件的关系
- 被 Bridge 层的各 Normalizer 继承（如 `Anthropic/Bridge/Normalizer/AssistantMessageNormalizer`）
- 依赖 `Contract::CONTEXT_MODEL`（context key）
