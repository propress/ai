# OpenResponses Bridge 分析报告

## 文件概述
`OpenResponses` Bridge 实现 **OpenAI Responses API** 的通用支持，是 `OpenAi\Gpt` 的底层实现，同时也可被其他兼容 Responses API 的服务使用（如 Azure Responses、未来可能兼容的服务）。

## 目录结构（28个PHP文件）
```
OpenResponses/
├── ResponsesModel.php             # Responses API 模型标记类
├── ModelCatalog.php               # 显式注册模型
├── FallbackModelCatalog.php       # 自动创建 ResponsesModel（全能力）
├── ModelClient.php                # POST /v1/responses（Responses API 专用）
├── PlatformFactory.php            # 可配置工厂（supportResponsesPath）
├── ResultConverter.php            # 解析 output[] 数组（Responses API 格式）
├── TokenUsageExtractor.php        # 提取 input/output_tokens（Responses API 字段）
└── Contract/
    ├── OpenResponsesContract.php  # 注册所有 Responses API Normalizer
    ├── ToolNormalizer.php         # Tool → {type:'function', name, description, parameters}
    ├── ToolCallNormalizer.php     # ToolCall → {type:'function_call', call_id, name, arguments}
    └── Message/
        ├── MessageBagNormalizer.php             # → {input:[], instructions?:}
        ├── AssistantMessageNormalizer.php       # → {role:'assistant', type:'message'}
        ├── ToolCallMessageNormalizer.php        # → {type:'function_call_output', call_id, output}
        └── Content/
            ├── TextNormalizer.php       # → {type:'input_text', text}
            ├── ImageNormalizer.php      # → {type:'input_image', image_url}
            ├── ImageUrlNormalizer.php   # → {type:'input_image', image_url}
            └── DocumentNormalizer.php   # → {type:'input_file', filename, file_data}
```

## Responses API 格式特征

### 请求格式（与旧 Chat Completions 对比）
| 场景 | Chat Completions | Responses API |
|---|---|---|
| 消息传入 | `messages[]` | `input[]` |
| 系统消息 | `messages[{role:system}]` | 顶层 `instructions` 字段 |
| 工具调用结果 | `{role:tool, tool_call_id}` | `{type:function_call_output, call_id}` |
| 工具调用 | `choices[0].message.tool_calls[].id` | `output[].id（function_call 类型）` |
| 结构化输出 | `response_format.json_schema` | `text.format.json_schema` |
| 文件上传 | 无直接支持 | `{type:input_file, filename, file_data}` |

### 响应格式
```json
{
  "output": [
    {"type": "message", "content": [{"type":"output_text","text":"..."}]},
    {"type": "function_call", "id":"...", "name":"...", "arguments":"{}"},
    {"type": "reasoning", "summary": [{"text":"..."}]}
  ]
}
```

## ModelClient 特殊处理
与 `Generic\Completions\ModelClient` 的区别：
1. **结构化输出格式转换**：`response_format` → `text.format`
2. **无 cacheRetention 过滤**（这是 OpenResponses 自己的 ModelClient）
3. **可选 API Key**：`null !== $this->apiKey` 检查，支持无认证本地服务

## ResponsesModel vs CompletionsModel
- `ResponsesModel` → `ModelClient` → `/v1/responses`（Responses API）
- `CompletionsModel` → `Generic\Completions\ModelClient` → `/v1/chat/completions`（Chat API）

两者使用完全不同的端点和格式，是 Platform 中并存的两个主要 LLM 范式。

## AssistantMessageNormalizer 的特殊逻辑
Responses API 格式中，**AssistantMessage 含工具调用时**需要将工具调用规范化为 input[] 数组中的多个 function_call 条目，而非单个 message 对象，因此委托给 ToolCallNormalizer 处理：
```php
if ($data->hasToolCalls()) {
    return $this->normalizer->normalize($data->getToolCalls(), ...); // 返回数组
}
```
`MessageBagNormalizer` 检测到这种情况时用 `array_merge` 而非 `[]= ` 展开。

## 与其他 Bridge 的关系
- `OpenAi\Gpt` 继承 `ResponsesModel`，使用 `OpenAi\Gpt\ModelClient`（而非 `OpenResponses\ModelClient`）
- `OpenAi\Contract\OpenAiContract` 继承 `OpenResponsesContract` 并添加 `AudioNormalizer`
- `Azure\Responses` 使用 `OpenResponses` Bridge 的 ModelClient
