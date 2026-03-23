# 带长期记忆的个性化助手

## 业务场景

你在做一个健身教练 AI。每个用户有不同的身体情况、健身目标和偏好。AI 需要记住这些信息，即使隔了几天用户再来对话，也能基于之前了解的信息给出个性化建议。

**典型应用：** 个性化推荐助手、健康顾问、学习辅导、个人秘书

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台 |
| **Agent** | 智能体框架 |
| **Agent Memory** | 记忆系统，将历史信息注入到对话上下文中 |
| **Store** | 向量存储，用于基于语义的记忆检索 |

## 记忆的工作原理

```
用户提问 → MemoryInputProcessor 检索相关记忆 → 把记忆作为上下文注入消息 → AI 基于上下文回答

记忆来源可以是：
1. 静态记忆（StaticMemoryProvider）：预设的固定事实
2. 嵌入记忆（EmbeddingProvider）：基于向量相似度从历史对话中检索
```

---

## Step 1：静态记忆 —— 最简单的个性化

如果用户的关键信息是已知的（从数据库加载），可以用静态记忆直接注入。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Memory\MemoryInputProcessor;
use Symfony\AI\Agent\Memory\StaticMemoryProvider;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpClient\HttpClient;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());

// 1. 系统提示
$systemPrompt = new SystemPromptInputProcessor(
    '你是一个专业的健身教练 AI。根据用户的个人情况给出定制化的建议。'
    . '回答风格：专业但友好，给出具体可执行的建议。'
);

// 2. 从数据库加载的用户画像作为静态记忆
// 在真实场景中，这些数据来自用户注册信息或之前的对话总结
$userProfile = new StaticMemoryProvider(
    '用户姓名：小王',
    '年龄：28岁，男性',
    '身高：175cm，体重：80kg',
    '健身目标：减脂增肌',
    '健身经验：初学者，刚开始健身两周',
    '可用器材：家里有一对哑铃和瑜伽垫',
    '饮食偏好：不吃辣，对海鲜过敏',
    '每周可用训练时间：工作日每天 30 分钟，周末 1 小时',
);

// 3. 创建记忆处理器
$memoryProcessor = new MemoryInputProcessor([$userProfile]);

// 4. 创建带记忆的 Agent
$agent = new Agent(
    $platform,
    'gpt-4o-mini',
    [$systemPrompt, $memoryProcessor], // 输入处理器：注入系统提示和记忆
);

// 5. 用户对话 —— AI 自动使用记忆信息
$questions = [
    '今天应该练什么？',
    '训练完吃什么比较好？',
    '我觉得哑铃太轻了，有没有什么动作可以加大难度？',
];

foreach ($questions as $question) {
    echo "小王：{$question}\n";

    $result = $agent->call(new MessageBag(Message::ofUser($question)));
    echo "教练：" . $result->getContent() . "\n\n";
}
```

**效果：** AI 回答时会自动考虑用户是初学者、只有哑铃和瑜伽垫、对海鲜过敏等信息，给出完全定制化的建议。

---

## Step 2：基于 Embedding 的长期记忆

静态记忆适合固定信息。但真实场景中，用户在对话中会透露很多信息，这些信息也需要被记住。

`EmbeddingProvider` 将历史对话向量化存入向量数据库，之后每次新对话都会自动检索语义相关的历史内容。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Memory\EmbeddingProvider;
use Symfony\AI\Agent\Memory\MemoryInputProcessor;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Store\Bridge\InMemory\Store;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\DocumentIndexer;
use Symfony\AI\Store\Indexer\DocumentProcessor;
use Symfony\Component\HttpClient\HttpClient;
use Symfony\Component\Uid\Uuid;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());

// ===== 1. 建立记忆存储 =====

$store = new Store();
$vectorizer = new Vectorizer($platform, $embeddingModel = 'text-embedding-3-small');

// 模拟从数据库加载的过往对话记录（现实中这些来自 Chat 组件保存的历史）
$pastConversations = [
    ['role' => 'user', 'time' => '2025-01-10', 'content' => '我最近膝盖有点不舒服，深蹲的时候疼'],
    ['role' => 'assistant', 'time' => '2025-01-10', 'content' => '建议暂停深蹲，改做箱式深蹲或腿举机，注意膝盖不要超过脚尖'],
    ['role' => 'user', 'time' => '2025-01-12', 'content' => '我周三和周五晚上有空，想固定这两天训练'],
    ['role' => 'assistant', 'time' => '2025-01-12', 'content' => '好的，周三安排上肢训练，周五安排下肢训练'],
    ['role' => 'user', 'time' => '2025-01-14', 'content' => '我买了一条弹力带，可以加入训练吗？'],
    ['role' => 'assistant', 'time' => '2025-01-14', 'content' => '当然可以，弹力带适合热身和辅助训练'],
];

// 将历史对话索引到向量存储
$documents = [];
foreach ($pastConversations as $msg) {
    $documents[] = new TextDocument(
        id: Uuid::v4(),
        content: "时间：{$msg['time']}，角色：{$msg['role']}，内容：{$msg['content']}",
        metadata: new Metadata($msg),
    );
}

$indexer = new DocumentIndexer(new DocumentProcessor($vectorizer, $store));
$indexer->index($documents);

echo "已加载 " . count($documents) . " 条历史记录到记忆系统\n\n";

// ===== 2. 创建带嵌入记忆的 Agent =====

$embeddingsModel = $platform->getModelCatalog()->getModel($embeddingModel);
$embeddingMemory = new EmbeddingProvider($platform, $embeddingsModel, $store);
$memoryProcessor = new MemoryInputProcessor([$embeddingMemory]);

$systemPrompt = new SystemPromptInputProcessor(
    '你是一个专业的健身教练。根据用户的历史对话和当前问题给出建议。'
    . '参考记忆中的信息来个性化你的回答。'
);

$agent = new Agent($platform, 'gpt-4o-mini', [$systemPrompt, $memoryProcessor]);

// ===== 3. 新对话 —— AI 自动检索相关记忆 =====

$questions = [
    '今天是周五，帮我安排一下今晚的训练吧',
    // → AI 会检索到"周五安排下肢训练"以及"膝盖不舒服"的记忆

    '深蹲可以做了吗？',
    // → AI 会检索到膝盖问题的记忆

    '弹力带可以怎么用在热身里？',
    // → AI 会检索到买了弹力带的记忆
];

foreach ($questions as $question) {
    echo "用户：{$question}\n";

    $result = $agent->call(new MessageBag(Message::ofUser($question)));
    echo "教练：" . $result->getContent() . "\n\n";
}
```

**效果：** 用户问"今天是周五"时，AI 会想起之前约定了周五练下肢。用户问"深蹲可以做了吗"时，AI 会想起膝盖的问题。

---

## Step 3：组合静态记忆和嵌入记忆

在真实应用中，通常同时使用两种记忆：

```php
// 静态记忆：用户档案（确定不变的信息）
$profileMemory = new StaticMemoryProvider(
    '姓名：小王，28岁男性',
    '身高175cm，体重80kg',
    '健身目标：减脂增肌',
);

// 嵌入记忆：历史对话（动态积累的信息）
$conversationMemory = new EmbeddingProvider($platform, $embeddingsModel, $store);

// 组合两种记忆
$memoryProcessor = new MemoryInputProcessor([
    $profileMemory,        // 固定的用户画像
    $conversationMemory,   // 基于语义检索的历史记忆
]);

$agent = new Agent($platform, 'gpt-4o-mini', [$systemPrompt, $memoryProcessor]);
```

---

## 完整示例：个人学习辅导助手

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Memory\MemoryInputProcessor;
use Symfony\AI\Agent\Memory\StaticMemoryProvider;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpClient\HttpClient;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());

// 学生档案
$studentProfile = new StaticMemoryProvider(
    '学生姓名：小李，高二学生',
    '目标：准备高考，目标大学是浙江大学计算机系',
    '强势科目：数学和物理',
    '弱势科目：英语（尤其是阅读理解和写作）',
    '每天可学习时间：放学后 3 小时',
    '学习风格偏好：喜欢通过做题来学习，不喜欢纯看书',
    '上次模考成绩：数学 135，物理 92，英语 105（满分 150）',
);

$systemPrompt = new SystemPromptInputProcessor(
    '你是一个高考备考辅导老师。根据学生的情况制定学习建议。'
    . '特别关注学生的弱势科目。建议要具体、可执行，考虑到学生的时间安排。'
);

$memoryProcessor = new MemoryInputProcessor([$studentProfile]);
$agent = new Agent($platform, 'gpt-4o-mini', [$systemPrompt, $memoryProcessor]);

// 模拟跨天的对话
echo "=== 周一 ===\n\n";

$result = $agent->call(new MessageBag(
    Message::ofUser('老师，这周的学习计划怎么安排？'),
));
echo "老师：" . $result->getContent() . "\n\n";

echo "=== 周三 ===\n\n";

$result = $agent->call(new MessageBag(
    Message::ofUser('我觉得英语阅读还是看不懂长句子，有什么办法？'),
));
echo "老师：" . $result->getContent() . "\n\n";

echo "=== 周五 ===\n\n";

$result = $agent->call(new MessageBag(
    Message::ofUser('这次物理小测考了 88 分，感觉退步了怎么办？'),
));
echo "老师：" . $result->getContent() . "\n\n";
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `StaticMemoryProvider` | 静态记忆，存放固定的用户信息/事实 |
| `EmbeddingProvider` | 嵌入记忆，基于语义相似度从历史中检索相关信息 |
| `MemoryInputProcessor` | 记忆处理器，将记忆注入到 AI 的上下文中 |
| 多记忆源组合 | 可以同时使用多个 MemoryProvider |
| 向量存储 | 嵌入记忆需要向量数据库来存储和检索 |

## 下一步

如果你的 AI 需要上网搜索实时信息并整理成报告，请看 [09-web-research-assistant.md](./09-web-research-assistant.md)。
