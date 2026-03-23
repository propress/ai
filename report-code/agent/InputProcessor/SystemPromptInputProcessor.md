# InputProcessor/SystemPromptInputProcessor.php 分析报告

## 概述

`SystemPromptInputProcessor` 负责在调用 AI 平台前向 `MessageBag` 注入系统提示（System Message），支持字符串、`Stringable`、可翻译接口（`TranslatableInterface`）以及文件（`File`）四种提示来源，并可选地将 Toolbox 中工具的描述拼接到提示末尾。

## 关键方法分析

### 构造函数

```php
public function __construct(
    private readonly \Stringable|TranslatableInterface|string|File $systemPrompt,
    private readonly ?ToolboxInterface $toolbox = null,
    private readonly ?TranslatorInterface $translator = null,
    private readonly LoggerInterface $logger = new NullLogger(),
)
```

- 若 `$systemPrompt` 是 `TranslatableInterface` 但未提供 `$translator`，立即抛出 `RuntimeException`，实现早期失败校验。

### processInput(Input $input): void

1. **幂等检查**：若 `MessageBag` 中已存在系统消息，则跳过注入（记录调试日志），避免重复。
2. **提示内容解析**：
   - `File`：调用 `asBinary()` 读取文件内容。
   - `TranslatableInterface`：调用 `trans($translator)` 翻译。
   - 其他：强制转换为字符串。
3. **工具描述拼接（可选）**：若 `$toolbox` 非空且有工具，将每个工具的名称和描述以 Markdown 格式拼接到提示末尾。
4. **注入系统消息**：调用 `MessageBag::withSystemMessage()` 创建带系统消息的新消息包，并更新 `Input`。

## 工具描述格式

```
{原始系统提示}

# Tools

The following tools are available to assist you in completing the user's request:

## {tool.name}
{tool.description}

## {tool.name2}
...
```

此格式适用于不支持原生工具调用 API 的模型，通过系统提示告知工具能力。

## 设计模式

- **策略模式（Strategy）**：通过构造参数注入不同类型的系统提示来源，避免条件分支散落各处。
- **幂等性保护**：检查已有系统消息，确保多次处理不产生副作用。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `InputProcessorInterface` | 实现此接口 |
| `Input` | 读取/修改 `MessageBag` |
| `ToolboxInterface` | 可选依赖，用于获取工具描述列表 |
| `Exception/RuntimeException` | 翻译器缺失时的早期失败 |
