# Security 目录分析报告

## 目录职责

`Security/` 目录是 Symfony AI AiBundle 模块的安全子系统，负责为 AI 工具调用提供声明式的访问控制能力。该子系统将 Symfony Security 组件的授权机制（角色检查、Voter 投票、表达式求值）引入 AI Agent 的工具调用流程，通过 PHP 8 属性和事件监听器的组合实现了基于 AOP（面向切面编程）的安全拦截。

**目录路径**: `src/ai-bundle/src/Security/`

---

## 目录结构

```
Security/
├── Attribute/                              ← 声明层：安全元数据
│   └── IsGrantedTool.php                   ← 工具权限属性定义
│
└── EventListener/                          ← 执行层：运行时安全检查
    └── IsGrantedToolAttributeListener.php  ← 属性解释器 + 授权执行器
```

---

## 包含的文件清单

| 文件 | 类型 | 说明 |
|------|------|------|
| `Attribute/IsGrantedTool.php` | PHP 8 属性类（final） | 声明 AI 工具的访问权限要求，支持角色、Voter 属性、表达式等 |
| `EventListener/IsGrantedToolAttributeListener.php` | 事件监听器类 | 监听工具调用事件，读取属性并执行授权检查 |

---

## 架构概览

### 整体架构图

```
                        Symfony Security 组件
                ┌──────────────────────────────────┐
                │  AuthorizationCheckerInterface    │
                │  AccessDecisionManager            │
                │  Voter 系统                       │
                │  ExpressionLanguage               │
                └──────────┬───────────────────────┘
                           │
                    ┌──────┴───────┐
                    │  AiBundle    │
                    │  Security/   │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ↓                         ↓
       Attribute/                EventListener/
    ┌──────────────┐         ┌───────────────────────────┐
    │ IsGrantedTool │ ←─读取─ │ IsGrantedToolAttribute    │
    │ (元数据定义)   │         │ Listener (运行时执行)      │
    └──────────────┘         └───────────┬───────────────┘
                                         │
                                    ┌────┴────┐
                                    ↓         ↓
                              Agent Toolbox   Symfony Security
                              (事件源)         (授权引擎)
```

### 双层架构设计

| 层面 | 目录 | 核心类 | 职责 | 运行时机 |
|------|------|--------|------|----------|
| **声明层** | `Attribute/` | `IsGrantedTool` | 定义"谁需要什么权限" | 编码时 |
| **执行层** | `EventListener/` | `IsGrantedToolAttributeListener` | 执行"检查是否有权限" | 运行时 |

**双层分离的意义**：
- 属性类不依赖 Symfony Security（可在未安装安全组件时编译）
- 监听器负责所有运行时逻辑（可被替换、装饰、测试）
- 安全规则与工具业务代码完全解耦

---

## 核心设计模式

### 1. 面向切面编程（AOP）

安全检查作为横切关注点，通过属性声明 + 事件拦截实现织入：

```
                    横切关注点（安全）
                         │
            ┌────────────┼────────────┐
            ↓            ↓            ↓
     Tool A 方法    Tool B 方法   Tool C 方法
    #[IsGranted]   #[IsGranted]  (无属性)
         ↓              ↓            ↓
    [安全检查]      [安全检查]    [直接执行]
         ↓              ↓            ↓
       执行            执行          执行
```

**与传统 AOP 的对比**：

| 方面 | 传统 AOP（如 AspectJ） | AiBundle 实现 |
|------|----------------------|---------------|
| 切面定义 | 独立切面类 | `IsGrantedTool` 属性 |
| 织入方式 | 编译时/加载时 | 运行时（事件） |
| 连接点 | 方法调用、字段访问等 | `ToolCallArgumentsResolved` 事件 |
| 增强类型 | Before/After/Around | Before（前置拦截） |
| 基础设施 | AOP 框架 | PHP 反射 + Symfony EventDispatcher |

### 2. 属性驱动配置（Attribute-Driven Configuration）

```
传统方式：                              属性驱动方式：
┌─────────────────────┐               ┌─────────────────────────────┐
│ security.yaml       │               │ MyTool.php                   │
│   access_control:   │               │                              │
│     - tool: MyTool  │               │ #[IsGrantedTool('ROLE_USER')]│
│       role: ADMIN   │               │ class MyTool { ... }         │
└─────────────────────┘               └─────────────────────────────┘
  配置与代码分离                          配置与代码同处
  需要维护映射关系                        声明即配置
```

### 3. 事件监听器模式（Event Listener Pattern）

```
Agent Toolbox ──→ ToolCallArgumentsResolved 事件
                           ↓
              Symfony EventDispatcher
                           ↓
              IsGrantedToolAttributeListener
                     ├── 授权通过 → 事件继续
                     └── 授权失败 → AccessDeniedException
```

### 4. 策略模式（Strategy Pattern）

Subject 解析支持三种策略，根据类型自动分派：

```
Subject 引用                     解析策略
─────────────────────────────────────────
\Closure                    → 调用闭包 fn($args, $tool)
Expression                  → ExpressionLanguage 求值
string                      → 参数数组键值查找
array                       → 逐元素递归解析
```

---

## 完整调用流程

### 从 AI 对话到安全检查

```
1. 用户向 AI Agent 发送消息
    ↓
2. AI 模型（如 GPT-4 / Claude）判断需要调用工具
    ↓
3. 模型返回工具调用指令（function_call）
    ↓
4. Agent Toolbox 接收指令
    ├── 查找对应的工具类和方法
    ├── 反序列化 JSON 参数为 PHP 类型
    └── 创建 ToolCallArgumentsResolved 事件
    ↓
5. EventDispatcher 分发事件
    ↓
6. IsGrantedToolAttributeListener 拦截事件
    ├── 通过反射获取 ReflectionClass 和 ReflectionMethod
    ├── 读取类级别和方法级别的 #[IsGrantedTool] 属性
    ├── (无属性 → 快速返回)
    ├── 遍历每个属性：
    │   ├── 解析 subject（Closure / Expression / 字符串 / 数组）
    │   ├── 调用 AuthorizationChecker::isGranted($attribute, $subject)
    │   │       ↓
    │   │   Symfony Security 内部：
    │   │   ├── AccessDecisionManager 收集所有 Voter
    │   │   ├── 每个 Voter 投票（GRANT / DENY / ABSTAIN）
    │   │   └── 根据策略汇总结果
    │   │       ↓
    │   ├── 授权通过 → 检查下一个属性
    │   └── 授权失败 → 抛出 AccessDeniedException
    ↓
7. 所有属性检查通过
    ↓
8. 工具方法被执行，返回结果给 Agent
    ↓
9. Agent 将结果发送回 AI 模型
```

### 授权失败时的流程

```
IsGrantedToolAttributeListener
    ↓ 授权失败
AccessDeniedException
    ├── message: 自定义消息 / AccessDecision 消息 / 'Access Denied.'
    ├── code: 自定义代码 / 403
    ├── attributes: ['ROLE_ADMIN']
    ├── subject: (解析后的主体)
    └── accessDecision: (Symfony 7.3+)
    ↓
异常向上传播
    ↓
Agent 捕获并处理（通常返回错误消息给模型）
```

---

## 与 Symfony Security 组件的集成

### 集成架构

```
AiBundle Security                    Symfony Security
─────────────────                    ────────────────
IsGrantedTool ──声明──→
                       IsGrantedToolAttributeListener
                              │
                              │ isGranted()
                              ↓
                       AuthorizationCheckerInterface
                              │
                              ↓
                       AccessDecisionManager
                              │
                    ┌─────────┼─────────┐
                    ↓         ↓         ↓
                RoleVoter  CustomVoter  ExpressionVoter
                    │         │         │
                    └─────────┼─────────┘
                              ↓
                        投票决策结果
```

### 可复用的 Symfony Security 能力

| 能力 | Symfony Security 组件 | AiBundle 集成方式 |
|------|----------------------|-------------------|
| 角色检查 | `RoleVoter` | `#[IsGrantedTool('ROLE_ADMIN')]` |
| 自定义投票 | 自定义 `Voter` | `#[IsGrantedTool('CUSTOM_PERM', 'subject')]` |
| 表达式安全 | `ExpressionVoter` | `#[IsGrantedTool(new Expression('...'))]` |
| 投票策略 | `AccessDecisionManager` | 透明复用 |
| 认证令牌 | `TokenStorageInterface` | 由 `AuthorizationChecker` 内部访问 |

---

## 服务注册与降级策略

### 正常路径（已安装 symfony/security-core）

```php
// config/services.php
->set('ai.security.is_granted_attribute_listener', IsGrantedToolAttributeListener::class)
    ->args([
        service('security.authorization_checker'),
        service('expression_language')->nullOnInvalid(),
    ])
    ->tag('kernel.event_listener')
```

### 降级路径（未安装 symfony/security-core）

```php
// AiBundle.php
if (!ContainerBuilder::willBeAvailable('symfony/security-core', AuthorizationCheckerInterface::class, ['symfony/ai-bundle'])) {
    // 1. 移除监听器服务
    $builder->removeDefinition('ai.security.is_granted_attribute_listener');

    // 2. 注册编译期错误提示
    $builder->registerAttributeForAutoconfiguration(
        IsGrantedTool::class,
        static fn () => throw new InvalidArgumentException(
            'Using #[IsGrantedTool] attribute requires additional dependencies. '
            . 'Try running "composer install symfony/security-core".'
        ),
    );
}
```

**降级效果**：

| 场景 | 行为 |
|------|------|
| 未安装 security-core，未使用属性 | 完全透明，无任何影响 |
| 未安装 security-core，使用了属性 | 容器编译时抛出 `InvalidArgumentException`，给出安装指引 |
| 已安装 security-core | 监听器正常工作，属性在运行时被检查 |

---

## 扩展点

### 1. 自定义 Voter

实现 Symfony 的 `VoterInterface` 或继承 `Voter` 抽象类，为 AI 工具定义自定义权限逻辑：

```php
class AiToolAccessVoter extends Voter
{
    protected function supports(string $attribute, mixed $subject): bool
    {
        return str_starts_with($attribute, 'AI_TOOL_');
    }

    protected function voteOnAttribute(string $attribute, mixed $subject, TokenInterface $token): bool
    {
        $user = $token->getUser();
        return $user->canUseTool($attribute, $subject);
    }
}
```

### 2. Expression 函数扩展

通过注入自定义 `ExpressionLanguage` 实例，添加领域特定的表达式函数：

```php
$el = new ExpressionLanguage();
$el->register('tool_tier', /* compiler */ ..., /* evaluator */ ...);
// 然后在属性中使用：#[IsGrantedTool('PREMIUM', new Expression('tool_tier(tool)'))]
```

### 3. Closure 主体构建

Closure 类型支持最大灵活性的主体解析，可以构建任意复杂的主体对象：

```php
#[IsGrantedTool('MANAGE', subject: fn($args, $tool) => new ToolAccessSubject(
    toolClass: get_class($tool),
    resourceId: $args['id'],
    operation: 'manage',
))]
```

### 4. 新增安全属性和监听器

目录结构天然支持添加新的安全属性类型和对应的监听器。例如：

```
Security/
├── Attribute/
│   ├── IsGrantedTool.php        ← 现有：访问控制
│   ├── RateLimitTool.php        ← 新增：速率限制
│   └── AuditTool.php            ← 新增：审计标记
│
└── EventListener/
    ├── IsGrantedToolAttributeListener.php  ← 现有
    ├── RateLimitToolListener.php           ← 新增
    └── AuditToolListener.php              ← 新增
```

### 5. 监听器装饰

通过 Symfony 的服务装饰器模式，在不修改原始监听器的情况下添加日志、监控等功能：

```php
class LoggingSecurityListener
{
    public function __construct(
        private IsGrantedToolAttributeListener $inner,
        private LoggerInterface $logger,
    ) {}

    public function __invoke(ToolCallArgumentsResolved $event): void
    {
        $this->logger->info('Security check for tool: '.$event->getMetadata()->getName());
        try {
            ($this->inner)($event);
            $this->logger->info('Security check passed');
        } catch (AccessDeniedException $e) {
            $this->logger->warning('Security check failed: '.$e->getMessage());
            throw $e;
        }
    }
}
```

---

## 外部知识

### Symfony Security Voter 系统

Voter 是 Symfony 的细粒度授权机制。每个 Voter 实现 `vote(TokenInterface $token, mixed $subject, array $attributes): int` 方法，返回：

| 投票结果 | 常量 | 说明 |
|----------|------|------|
| 授予 | `VoterInterface::ACCESS_GRANTED` | 认为应授权 |
| 拒绝 | `VoterInterface::ACCESS_DENIED` | 认为应拒绝 |
| 弃权 | `VoterInterface::ACCESS_ABSTAIN` | 不处理此类权限 |

`AccessDecisionManager` 根据配置的策略（affirmative/consensus/unanimous/priority）汇总所有 Voter 的投票结果。

### Symfony ExpressionLanguage

ExpressionLanguage 组件提供安全的表达式求值能力，允许在配置中嵌入简单的逻辑。在安全上下文中，表达式可以访问：

- `tool`：当前工具实例
- `args`：已解析的参数数组

### PHP 8 属性

PHP 8 原生属性通过 `#[...]` 语法声明，通过反射 API 读取。关键方法：

- `ReflectionClass::getAttributes(className)`：获取类上的属性
- `ReflectionMethod::getAttributes(className)`：获取方法上的属性
- `ReflectionAttribute::newInstance()`：实例化属性对象

---

## 与 AiBundle 其他子系统的关系

```
AiBundle
├── Command/           ← 使用 InvalidArgumentException（共享异常体系）
├── DependencyInjection/ ← 容器编译、配置加载
├── Exception/         ← 提供 InvalidArgumentException 用于降级提示
├── Profiler/          ← 开发时调试工具
├── Security/          ← 本目录：安全子系统
│   ├── Attribute/     ← 声明 + 元数据
│   └── EventListener/ ← 运行时执行
├── config/            ← services.php 中注册安全服务
└── AiBundle.php       ← 降级处理，条件性移除安全服务
```

**关键依赖关系**：

| 方向 | 依赖方 | 被依赖方 | 说明 |
|------|--------|----------|------|
| → | Security/EventListener | Security/Attribute | 监听器读取属性 |
| → | Security/EventListener | Agent Toolbox | 监听 `ToolCallArgumentsResolved` 事件 |
| → | Security/EventListener | Symfony Security | 调用 `AuthorizationCheckerInterface` |
| → | AiBundle.php | Security/ | 条件注册/移除安全服务 |
| → | config/services.php | Security/EventListener | 服务定义 |

---

## 测试覆盖

### 测试文件

`src/ai-bundle/tests/Security/IsGrantedToolAttributeListenerTest.php`

### 测试策略

| 测试维度 | 覆盖情况 |
|----------|----------|
| 方法级属性 — 授权通过 | ✅ |
| 方法级属性 — 授权失败 | ✅ |
| 类级属性 — 授权失败 | ✅ |
| 字符串 subject 解析 | ✅ |
| Expression subject 解析 | ✅ |
| 自定义拒绝消息 | ✅ |
| 多种工具类型（DataProvider） | ✅ |

### 测试夹具

- `ToolWithIsGrantedOnMethod`：包含 `simple()`、`argumentAsSubject()`、`expressionAsSubject()` 三个方法，分别测试纯角色检查、字符串 subject、Expression subject
- `ToolWithIsGrantedOnClass`：类级别 `#[IsGrantedTool]`，使用 Expression subject 和自定义消息

---

## 总结

AiBundle 的 `Security/` 目录用两个类（一个属性 + 一个监听器）构建了一个完整的 AI 工具安全子系统。其设计的核心价值在于：

1. **声明式安全** — `#[IsGrantedTool]` 属性让安全规则与工具代码同处一地，一目了然
2. **Symfony 生态集成** — 完整复用 Symfony Security 的 Voter、AuthorizationChecker、ExpressionLanguage，无需重新发明
3. **事件驱动的 AOP** — 安全作为横切关注点，通过事件拦截透明织入工具调用流程
4. **灵活的主体解析** — 字符串 / Expression / Closure / 数组四种策略，覆盖从简单到复杂的所有场景
5. **优雅降级** — 在 `symfony/security-core` 未安装时提供清晰的编译期错误，而非运行时崩溃
6. **面向扩展** — 目录结构、设计模式、非 final 监听器类，为未来添加更多安全机制预留了空间
7. **零侵入** — 工具开发者只需添加属性标注，无需修改业务代码或了解安全检查的内部机制
