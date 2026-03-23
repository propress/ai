# ResultNormalizer 分析报告

## 文件概述
`ResultNormalizer` 实现 `NormalizerInterface + DenormalizerInterface`，将 `ResultInterface` 序列化为可存储的数组，并从数组反序列化恢复。设计为与 Symfony Serializer 集成，通过 `supportsNormalization/supportsDenormalization` 自动激活。

## 序列化格式
所有序列化结果统一格式：`{class: string, payload: mixed}`，根据 `class` 决定如何处理 `payload`。

## 特殊限制
`StreamResult` 无法序列化（流式响应不可持久化），调用时抛出 `InvalidArgumentException`。

## 与其他文件的关系
- 被 `CachePlatform` 构造时内置为默认序列化器
- 处理 `platform` 模块所有 Result 类型
