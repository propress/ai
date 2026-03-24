# 多轮对话与会话持久化

## 业务场景

你正在开发一个在线客服系统。用户通过网页与 AI 客服对话，关闭页面后再打开，仍然能看到之前的对话并继续聊天。不同用户之间的对话互相隔离。

**典型应用：** 在线客服系统、咨询助手、教育辅导对话

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台，发送消息获取回复 |
| **Agent** | 封装调用逻辑，作为 Chat 的后端引擎 |
| **Chat** | 管理对话状态，自动持久化消息历史 |

## 前置准备

```bash
composer require symfony/ai-platform symfony/ai-platform-openai
composer require symfony/ai-agent
composer require symfony/ai-chat

# 根据你选择的存储后端安装对应的 Bridge
composer require symfony/ai-chat-in-memory     # 内存存储（开发/测试）
# 或以下任一：
# composer require symfony/ai-chat-symfony-cache  # Symfony Cache
# composer require symfony/ai-chat-redis          # Redis
# composer require symfony/ai-chat-doctrine-dbal  # 关系型数据库（MySQL/PostgreSQL 等）
# composer require symfony/ai-chat-mongodb        # MongoDB
```

---

## Step 1：理解 Chat 的架构

在 [01-basic-chatbot.md](./01-basic-chatbot.md) 中，我们手动维护对话历史数组。`Chat` 模块自动化了这个过程：

```
用户发消息 → Chat 从存储加载历史 → 组装完整消息列表 → 调用 Agent/Platform → 保存新消息到存储 → 返回回复
```

核心类的关系：
```
Chat（对话管理器）
 ├── Agent（AI 调用引擎）
 │    └── Platform（AI 平台连接）
 └── MessageStore（消息持久化）
      └── 具体实现：InMemory / Redis / DBAL / MongoDB...
```

---

## Step 2：使用内存存储的基础对话

先用内存存储来理解 Chat 的工作方式。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\InMemory\Store as InMemoryStore;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpClient\HttpClient;

// 1. 创建 Platform 和 Agent
$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());
$agent = new Agent($platform, 'gpt-4o-mini');

// 2. 创建 Chat（使用内存存储）
$chat = new Chat($agent, new InMemoryStore());

// 3. 初始化对话：设置系统提示
$chat->initiate(new MessageBag(
    Message::forSystem('你是一个在线英语老师。用简单的例句帮助用户学英语。每次只教一个知识点。'),
));

// 4. 第一轮对话
$response = $chat->submit(Message::ofUser('我想学习如何用英语自我介绍'));
echo "老师：" . $response->getContent() . "\n\n";

// 5. 第二轮对话 —— Chat 自动携带上一轮的上下文
$response = $chat->submit(Message::ofUser('能给我一个更正式的版本吗？'));
echo "老师：" . $response->getContent() . "\n\n";

// 6. 第三轮对话 —— AI 知道之前的所有内容
$response = $chat->submit(Message::ofUser('如果是在面试中自我介绍呢？'));
echo "老师：" . $response->getContent() . "\n\n";
```

**与手动方式的区别：** 你不需要手动管理消息数组。Chat 在每次 `submit()` 时自动：
1. 加载之前所有消息
2. 附加新的用户消息
3. 调用 Agent 获取回复
4. 保存用户消息和 AI 回复到存储
5. 返回 AI 回复

---

## Step 3：使用 Redis 实现跨请求持久化

在 Web 应用中，每个 HTTP 请求是独立的进程。我们需要将对话存入 Redis，这样跨请求也能保持对话上下文。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\Bridge\Redis\Store as RedisStore;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpClient\HttpClient;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());
$agent = new Agent($platform, 'gpt-4o-mini');

// 使用 Redis 存储
$redis = new \Redis();
$redis->connect('127.0.0.1', 6379);
$store = new RedisStore($redis);

$chat = new Chat($agent, $store);
```

### 模拟 Web 场景：跨请求的对话

```php
// ============================================
// 模拟第一个 HTTP 请求：用户开始新对话
// ============================================
// 通常 conversationId 由前端生成并存在 cookie/session 中
$conversationId = 'user-123-conv-456';

$chat->initiate(
    new MessageBag(
        Message::forSystem('你是一个旅行顾问。帮用户规划旅行。'),
    ),
    $conversationId, // 用对话 ID 隔离不同用户/会话
);

$response = $chat->submit(
    Message::ofUser('我想去日本玩 5 天，预算 1 万元'),
    $conversationId,
);
echo "顾问：" . $response->getContent() . "\n";
// 请求结束，对话已保存到 Redis

// ============================================
// 模拟第二个 HTTP 请求：用户回来继续对话
// ============================================
// 从 cookie/session 中取回 conversationId
$conversationId = 'user-123-conv-456';

// 不需要再次 initiate —— Chat 从 Redis 加载之前的对话
$response = $chat->submit(
    Message::ofUser('我对京都特别感兴趣，可以多安排一些时间吗？'),
    $conversationId,
);
echo "顾问：" . $response->getContent() . "\n";

// ============================================
// 模拟第三个 HTTP 请求
// ============================================
$response = $chat->submit(
    Message::ofUser('京都有什么特别推荐的美食吗？'),
    $conversationId,
);
echo "顾问：" . $response->getContent() . "\n";
```

---

## Step 4：使用 Doctrine DBAL 存入数据库

如果你的应用使用关系型数据库（MySQL/PostgreSQL），可以将对话存入数据库表。

```php
<?php

use Doctrine\DBAL\DriverManager;
use Symfony\AI\Chat\Bridge\DoctrineDbal\Store as DbalStore;

// 创建数据库连接
$connection = DriverManager::getConnection([
    'url' => 'mysql://user:password@127.0.0.1:3306/myapp',
]);

// 创建 Chat Store（会自动管理表结构）
$store = new DbalStore($connection);

// 后续使用方式与 Redis 完全相同
$chat = new Chat($agent, $store);
```

> **优势：** 对话数据可以配合其他业务数据做分析查询，比如"过去一个月用户最常问的问题是什么"。

---

## Step 5：使用 Symfony Session 存储

在 Symfony 框架中，可以直接利用 Session 组件存储对话。适合简单场景，无需额外数据库。

```php
<?php

use Symfony\AI\Chat\Bridge\SymfonySession\Store as SessionStore;
use Symfony\Component\HttpFoundation\Session\Session;

// 在 Symfony Controller 中
$session = $request->getSession();
$store = new SessionStore($session);

$chat = new Chat($agent, $store);

// 对话 ID 自动与 Session 关联，不同用户天然隔离
$response = $chat->submit(Message::ofUser('你好'));
```

---

## 完整示例：模拟一个在线教育辅导场景

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\InMemory\Store as InMemoryStore;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpClient\HttpClient;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());
$agent = new Agent($platform, 'gpt-4o-mini');
$chat = new Chat($agent, new InMemoryStore());

// 设置系统提示：定义 AI 的教学风格
$chat->initiate(new MessageBag(
    Message::forSystem(
        '你是一个耐心的数学辅导老师，面向初中学生。'
        . '教学原则：1) 先问学生哪里不懂；2) 用简单的例子解释；3) 每次只讲一个概念；'
        . '4) 讲完后给学生一道练习题检验理解；5) 如果学生做错了，鼓励并引导纠正。'
    ),
));

// 模拟完整的教学对话流程
$dialogue = [
    '老师你好，我不太懂一元二次方程怎么解',
    '因式分解是什么意思？',
    '哦，我懂了。那 x² + 5x + 6 = 0 怎么用因式分解？',
    '答案是 x = -2 和 x = -3 对吗？',
    '太好了！能再给我一道题练习吗？',
];

echo "=== 数学辅导对话 ===\n\n";

foreach ($dialogue as $question) {
    echo "学生：{$question}\n\n";

    $response = $chat->submit(Message::ofUser($question));
    echo "老师：" . $response->getContent() . "\n\n";
    echo str_repeat('-', 50) . "\n\n";
}
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `Chat` | 对话管理器，自动管理消息历史和持久化 |
| `chat->initiate()` | 初始化新对话（设置系统提示），可传入对话 ID |
| `chat->submit()` | 提交用户消息，自动带上下文，返回 AI 回复 |
| `InMemoryStore` | 内存存储，适合开发测试 |
| `RedisStore` | Redis 存储，适合生产环境需要快速读写 |
| `DbalStore` | 数据库存储，适合需要长期保存和查询分析 |
| `SessionStore` | Session 存储，适合 Symfony 简单场景 |
| 对话 ID | 隔离不同用户/会话的唯一标识 |

## 下一步

现在你的聊天机器人可以记住对话了。但如果用户问"今天天气怎么样"或"帮我查一下某个产品的库存"，AI 无法回答因为它没有工具。请看 [03-tool-augmented-assistant.md](./03-tool-augmented-assistant.md) 学习如何给 AI 添加工具能力。
