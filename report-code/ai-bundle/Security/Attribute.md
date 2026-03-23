# Attribute 目录分析报告

## 目录职责

`Attribute/` 目录包含 Symfony AI AiBundle 安全子系统的 PHP 8 属性（Attribute）定义。这些属性类作为声明式安全配置的载体，允许开发者通过在 AI 工具类和方法上标注属性来定义访问控制规则，无需在业务逻辑中嵌入授权代码。

**目录路径**: `src/ai-bundle/src/Security/Attribute/`

---

## 包含的文件清单

| 文件 | 类型 | 说明 |
|------|------|------|
| `IsGrantedTool.php` | PHP 8 属性类（final） | AI 工具访问控制属性，声明工具调用所需的安全权限 |

---

## 架构概览

### 类图

```
                PHP 内置属性系统
                      │
                #[\Attribute]
                      │
        ┌─────────────┴──────────────┐
        │                            │
  Symfony Security              AiBundle Security
  #[IsGranted]                  #[IsGrantedTool]
  (控制器访问控制)                (工具访问控制)
        │                            │
        ↓                            ↓
  IsGrantedAttributeListener    IsGrantedToolAttributeListener
  (kernel.controller_arguments)  (ToolCallArgumentsResolved)
```

### 核心设计原则

1. **声明式配置** — 属性取代显式的授权检查代码，将安全规则提升为元数据
2. **关注点分离** — 属性仅持有配置数据，不包含执行逻辑
3. **Symfony 一致性** — API 设计与 Symfony SecurityBundle 的 `#[IsGranted]` 高度一致
4. **灵活的主体解析** — 支持字符串、Expression、Closure、数组四种主体类型

---

## 设计模式详解

### 自定义属性模式（Custom Attribute Pattern）

PHP 8 属性是一种原生的元编程机制，允许将结构化的元数据附加到代码声明上：

```php
// 属性定义（元数据结构）
#[\Attribute(\Attribute::IS_REPEATABLE | \Attribute::TARGET_CLASS | \Attribute::TARGET_METHOD)]
final class IsGrantedTool
{
    public function __construct(
        public string|Expression $attribute,
        public array|string|Expression|\Closure|null $subject = null,
        public ?string $message = null,
        public ?int $exceptionCode = null,
    ) {
    }
}

// 属性使用（声明式标注）
#[IsGrantedTool('ROLE_ADMIN')]
class AdminTool { ... }

// 属性读取（运行时反射）
$attrs = (new \ReflectionClass($tool))->getAttributes(IsGrantedTool::class);
foreach ($attrs as $attr) {
    $instance = $attr->newInstance(); // 创建 IsGrantedTool 实例
}
```

**模式的三个阶段**：

| 阶段 | 参与者 | 操作 |
|------|--------|------|
| 定义 | `IsGrantedTool` | 定义元数据结构（构造参数即元数据字段） |
| 标注 | 工具开发者 | 在工具类/方法上声明安全要求 |
| 读取 | `IsGrantedToolAttributeListener` | 通过反射读取属性并执行安全检查 |

### 配置对象模式（Configuration Object Pattern）

`IsGrantedTool` 本质上是一个不可变的配置对象（Value Object）。它通过构造函数接收所有配置，并通过公共属性暴露给消费者。PHP 8 的构造器属性提升使这一模式的实现极为简洁。

---

## IsGrantedTool 属性详解

### 属性元数据

```
#[\Attribute(
    \Attribute::IS_REPEATABLE     → 同一目标可多次标注
  | \Attribute::TARGET_CLASS      → 可标注在类上
  | \Attribute::TARGET_METHOD     → 可标注在方法上
)]
```

### 参数总览

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `$attribute` | `string\|Expression` | 是 | — | 安全属性（角色或自定义权限标识） |
| `$subject` | `array\|string\|Expression\|\Closure\|null` | 否 | `null` | 权限检查的主体 |
| `$message` | `?string` | 否 | `null` | 自定义拒绝消息 |
| `$exceptionCode` | `?int` | 否 | `null` | 自定义异常代码 |

### 主体解析策略

```
$subject 类型              解析方式                          使用场景
─────────────────────────────────────────────────────────────────────
null                     → 不传递主体                        → 纯角色检查
string                   → 从参数数组中按键名查找              → 简单参数引用
Expression               → ExpressionLanguage 求值           → 带逻辑的参数提取
\Closure                 → 直接调用 fn($args, $tool)         → 复杂主体构建
array                    → 逐个元素解析，构建关联数组          → 多字段主体
```

---

## 使用示例

### 基本角色检查（类级别）

```php
use Symfony\AI\AiBundle\Security\Attribute\IsGrantedTool;

#[IsGrantedTool('ROLE_USER', message: 'This tool requires authentication.')]
class SearchTool
{
    public function __invoke(string $query): array
    {
        return $this->searchEngine->search($query);
    }
}
```

### 参数级权限（方法级别）

```php
class DocumentTool
{
    #[IsGrantedTool('DOCUMENT_VIEW', 'documentId')]
    public function view(int $documentId): Document { ... }

    #[IsGrantedTool('DOCUMENT_EDIT', 'documentId')]
    public function edit(int $documentId, string $content): void { ... }
}
```

### 多重权限检查

```php
#[IsGrantedTool('ROLE_ADMIN')]
#[IsGrantedTool('TOOL_OWNER', 'projectId')]
public function deleteProject(int $projectId): void { ... }
```

---

## 在 AiBundle 安全子系统中的定位

```
Security/
├── Attribute/                     ← 本目录：声明式安全配置
│   └── IsGrantedTool.php          ← 工具权限属性定义
│
└── EventListener/                 ← 运行时安全执行
    └── IsGrantedToolAttributeListener.php  ← 属性读取 + 授权检查
```

**Attribute** 目录与 **EventListener** 目录构成了安全子系统的两个层面：

| 层面 | 目录 | 职责 | 时机 |
|------|------|------|------|
| 声明层 | Attribute/ | 定义"需要什么权限" | 编码时 |
| 执行层 | EventListener/ | 执行"检查是否有权限" | 运行时 |

这种分离确保了安全规则的声明独立于安全检查的执行，符合 AOP 的核心原则。

---

## 与其他组件属性的对比

| 特征 | AiBundle `#[IsGrantedTool]` | Symfony `#[IsGranted]` |
|------|---------------------------|------------------------|
| **作用域** | AI 工具方法 | HTTP 控制器方法 |
| **目标** | TARGET_CLASS + TARGET_METHOD | TARGET_CLASS + TARGET_METHOD |
| **可重复** | ✅ IS_REPEATABLE | ✅ IS_REPEATABLE |
| **subject 支持 Closure** | ✅ 是 | ❌ 否 |
| **拦截点** | `ToolCallArgumentsResolved` 事件 | `kernel.controller_arguments` 事件 |

---

## 扩展指南

### 添加新的安全属性

若未来需要更多类型的工具安全检查（如速率限制、IP 白名单等），可以在此目录添加新的属性类：

```php
namespace Symfony\AI\AiBundle\Security\Attribute;

#[\Attribute(\Attribute::TARGET_CLASS | \Attribute::TARGET_METHOD)]
final class RateLimitTool
{
    public function __construct(
        public int $maxCalls,
        public int $periodSeconds,
        public ?string $message = null,
    ) {
    }
}
```

然后在 `EventListener/` 目录中添加对应的监听器来解释执行该属性。

---

## 总结

`Attribute/` 目录目前仅包含一个文件 `IsGrantedTool.php`，但它在 AiBundle 安全架构中扮演着关键角色：

1. **安全规则的唯一声明入口** — 所有工具级别的访问控制都通过此属性声明
2. **纯元数据** — 不含任何执行逻辑，职责清晰
3. **Symfony 风格的 API** — 与 `#[IsGranted]` 一致的使用体验
4. **灵活且可扩展** — 支持多种主体类型，覆盖各种授权场景
5. **面向未来** — 目录结构为添加更多安全属性预留了空间
