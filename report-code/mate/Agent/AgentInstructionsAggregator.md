# AgentInstructionsAggregator.php 分析报告

## 文件概述

`AgentInstructionsAggregator` 负责从所有已安装扩展中聚合 Agent 指令文档。每个扩展可提供一个 `INSTRUCTIONS.md` 文件，描述其 MCP 工具的用法。聚合后的内容通过 MCP 协议的 `instructions` 字段在服务器握手时发送给 AI 代理。

## 类签名与依赖

```php
namespace Symfony\AI\Mate\Agent;

final class AgentInstructionsAggregator
{
    public function __construct(
        private string $rootDir,                    // 项目根目录
        private array $extensions,                  // 扩展数据映射
        private LoggerInterface $logger,            // PSR-3 日志
    )
}
```

**依赖**: `Psr\Log\LoggerInterface`, `ComposerExtensionDiscovery`（类型导入）

## 方法级别分析

### `aggregate(?array $extensions = null): ?string`
- **输入**: 可选的扩展数据数组，为 null 时使用构造函数中的默认扩展
- **输出**: 聚合后的 Markdown 字符串，无指令时返回 null
- **逻辑**: 遍历扩展 → 区分 `_custom`（根项目）和 vendor 扩展 → 读取指令文件 → 添加全局头部 → 用 `---` 分隔符连接 → 对扩展内容执行 heading 降级

### `loadExtensionInstructions(string $packageName, array $data): ?string`
- **输入**: 包名和扩展数据
- **输出**: 指令文件内容或 null
- **路径构建**: `$rootDir/vendor/$packageName/$instructionsPath`

### `loadRootProjectInstructions(array $data): ?string`
- **输入**: 根项目扩展数据
- **输出**: 指令文件内容或 null
- **路径构建**: `$rootDir/$instructionsPath`

### `readInstructionsFile(string $path, string $source): ?string`
- **输入**: 文件路径和来源标识
- **输出**: 文件内容或 null（文件不存在/读取失败/内容为空时）
- **行为**: 完整的错误处理链，每种失败情况都有对应的日志级别

### `getGlobalHeader(): string`
- **输出**: 固定的 Markdown 头部文本

### `deepenMarkdownHeadings(string $content): string`
- **输入**: Markdown 内容
- **输出**: 所有标题级别增加 1 的 Markdown（最大到 h6）
- **技巧**: 使用正则 `^(#{1,6})(\s+.*)$` 匹配标题行，避免影响代码块中的 `#`

## 设计模式

1. **聚合器模式 (Aggregator)**: 从多个来源收集内容合并为一个结果
2. **空对象返回**: 各层方法在失败时返回 null 而非抛异常，调用方统一处理

## 调用场景

- `ServeCommand.execute()` → `aggregate()` → 结果设置到 MCP Server 的 instructions 字段
- `AgentInstructionsMaterializer.materializeForExtensions()` → `aggregate()` → 写入文件

## 可扩展性

- 可替换指令文件格式（当前仅支持 Markdown）
- `deepenMarkdownHeadings` 可定制嵌套深度
- 可添加指令内容的后处理/模板化逻辑

## 技巧分析

- **Heading 降级**: 将扩展的 `# h1` 变成 `## h2`，保证聚合后的文档层级正确，全局标题为 h2，扩展内容至少为 h3
- **`_custom` 特殊处理**: 根项目的路径解析不包含 vendor/ 前缀，与第三方包区分
