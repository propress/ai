# Tool 分析报告

## 文件概述
`Tool` 是 Function Calling 工具的**数据容器**，封装工具的元数据（名称、描述、参数 Schema 和执行引用），在传递给 AI 模型时被 `ToolNormalizer` 序列化为标准格式。

## 类定义
- **类型**: `final class`（值对象）

## 属性
| 属性 | 类型 | 说明 |
|---|---|---|
| `$reference` | `ExecutionReference` | 指向执行此工具的 PHP 类/方法 |
| `$name` | `string` | 工具名（AI 调用时使用的函数名） |
| `$description` | `string` | 工具说明（AI 理解工具用途的关键） |
| `$parameters` | `?array` (JsonSchema) | 参数 JSON Schema（null 表示无参数） |

## 方法分析
所有方法为标准 getter，无副作用：
- `getReference(): ExecutionReference`
- `getName(): string`
- `getDescription(): string`
- `getParameters(): ?array`

## 设计模式
**值对象（Value Object）**：不可变，只携带数据。`Tool` 的"执行"逻辑完全不在此类中——`ExecutionReference` 只是一个指针，实际执行由 Agent 模块中的 `ToolExecutor` 负责。这实现了 Tool 的定义与执行的关注点分离。

## 与其他文件的关系
- 被 `ToolNormalizer` 序列化（传给 AI 模型）
- `$reference` 指向 `ExecutionReference`
- `$parameters` 由 `Contract\JsonSchema\Factory::buildParameters()` 生成
- 由 Agent 模块的 `ToolBox` 或 `ToolFactory` 创建

## 使用示例
```php
$tool = new Tool(
    reference: new ExecutionReference(WeatherTool::class, '__invoke'),
    name: 'get_weather',
    description: '获取指定城市的天气信息',
    parameters: $factory->buildParameters(WeatherTool::class, '__invoke'),
);
```
