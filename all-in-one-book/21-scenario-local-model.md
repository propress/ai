# 第 21 章：实战 —— 本地模型私有化部署

## 学习目标

- 理解本地模型私有化部署的适用场景和优势
- 掌握 Ollama 的安装与模型管理
- 学会使用 Ollama Bridge 将本地模型接入 Symfony AI
- 实现本地 RAG 数据零泄漏方案
- 掌握混合部署策略（本地 + 云端）

## 前置知识

- 熟悉 Symfony AI Platform 的基本使用
- 了解 Docker 或本地服务管理
- 了解 GPU/CPU 推理的基本概念
- 了解 RAG（检索增强生成）基本流程

## 业务场景描述

某些场景（数据安全、合规要求、离线运行、成本控制）需要使用本地模型，避免数据外传。Ollama 让本地运行大模型变得简单。

**典型应用**：金融、医疗、政府等数据敏感行业；离线环境；内部开发工具。

## 架构概述

```text
本地模型部署架构
══════════════

  ┌──────────────────────────────────────────────────────────┐
  │  企业内网                                                  │
  │                                                          │
  │  ┌──────────┐     ┌──────────────┐     ┌──────────────┐ │
  │  │ Symfony   │────▶│ Ollama Server │────▶│ 本地 GPU/CPU  │ │
  │  │ 应用      │     │ :11434        │     │ Llama 3.1    │ │
  │  │           │◀────│              │◀────│ Mistral      │ │
  │  └──────────┘     └──────────────┘     │ nomic-embed  │ │
  │                                        └──────────────┘ │
  │  数据完全不出内网 ✅                                        │
  └──────────────────────────────────────────────────────────┘

  vs. 云端模型：

  ┌──────────┐     ┌─────── 互联网 ───────┐     ┌──────────┐
  │ Symfony   │────▶│                      │────▶│ OpenAI   │
  │ 应用      │◀────│  数据通过互联网传输 ⚠️  │◀────│ API      │
  └──────────┘     └──────────────────────┘     └──────────┘
```

## 环境准备

### 安装 Ollama

```bash
# macOS / Linux
curl -fsSL https://ollama.com/install.sh | sh

# 下载模型（根据需求选择）
ollama pull llama3.1          # 对话模型（8B 参数，需要 ~4.7GB 显存）
ollama pull mistral           # 另一个优秀的对话模型
ollama pull nomic-embed-text  # Embedding 模型（RAG 必需）
ollama pull codellama         # 代码专用模型

# 验证安装
ollama list    # 查看已下载的模型
ollama serve   # 启动服务器（默认 :11434）
```

## 核心实现

### 使用 Ollama Bridge

```php
<?php

use Symfony\AI\Platform\Bridge\Ollama\PlatformFactory;

// Ollama 默认运行在 http://localhost:11434
$platform = PlatformFactory::create();

// 使用本地模型——API 与云端模型完全一致
$response = $platform->invoke(
    'llama3.1',   // 模型名就是 ollama pull 时的名字
    $messages,
);

echo $response->asText();
```

### 本地 RAG——数据零泄漏

```php
use Symfony\AI\Platform\Bridge\Ollama\PlatformFactory;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Bridge\Postgres\Store;

// 使用本地平台——所有数据处理在本地完成
$platform = PlatformFactory::create();

// 向量化文档——使用本地 Embedding 模型
$vectorizer = new Vectorizer($platform, 'nomic-embed-text');
$vectorizer->vectorize($documents);

// 存储到本地 PostgreSQL
$store = new Store($connectionPool);
$store->add($documents);

// 查询也使用本地 LLM
$agentProcessor = new AgentProcessor($toolbox);
$agent = new Agent($platform, 'llama3.1', [$agentProcessor], [$agentProcessor]);
$response = $agent->call($messages);

// 整个流程中，数据完全不离开本地网络
```

### 混合部署：本地 + 云端

```php
// 敏感数据处理用本地模型——确保数据安全
$localPlatform = OllamaFactory::create();

// 非敏感的通用任务用云端模型——质量更好
$cloudPlatform = OpenAiFactory::create($_ENV['OPENAI_API_KEY']);

// 策略 1：根据数据敏感性选择平台
$platform = $this->isSensitiveData($input) ? $localPlatform : $cloudPlatform;

// 策略 2：使用 FailoverPlatform，云端优先、本地兜底
// 好处：即使云端 API 故障，服务也不中断
$platform = new FailoverPlatform(
    platforms: [
        $cloudPlatform,    // 优先：质量好
        $localPlatform,    // 兜底：永远可用（本地服务器不会宕机）
    ],
    rateLimiterFactory: $rateLimiterFactory,
);
```

## 运行与验证

1. 启动 Ollama 服务：`ollama serve`
2. 确认模型可用：`ollama list`
3. 运行 PHP 脚本调用本地模型，验证返回结果
4. 测试 RAG 流程：索引文档 → 向量化 → 查询，确认全程无外网请求
5. 混合部署场景：分别测试敏感/非敏感数据路由是否正确

## 错误处理

- **Ollama 服务未启动**：捕获连接异常，提示用户运行 `ollama serve`
- **模型未下载**：检查 `ollama list` 输出，提示用户拉取所需模型
- **显存不足**：选择更小的模型（如 Phi-3 3.8B）或使用 CPU 推理模式
- **混合部署中云端失败**：FailoverPlatform 自动切换到本地模型

## 生产环境注意事项

### 本地模型选型指南

| 模型 | 参数量 | 显存需求 | 特长 | 推荐用途 |
|------|:------:|:-------:|------|---------|
| Llama 3.1 8B | 8B | ~5GB | 通用对话 | 一般问答 |
| Llama 3.1 70B | 70B | ~40GB | 复杂推理 | 专业分析 |
| Mistral 7B | 7B | ~4GB | 效率高 | 快速响应 |
| CodeLlama | 7-34B | 4-20GB | 代码生成 | 开发辅助 |
| nomic-embed-text | - | ~0.3GB | Embedding | RAG 向量化 |
| Phi-3 | 3.8B | ~2.4GB | 轻量级 | 资源受限环境 |

## 扩展方向

- 使用 Docker 容器化部署 Ollama，便于在集群中横向扩展
- 结合模型量化技术（GGUF 格式）降低显存需求
- 实现模型热切换：根据任务复杂度动态选择不同参数量的模型
- 搭建内部模型注册中心，统一管理企业内多个 Ollama 实例

## 完整源代码

本章涉及的完整代码片段已在各小节中给出。核心要点：

1. 使用 `PlatformFactory::create()` 创建本地 Ollama 平台
2. 本地 RAG 全流程（Vectorizer → Store → Agent）确保数据零泄漏
3. 混合部署通过 FailoverPlatform 实现云端优先、本地兜底

## 下一步

下一个场景我们将学习如何构建生产级内容审核流水线——参见 [第 22 章：实战 —— 内容审核流水线](22-scenario-content-moderation.md)。
