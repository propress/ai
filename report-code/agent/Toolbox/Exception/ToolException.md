# Toolbox/Exception/ToolException.php 分析报告

## 概述

`ToolException` 表示工具定义无效的异常（如类不存在、缺少 `#[AsTool]` 注解），继承自 `Agent\Exception\InvalidArgumentException`，提供两个命名静态构造器。

## 静态构造器

### invalidReference(mixed $reference): self

工具引用无效时使用（如传入的类名不存在，或 `MemoryToolFactory` 中未找到已注册工具）：
`'The reference "..." is not a valid tool.'`

### missingAttribute(string $className): self

类存在但未标注 `#[AsTool]` 注解时使用：
`'The class "..." is not a tool, please add AsTool attribute.'`

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Agent/Exception/InvalidArgumentException` | 继承此类 |
| `Toolbox/Exception/ExceptionInterface` | 实现此接口 |
| `ToolFactory/ReflectionToolFactory` | 抛出此异常（引用无效或缺少注解） |
| `ToolFactory/MemoryToolFactory` | 抛出此异常（引用未注册） |
| `ToolFactory/ChainFactory` | 捕获此异常（决定是否继续尝试下一工厂） |
| `ToolConfigurationException` | 继承此类（更具体的配置错误） |
