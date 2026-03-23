# EventListener 目录分析报告

## 目录职责

`EventListener/` 目录包含 Symfony AI AiBundle 安全子系统的运行时事件监听器。这些监听器负责在 AI 工具调用过程中拦截事件，读取声明式安全属性并执行实际的授权检查。它们是安全子系统的"执行引擎"，将静态的属性声明转化为动态的运行时行为。

**目录路径**: `src/ai-bundle/src/Security/EventListener/`

---

## 包含的文件清单

| 文件 | 类型 | 说明 |
|------|------|------|
| `IsGrantedToolAttributeListener.php` | 事件监听器类 | 监听 `ToolCallArgumentsResolved` 事件，执行 `#[IsGrantedTool]` 属性声明的授权检查 |

---

## 架构概览

### 事件流拓扑

```
AI Agent
    ↓ 模型输出工具调用指令
Toolbox（Agent 组件）
    ↓ 反序列化参数
ToolCallArgumentsResolved 事件
    ↓ EventDispatcher 分发
┌─────────────────────────────────────────┐
│ IsGrantedToolAttributeListener          │  ← EventListener/ 目录
│   ├── 反射读取 #[IsGrantedTool] 属性      │
│   ├── 解析 subject（字符串/表达式/闭包）    │
│   ├── 调用 AuthorizationChecker          │
│   └── 通过 → 继续 / 失败 → 抛出异常        │
└─────────────────────────────────────────┘
    ↓ (通过)
工具方法执行
```

### 核心设计原则

1. **事件驱动** — 通过 Symfony EventDispatcher 与工具执行流程解耦
2. **反射驱动** — 利用 PHP 反射 API 读取属性元数据
3. **委托授权** — 将实际授权判定委托给 Symfony Security 的 Voter 系统
4. **快速路径优化** — 无属性的工具调用不产生额外开销

---

## 设计模式详解

### 事件监听器模式（Event Listener Pattern）

本目录中的监听器遵循 Symfony 的事件监听器惯例：

```php
class IsGrantedToolAttributeListener
{
    public function __invoke(ToolCallArgumentsResolved $event): void
    {
        // 事件处理逻辑
    }
}
```

**关键特点**：
- 使用 `__invoke` 魔术方法，使类可被直接调用
- 参数类型声明 `ToolCallArgumentsResolved` 用于 Symfony 的自动事件绑定
- 通过 `kernel.event_listener` 标签注册到 EventDispatcher

### 拦截器模式（Interceptor Pattern）

监听器在工具执行前进行拦截，根据检查结果决定是否允许继续：

```
请求 → [拦截器] → 目标
         │
         ├── 条件满足 → 放行
         └── 条件不满足 → 阻止（抛出异常）
```

### 策略分派（Strategy Dispatch）

`getIsGrantedSubject()` 方法内部根据 subject 类型分派到不同的解析策略：

```
Subject 引用类型
    ├── Closure    → 策略 A：直接调用
    ├── Expression → 策略 B：ExpressionLanguage 求值
    └── string     → 策略 C：参数数组键值查找
```

---

## IsGrantedToolAttributeListener 概要

### 类签名

```php
class IsGrantedToolAttributeListener
{
    public function __construct(
        private readonly AuthorizationCheckerInterface $authChecker,
        private ?ExpressionLanguage $expressionLanguage = null,
    )
```

### 方法总览

| 方法 | 可见性 | 输入 | 输出 | 说明 |
|------|--------|------|------|------|
| `__construct` | public | `AuthorizationCheckerInterface`, `?ExpressionLanguage` | — | 注入授权检查器和可选的表达式引擎 |
| `__invoke` | public | `ToolCallArgumentsResolved` | `void` 或抛出异常 | 事件处理：读取属性、解析主体、执行授权检查 |
| `getIsGrantedSubject` | private | `string\|Expression\|\Closure`, `object`, `array` | `mixed` | 根据引用类型解析授权主体 |

### 依赖关系

```
IsGrantedToolAttributeListener
    ├── [必需] AuthorizationCheckerInterface ← Symfony Security
    ├── [可选] ExpressionLanguage            ← Symfony ExpressionLanguage
    ├── [读取] IsGrantedTool                 ← AiBundle Security Attribute
    ├── [监听] ToolCallArgumentsResolved     ← Agent Toolbox Event
    ├── [抛出] AccessDeniedException         ← Symfony Security Exception
    └── [兼容] AccessDecision                ← Symfony Security 7.3+ BC 层
```

---

## 服务注册

### DI 容器配置（services.php）

```php
->set('ai.security.is_granted_attribute_listener', IsGrantedToolAttributeListener::class)
    ->args([
        service('security.authorization_checker'),
        service('expression_language')->nullOnInvalid(),
    ])
    ->tag('kernel.event_listener')
```

### 注册要素

| 配置项 | 值 | 说明 |
|--------|-----|------|
| 服务 ID | `ai.security.is_granted_attribute_listener` | 容器中的唯一标识 |
| 参数 1 | `security.authorization_checker` | Symfony Security 的授权检查器 |
| 参数 2 | `expression_language`（可为 null） | `nullOnInvalid()` 确保服务不存在时注入 null |
| 标签 | `kernel.event_listener` | 自动注册到 EventDispatcher |

### 优雅降级机制

当 `symfony/security-core` 未安装时，`AiBundle.php` 会：

1. **移除服务定义** — `$builder->removeDefinition('ai.security.is_granted_attribute_listener')`
2. **注册编译期错误** — 任何使用 `#[IsGrantedTool]` 属性的服务在容器编译时抛出 `InvalidArgumentException`

这确保了：
- 不安装安全组件时，不会出现运行时的 ClassNotFound 错误
- 误用属性时，会在应用启动阶段（而非运行时）得到明确的错误提示

---

## 在 AiBundle 安全子系统中的定位

```
Security/
├── Attribute/                          ← 声明层
│   └── IsGrantedTool.php               ← "定义安全规则"
│
└── EventListener/                      ← 执行层（本目录）
    └── IsGrantedToolAttributeListener.php  ← "执行安全规则"
```

**声明与执行的分离**：

| 层面 | 职责 | 时机 | 依赖 |
|------|------|------|------|
| Attribute/ | 定义权限要求（元数据） | 编码时 | 无运行时依赖 |
| EventListener/ | 读取并执行权限检查 | 运行时 | Symfony Security, 反射 API |

这种分离带来的好处：
- 属性类可以在任何环境中使用（无安全组件也可编译）
- 监听器可以被替换或扩展，而不影响属性定义
- 测试时可以单独测试属性解析和授权逻辑

---

## 与 Symfony 事件监听器体系的对比

| 特征 | AiBundle 安全监听器 | Symfony Security 监听器 |
|------|---------------------|------------------------|
| **监听器** | `IsGrantedToolAttributeListener` | `IsGrantedAttributeListener` |
| **监听事件** | `ToolCallArgumentsResolved` | `KernelEvent\ControllerArguments` |
| **目标** | AI 工具方法 | HTTP 控制器方法 |
| **属性来源** | 工具类/方法反射 | 控制器类/方法反射 |
| **主体来源** | 工具参数数组 + 工具实例 | 控制器参数 + Request |
| **异常** | `AccessDeniedException` | `AccessDeniedException` / `HttpException` |

**核心相似性**：两者使用完全相同的底层授权机制（`AuthorizationCheckerInterface` + Voter 系统）。

---

## 完整调用时序

```
                                    时间线 →

Agent          Toolbox        EventDispatcher    Listener           Security
  │               │                │                │                  │
  │──调用工具──→  │                │                │                  │
  │               │──解析参数──→   │                │                  │
  │               │               │                │                  │
  │               │──触发事件──→  │                │                  │
  │               │              │──分发事件──→    │                  │
  │               │              │               │──反射读取属性──→  │
  │               │              │               │──解析 subject──→  │
  │               │              │               │──isGranted()───→ │
  │               │              │               │                  │──Voter投票
  │               │              │               │                  │──决策汇总
  │               │              │               │  ←──结果────────  │
  │               │              │               │                  │
  │               │              │  ←──返回/异常── │                  │
  │               │  ←──────────  │                │                  │
  │  ←──执行/拒绝── │               │                │                  │
```

---

## 扩展指南

### 添加新的安全事件监听器

若需要在工具调用的其他阶段添加安全检查，可以创建新的监听器：

```php
namespace Symfony\AI\AiBundle\Security\EventListener;

use Symfony\AI\Agent\Toolbox\Event\ToolCallResultReceived;

class ToolResultAuditListener
{
    public function __invoke(ToolCallResultReceived $event): void
    {
        // 在工具执行后审计结果
        $this->auditLogger->log(
            $event->getTool(),
            $event->getResult(),
        );
    }
}
```

### 自定义监听器装饰

通过 Symfony 的服务装饰机制，可以在不修改原监听器的情况下添加日志等功能：

```yaml
services:
    App\Security\LoggingIsGrantedToolListener:
        decorates: ai.security.is_granted_attribute_listener
        arguments:
            - '@.inner'
            - '@logger'
```

---

## 总结

`EventListener/` 目录是 AiBundle 安全子系统的运行时核心。目前包含的 `IsGrantedToolAttributeListener` 监听器实现了以下关键能力：

1. **事件驱动的安全拦截** — 在工具执行前自动检查权限，无需工具开发者编写任何授权代码
2. **属性驱动的配置** — 从 `#[IsGrantedTool]` 属性中读取安全规则，实现声明式安全
3. **Symfony Security 集成** — 完整复用 Voter、AuthorizationChecker、ExpressionLanguage 等组件
4. **健壮的降级策略** — 在缺少安全依赖时提供清晰的编译期错误
5. **面向扩展** — 非 final 类设计、标准的事件监听器模式，便于添加新的安全监听器或装饰现有监听器
