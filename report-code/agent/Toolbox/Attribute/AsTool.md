# Toolbox/Attribute/AsTool.php 分析报告

## 概述

`#[AsTool]` 是 PHP 8 属性，标注在工具类上，声明该类（或其某个方法）是一个可被 LLM 调用的工具，提供工具的名称、描述和入口方法。`ReflectionToolFactory` 通过反射读取此注解来生成工具元数据。

## 关键参数分析

### name: string

工具的唯一标识名称，LLM 在工具调用时使用此名称引用工具（必填）。

### description: string

工具功能的自然语言描述，LLM 根据此描述判断何时应该使用该工具（必填）。

### method: string（默认 `'__invoke'`）

工具的入口方法名。默认使用 `__invoke`（类本身就是可调用对象），也可指定具体方法名（如 `Filesystem` 使用 `'read'`、`'write'` 等分别对应不同操作）。

## 属性特性

- `TARGET_CLASS`：只能标注在类上。
- `IS_REPEATABLE`：同一类可重复标注，允许一个类暴露多个工具方法（如 `Filesystem` 有 10 个 `#[AsTool]` 声明）。

## 使用示例

```php
// 简单工具：使用 __invoke
#[AsTool('search_web', 'Search the internet for information')]
final class WebSearch
{
    public function __invoke(string $query): array { ... }
}

// 多方法工具
#[AsTool('file_read', 'Read file content', method: 'read')]
#[AsTool('file_write', 'Write file content', method: 'write')]
final class FileManager
{
    public function read(string $path): string { ... }
    public function write(string $path, string $content): void { ... }
}
```

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `ToolFactory/ReflectionToolFactory` | 读取此注解并生成 `Tool` 元数据 |
| `Exception/ToolException` | 类上缺少此注解时由 `ReflectionToolFactory` 抛出 |
| Bridge 工具（Brave、Wikipedia 等） | 均通过此注解注册工具 |
