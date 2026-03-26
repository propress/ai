# 第 19 章：实战 —— 带记忆的个性化助手

## 学习目标

- 理解 Memory 系统的架构与 MemoryInputProcessor 的注入机制
- 掌握 StaticMemoryProvider（用户画像）和 EmbeddingProvider（动态记忆）的使用
- 学会创建自定义 MemoryProvider 从数据库加载用户偏好
- 了解记忆的禁用机制与生产环境最佳实践

## 前置知识

- 已了解 Agent 工具调用循环与 InputProcessor 机制（参见 [第 15 章](15-scenario-tool-agent.md)）
- 了解向量存储基础（用于动态记忆，参见 [第 16 章](16-scenario-rag.md)）
- 熟悉 Symfony 安全组件基础（用于自定义 MemoryProvider）

## 业务场景描述

构建一个能记住用户偏好的个性化 AI 助手。助手通过长期记忆了解用户的技术栈、工作习惯、项目背景，提供更精准的建议。

**典型应用**：个人编程助手、学习辅导、健康顾问、私人教练。

## 架构概述

```php
Memory 系统的完整架构
═══════════════════

  ┌──────────────────────────────────────────────────────────┐
  │  Agent::call($messages)                                    │
  │                                                            │
  │  InputProcessor 管线：                                       │
  │  ┌─ SystemPromptInputProcessor ─── 注入系统提示             │
  │  ├─ MemoryInputProcessor ──────── 注入记忆（本场景核心）     │
  │  └─ AgentProcessor ───────────── 注入工具定义               │
  └──────────────────────────────────────────────────────────┘

  MemoryInputProcessor 的内部行为：
  ════════════════════════════════

  1. 收集所有 MemoryProvider 的记忆内容
  2. 构建 "# Conversation Memory" 文本块
  3. 追加到 System Message 末尾

  System Message（原始）:              System Message（注入后）:
  ──────────────────────              ──────────────────────────
  "你是编程助手"                       "你是编程助手
                                       
                                       # Conversation Memory
                                       - 用户是 PHP 开发者
                                       - 使用 Symfony 框架
                                       - 偏好 PostgreSQL
                                       - 项目使用 PHP 8.4
                                       - 喜欢简洁代码风格"

  MemoryProvider 类型层次
  ═══════════════════════

  MemoryProviderInterface
  ├── StaticMemoryProvider ──── 固定记忆列表（用户画像）
  ├── EmbeddingProvider ─────── 向量检索记忆（动态、海量）
  └── 自定义实现 ──────────────── 从数据库/缓存/API 加载
```

## 环境准备

```bash
composer require symfony/ai-platform symfony/ai-open-ai-platform \
    symfony/ai-agent
# 动态记忆还需要：
# symfony/ai-store symfony/ai-postgres-store
```

## 核心实现

### 静态记忆——用户画像

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Memory\StaticMemoryProvider;

// 创建用户画像记忆
$memoryProvider = new StaticMemoryProvider([
    '用户是一名 PHP 开发者，使用 Symfony 框架',
    '用户偏好使用 PostgreSQL 数据库',
    '用户的项目使用 PHP 8.4',
    '用户喜欢简洁的代码风格，不喜欢过多注释',
    '用户所在时区为 Asia/Shanghai',
    '用户的团队使用 PHPUnit 11 做单元测试',
    '用户的项目遵循 PSR-12 编码规范',
]);

// 创建带记忆的 Agent——MemoryInputProcessor 是一个 InputProcessor
$memoryProcessor = new MemoryInputProcessor([$memoryProvider]);
$agent = new Agent(
    $platform,
    $model,
    inputProcessors: [$memoryProcessor],
);

$response = $agent->call(new MessageBag(
    Message::ofUser('帮我设计一个缓存服务类'),
));

// AI 基于记忆生成 PHP 8.4 + Symfony + PostgreSQL 风格的代码
// 代码会很简洁（因为用户不喜欢过多注释）
// 使用 PSR-12 规范
echo $response->getContent();
```

### 动态记忆——向量化记忆

对于大量记忆内容（用户历史对话、项目笔记、学习记录），使用向量存储进行语义检索：

```php
use Symfony\AI\Agent\Memory\EmbeddingProvider;

// 使用向量存储作为记忆后端
$memoryProvider = new EmbeddingProvider(
    $store,       // 向量存储（PostgreSQL / Redis / ChromaDB）
    $vectorizer,  // Embedding 模型
);

// 添加记忆
$memoryProvider->add('用户最近在开发电商项目的购物车功能');
$memoryProvider->add('用户遇到了 Doctrine N+1 查询性能问题');
$memoryProvider->add('用户计划下周迁移到 PHP 8.4');
$memoryProvider->add('用户的项目使用 Redis 做缓存层');

// 当用户问 "怎么优化数据库查询？" 时：
// EmbeddingProvider 自动检索最相关的记忆：
//   → "用户遇到了 Doctrine N+1 查询性能问题"
//   → "用户的项目使用 Redis 做缓存层"
// 然后注入到 System Message 中

$memoryProcessor = new MemoryInputProcessor([$memoryProvider]);
$agent = new Agent($platform, $model, inputProcessors: [$memoryProcessor]);
$response = $agent->call(new MessageBag(
    Message::ofUser('怎么优化数据库查询？'),
));

// AI 的回答会针对性地解决 Doctrine N+1 问题
// 并建议使用 Redis 缓存查询结果
echo $response->getContent();
```

### 禁用记忆

某些场景下需要临时禁用记忆注入：

```php
// 使用 use_memory: false 选项禁用记忆
$response = $agent->call($messages, [
    'use_memory' => false,
]);
```

### 自定义记忆提供器

```php
use Symfony\AI\Agent\Memory\Memory;
use Symfony\AI\Agent\Memory\MemoryProviderInterface;
use Symfony\AI\Agent\Input;

class DatabaseMemoryProvider implements MemoryProviderInterface
{
    public function __construct(
        private readonly UserPreferenceRepository $repo,
        private readonly Security $security,
    ) {}

    /**
     * @return Memory[]
     */
    public function load(Input $input): array
    {
        $user = $this->security->getUser();
        if (null === $user) {
            return [];
        }

        $preferences = $this->repo->findByUser($user);

        return array_map(
            fn (UserPreference $p) => new Memory($p->getDescription()),
            $preferences,
        );
    }
}
```

## 运行与验证

1. **静态记忆验证**：创建带用户画像的 Agent，提问通用问题，确认回答反映了用户偏好（如技术栈、代码风格）
2. **动态记忆验证**：添加多条记忆，提问相关问题，确认 Agent 检索到了最相关的记忆
3. **记忆禁用验证**：使用 `use_memory: false` 调用，确认回答不再包含记忆上下文
4. **无记忆降级**：测试当 MemoryProvider 返回空列表时，Agent 仍能正常工作

## 错误处理

- **MemoryProvider 异常**：当数据库或向量存储不可用时，记忆加载应优雅降级，不影响 Agent 核心功能
- **记忆内容过多**：设置注入记忆的数量上限，防止 System Message 超出 Token 限制
- **用户未认证**：自定义 MemoryProvider 中处理未登录用户的情况，返回空记忆列表

## 生产环境注意事项

- **隐私保护**：用户记忆属于敏感数据，需加密存储并严格控制访问权限
- **记忆过期**：为记忆设置 TTL（生存时间），定期清理过时的记忆条目
- **多租户隔离**：确保每个用户只能访问自己的记忆，防止跨用户信息泄露
- **Token 预算**：监控记忆注入占用的 Token 数量，为主要对话内容留出足够空间
- **记忆质量**：定期审查自动收集的记忆内容，过滤低质量或矛盾的记忆

## 扩展方向

- 实现自动记忆提取：从对话中自动识别并存储用户偏好
- 添加记忆管理界面：让用户查看、编辑和删除已存储的记忆
- 支持记忆优先级：重要记忆优先注入，次要记忆在 Token 不足时省略
- 实现跨会话记忆：将短期对话记忆持久化为长期记忆

## 完整源代码

完整的带记忆个性化助手代码请参考 `examples/` 目录中的 Memory 集成示例。核心代码已在上述各节中完整展示。

## 下一步

恭喜你完成了所有进阶实战场景！你已经掌握了工具调用、RAG 检索、多智能体协作、联网搜索和记忆系统。接下来可以将这些技术组合起来，构建更复杂的 AI 应用。
