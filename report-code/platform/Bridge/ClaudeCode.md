# ClaudeCode Bridge 分析报告

## 概述

ClaudeCode Bridge 将 Anthropic 的 Claude Code CLI/SDK 集成到 Symfony AI Platform 中。与其他桥接器不同，**它不通过 HTTP 与 API 通信，而是通过 Symfony Process 组件将 `claude` CLI 作为子进程启动**，以 `stream-json` 格式接收输出，然后解析结果。该桥接器适合在本地环境中利用 Claude Code 进行代码辅助、问答等任务。

---

## 目录结构

```
ClaudeCode/
├── ClaudeCode.php                       # 模型标识类（继承 Model）
├── ModelCatalog.php                     # 静态模型目录（opus / sonnet / haiku）
├── ModelClient.php                      # 核心客户端，启动 claude CLI 子进程
├── PlatformFactory.php                  # Platform 工厂方法
├── RawProcessResult.php                 # 封装 Symfony Process，实现 RawResultInterface
├── ResultConverter.php                  # 将 CLI 输出转换为 TextResult / StreamResult
├── TokenUsageExtractor.php              # 从 CLI 结果中提取 Token 用量
├── Contract/
│   ├── ClaudeCodeContract.php           # Contract 工厂，注册 MessageBagNormalizer
│   └── MessageBagNormalizer.php         # 将 MessageBag 序列化为 CLI 所需的 prompt/system_prompt
└── Exception/
    └── CliNotFoundException.php         # CLI 二进制不可用时抛出的异常
```

---

## 关键设计模式

### 1. 子进程执行（非 HTTP）
`ModelClient` 使用 `Symfony\Component\Process\Process` 启动 `claude` CLI，区别于其他桥接器使用 `HttpClientInterface`。进程以非阻塞方式启动，输出由 `RawProcessResult` 封装处理。

### 2. 流式 JSON 解析
CLI 以 `--output-format stream-json` 模式运行，每行输出一条 JSON 对象。`RawProcessResult::getDataStream()` 通过轮询（10ms 间隔）逐行 yield 解析后的数据；`getData()` 等待进程结束后提取 `type=result` 的最终结果行。

### 3. 选项到 CLI 参数映射
`ModelClient::OPTION_FLAG_MAP` 将 PHP 选项键映射到对应的 CLI 参数（如 `tools` → `--allowedTools`），并支持布尔值、数组等多种值类型。

### 4. Contract / Normalizer 分离
`MessageBagNormalizer` 继承 `ModelContractNormalizer`，仅在模型为 `ClaudeCode` 实例时生效，将消息包中最后一条用户消息提取为 `prompt`，系统消息提取为 `system_prompt`。

### 5. Token 用量提取
`TokenUsageExtractor` 从 CLI 输出的 `usage` 字段提取 `input_tokens`、`output_tokens` 以及缓存 Token（`cache_creation_input_tokens` + `cache_read_input_tokens`）。

---

## 组件关系图

```
PlatformFactory
    └── Platform
            ├── ModelClient          ← 启动 claude CLI 子进程
            ├── ResultConverter      ← 解析 stream-json 输出
            ├── ModelCatalog         ← 提供 opus/sonnet/haiku 三个模型
            └── ClaudeCodeContract
                    └── MessageBagNormalizer  ← 序列化 MessageBag

ModelClient → RawProcessResult (封装 Symfony Process)
ResultConverter → TextResult | StreamResult
TokenUsageExtractor → TokenUsage
Exception: CliNotFoundException (CLI 未找到时)
```

---

## 支持的模型

| 模型名称  | 输入能力                        | 输出能力                    |
|-----------|-------------------------------|---------------------------|
| `opus`    | INPUT_MESSAGES, INPUT_TEXT    | OUTPUT_TEXT, OUTPUT_STREAMING |
| `sonnet`  | INPUT_MESSAGES, INPUT_TEXT    | OUTPUT_TEXT, OUTPUT_STREAMING |
| `haiku`   | INPUT_MESSAGES, INPUT_TEXT    | OUTPUT_TEXT, OUTPUT_STREAMING |

---

## 外部依赖

- **Symfony Process 组件**：用于启动 `claude` CLI 子进程
- **claude CLI**：需通过 `npm install -g @anthropic-ai/claude-code` 安装
- **PSR-3 Logger**：可选注入，用于记录子进程命令信息
