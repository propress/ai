# 第 13 章：实战 —— 多语言翻译助手

## 学习目标

通过本章实战，你将掌握：
- 使用 Prompt Engineering 构建专业翻译助手
- 设计包含术语表和翻译规则的 System Prompt
- 结合 StructuredOutput 获取翻译质量评估
- 配合 Chat 组件实现多轮翻译修正

## 前置知识

<!-- 建议在正式学习前回顾以下章节 -->
- Platform 组件基础用法
- System Prompt 设计技巧
- StructuredOutput（用于结构化翻译结果）
- Chat 组件（用于多轮修正）

## 业务场景描述

构建一个支持多语言翻译的 AI 助手，能保持术语一致性，支持上下文感知翻译和多轮修正。

**典型应用**：技术文档翻译、产品本地化、多语言客服、学术论文翻译。

## 架构概述

翻译助手的核心是精心设计的 System Prompt——它定义了翻译规则、术语表和质量标准。这是 **Prompt Engineering** 的经典应用：

```text
┌─────────────────────────────────────────┐
│          System Prompt（翻译指令）         │
│                                         │
│  角色定义：专业技术文档翻译专家              │
│  翻译规则：7 条核心规则                    │
│  术语表：Platform→平台, Agent→智能代理...  │
│  质量标准：信达雅                          │
│  禁止事项：不翻译代码、变量名               │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│          User Message（待翻译内容）        │
│                                         │
│  "The Agent component provides a        │
│   framework for building AI agents..."  │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│          AI 翻译结果                      │
│                                         │
│  "Agent 组件为构建 AI 智能代理提供了       │
│   一个完整的框架..."                      │
│  （术语一致、格式保留、代码块不翻译）       │
└─────────────────────────────────────────┘
```

## 环境准备

<!-- 安装所需依赖 -->

```bash
composer require symfony/ai-platform symfony/ai-open-ai-platform
# 多轮修正还需要：
# composer require symfony/ai-chat symfony/ai-redis-message-store
```

## 核心实现

```php
<?php

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Bridge\OpenAi\Gpt;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);

$systemPrompt = <<<PROMPT
你是一个专业的技术文档翻译专家，专注于将英文技术文档翻译为中文。

翻译规则：
1. 保持技术术语的准确性——参照术语表
2. 代码块（```）中的内容不翻译
3. 变量名、类名、方法名不翻译
4. 保持 Markdown 格式完整
5. 长句拆分为短句，增强可读性
6. 保留原文中的链接和引用
7. 不确定的翻译加注释标注

术语表（必须严格遵守）：
   - Platform → 平台
   - Agent → 智能代理
   - Store → 存储
   - Bridge → 桥接器
   - Toolbox → 工具箱
   - Processor → 处理器
   - MessageBag → 消息包
   - Embedding → 嵌入向量
PROMPT;

$messages = new MessageBag(
    Message::forSystem($systemPrompt),
    Message::ofUser(
        "请将以下英文翻译为中文：\n\n"
        ."The Agent component provides a framework for building AI agents "
        ."that can use tools to accomplish tasks. The Toolbox manages all "
        ."available tools and their execution. When the Agent receives a "
        ."user message, it sends it to the LLM along with the tool definitions."
    ),
);

$response = $platform->invoke('gpt-4o', $messages, [
    'temperature' => 0.3,  // 低 temperature 保证翻译一致性
]);

echo $response->asText();
```

### 结构化翻译结果

```php
class TranslationResult
{
    public function __construct(
        #[With(description: '翻译后的完整文本')]
        public readonly string $translation,
        #[With(description: '翻译质量自评分 1-10，10 为最高')]
        public readonly int $qualityScore,
        #[With(description: '翻译过程中的注意事项和不确定之处')]
        public readonly array $notes,
        #[With(description: '使用到的术语映射列表')]
        public readonly array $glossaryUsed,
    ) {}
}

$response = $platform->invoke($model, $messages, [
    'response_format' => TranslationResult::class,
    'temperature' => 0.3,
]);

$result = $response->asObject();
echo $result->translation;     // 翻译文本
echo $result->qualityScore;    // 9
print_r($result->notes);       // ["Agent 在此上下文中也可译为'代理']
print_r($result->glossaryUsed);// ["Agent→智能代理", "Toolbox→工具箱"]
```

### 多轮修正翻译

配合 Chat 组件实现带上下文的翻译修正：

```php
// 发起翻译对话
$chat->initiate(new MessageBag(
    Message::forSystem($systemPrompt),
));

// 第一轮：翻译
$chat->submit(Message::ofUser('翻译以下内容：...'));

// 第二轮：修正——AI 记得之前的翻译
$chat->submit(Message::ofUser('把 "智能代理" 改为 "Agent"，保留英文原文'));

// 第三轮：继续修正
$chat->submit(Message::ofUser('第二段的语句太长了，拆成两句'));
```

## 运行与验证

<!-- 运行示例并验证输出的步骤 -->

## 错误处理

### 通用错误处理模式

每个场景在生产环境中都应该包含完善的错误处理：

```php
use Symfony\AI\Platform\Exception\AuthenticationException;
use Symfony\AI\Platform\Exception\RateLimitExceededException;
use Symfony\AI\Platform\Exception\ContentFilterException;
use Symfony\AI\Platform\Exception\ExceedContextSizeException;

function safeInvoke(
    PlatformInterface $platform,
    MessageBag $messages,
    string $model,
    array $options = [],
): string {
    try {
        $response = $platform->invoke($model, $messages, $options);
        return $response->asText();
    } catch (AuthenticationException) {
        throw new \RuntimeException('AI 服务认证失败，请检查 API Key 配置');
    } catch (RateLimitExceededException $e) {
        // 可以实现退避重试
        sleep($e->getRetryAfter() ?? 5);
        return safeInvoke($platform, $messages, $model, $options);
    } catch (ExceedContextSizeException) {
        // 输入过长，截断消息历史
        $truncated = new MessageBag(...array_slice($messages->getMessages(), -5));
        return safeInvoke($platform, $truncated, $model, $options);
    } catch (ContentFilterException) {
        return '您的输入触发了安全过滤，请修改后重试。';
    } catch (\Throwable $e) {
        $this->logger->error('AI 调用异常', ['exception' => $e]);
        return '抱歉，AI 服务暂时不可用。';
    }
}
```

### Token 成本优化

| 优化策略 | 做法 | 效果 |
|---------|------|------|
| **精简系统提示** | 200 字而非 1000 字 | 减少 30-50% 输入 Token |
| **限制上下文** | 只保留最近 N 轮对话 | 避免上下文爆炸 |
| **选择合适模型** | 简单任务用 mini 模型 | 降低 80-90% 成本 |
| **缓存重复请求** | CachePlatform | 重复请求零成本 |
| **设置 max_tokens** | 限制输出长度 | 控制输出成本 |

## 生产环境注意事项

<!-- 部署到生产环境时的配置和优化建议 -->

## 扩展方向

<!-- 基于本场景的进一步扩展思路 -->

## 完整源代码

<!-- 完整可运行的源代码汇总 -->

## 下一步

下一章我们将学习 [第 14 章：实战 —— 实时流式对话](14-scenario-streaming.md)，使用 SSE 构建 ChatGPT 式的实时流式对话 Web 应用。
