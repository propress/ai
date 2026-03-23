# AgentInstructionsMaterializer.php 分析报告

## 文件概述

`AgentInstructionsMaterializer` 将聚合后的 Agent 指令写入两个文件：
1. `mate/AGENT_INSTRUCTIONS.md` — 完整的指令文档
2. `AGENTS.md`（项目根目录）— 在已有文件中插入/更新由标记界定的托管块

## 类签名与依赖

```php
namespace Symfony\AI\Mate\Agent;

final class AgentInstructionsMaterializer
{
    public const AGENTS_START_MARKER = '<!-- BEGIN AI_MATE_INSTRUCTIONS -->';
    public const AGENTS_END_MARKER = '<!-- END AI_MATE_INSTRUCTIONS -->';

    public function __construct(
        private string $rootDir,
        private AgentInstructionsAggregator $aggregator,
        private LoggerInterface $logger,
    )
}
```

## 方法级别分析

### `materializeForExtensions(array $extensions): MaterializationResult`
- **输入**: 扩展数据数组
- **输出**: `array{instructions_file_updated: bool, agents_file_updated: bool}`
- **流程**: 聚合指令 → 写 AGENT_INSTRUCTIONS.md → 写 AGENTS.md 托管块

### `synchronizeFromCurrentInstructionsFile(): MaterializationResult`
- **输入**: 无（使用当前已有的指令文件）
- **输出**: 同上
- **用途**: `InitCommand` 调用，仅同步 AGENTS.md 而不重新生成指令

### `writeInstructionsFile(string $instructions): bool`
- **输入**: 完整指令文本
- **输出**: 是否写入成功
- **路径**: `$rootDir/mate/AGENT_INSTRUCTIONS.md`

### `writeAgentsFile(?array $extensions): bool`
- **输入**: 可选扩展数组
- **输出**: 是否写入成功
- **行为**: 文件不存在时创建；存在时查找标记替换托管块

### `replaceManagedBlock(string $content, string $managedBlock): string`
- **输入**: 原始文件内容和新的托管块
- **输出**: 替换后的内容
- **逻辑**:
  1. 查找 START_MARKER 和 END_MARKER 位置
  2. 标记存在 → 替换标记间内容，保留前后文
  3. 标记不存在 → 追加到文件末尾
  4. 文件为空 → 仅返回托管块

### `buildManagedBlock(?array $extensions): string`
- **输入**: 可选扩展数组
- **输出**: 完整的托管块字符串（包含 START/END 标记）
- **内容**: AI Mate 角色说明 + 已安装扩展列表

### `buildInstalledExtensionsText(?array $extensions): string`
- **输入**: 可选扩展数组
- **输出**: 扩展列表文本
- **逻辑**: 过滤掉 `_custom`，排序后用逗号连接

### `normalizeContent(string $content): string`
- **输入/输出**: 确保内容以单个换行符结尾

### `getFallbackInstructions(): string`
- **输出**: 无扩展时的默认指令文本

## 设计模式

1. **模板方法 (Template Method)**: `materializeForExtensions` 定义了固定的写入步骤
2. **标记模式 (Marker Pattern)**: 使用 HTML 注释标记界定托管区域，保护用户自定义内容
3. **幂等操作**: 多次调用产生相同结果

## 调用场景

- `InitCommand.execute()` → `synchronizeFromCurrentInstructionsFile()`
- `DiscoverCommand.execute()` → `materializeForExtensions(enabledExtensions)`

## 技巧分析

- **HTML 注释标记**: `<!-- BEGIN/END AI_MATE_INSTRUCTIONS -->` 对 Markdown 渲染不可见，但允许程序精确定位托管块
- **保护用户内容**: `replaceManagedBlock` 精心处理前缀/后缀的空白，确保用户在 AGENTS.md 中的自定义内容不受影响
- **降级策略**: 聚合无结果时使用 `getFallbackInstructions()` 而非留空

## 可扩展性

- 可添加更多输出目标（如 `.cursorrules`、`CLAUDE.md` 等 AI 工具配置文件）
- 标记模式可扩展到其他文件
- 托管块内容模板可自定义
