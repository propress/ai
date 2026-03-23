# IsGrantedTool 分析报告

## 文件概述

`IsGrantedTool` 是 Symfony AI AiBundle 模块中用于 AI 工具访问控制的 PHP 8 属性（Attribute）类。它借鉴了 Symfony SecurityBundle 中 `#[IsGranted]` 属性的设计理念，将其适配到 AI 工具（Tool）的权限检查场景。开发者通过在工具类或方法上标注 `#[IsGrantedTool]` 属性，即可声明式地定义该工具的访问权限要求，无需在业务逻辑中编写授权检查代码。

**文件路径**: `src/ai-bundle/src/Security/Attribute/IsGrantedTool.php`

**命名空间**: `Symfony\AI\AiBundle\Security\Attribute`

**作者**: Valtteri R <valtzu@gmail.com>

---

## 类定义

```php
namespace Symfony\AI\AiBundle\Security\Attribute;

use Symfony\AI\Platform\Tool\Tool;
use Symfony\Component\ExpressionLanguage\Expression;

#[\Attribute(\Attribute::IS_REPEATABLE | \Attribute::TARGET_CLASS | \Attribute::TARGET_METHOD)]
final class IsGrantedTool
{
    /**
     * @param string|Expression                                                             $attribute     The attribute that will be checked against a given authentication token and optional subject
     * @param array<mixed>|string|Expression|\Closure(array<string,mixed>, Tool):mixed|null $subject       An optional subject - e.g. the current object being voted on
     * @param string|null                                                                   $message       A custom message when access is not granted
     * @param int|null                                                                      $exceptionCode If set, will add the exception code to thrown exception
     */
    public function __construct(
        public string|Expression $attribute,
        public array|string|Expression|\Closure|null $subject = null,
        public ?string $message = null,
        public ?int $exceptionCode = null,
    ) {
    }
}
```

### 关键特征

| 属性 | 说明 |
|------|------|
| **类型** | final class（不可继承） |
| **类别** | PHP 8 属性类（Attribute） |
| **命名空间** | `Symfony\AI\AiBundle\Security\Attribute` |
| **目标** | 类（TARGET_CLASS）和方法（TARGET_METHOD） |
| **可重复** | 是（IS_REPEATABLE），可在同一目标上多次标注 |
| **职责** | 声明 AI 工具的访问权限要求（安全属性 + 可选主体） |

---

## 属性元数据标记

```php
#[\Attribute(\Attribute::IS_REPEATABLE | \Attribute::TARGET_CLASS | \Attribute::TARGET_METHOD)]
```

此标记由三个标志位组合而成：

| 标志 | 值 | 含义 |
|------|-----|------|
| `\Attribute::IS_REPEATABLE` | 位标志 | 允许在同一目标上多次使用该属性 |
| `\Attribute::TARGET_CLASS` | 位标志 | 可应用于类声明 |
| `\Attribute::TARGET_METHOD` | 位标志 | 可应用于方法声明 |

**IS_REPEATABLE 的必要性**：一个工具可能需要同时满足多个权限条件。例如，既需要 `ROLE_ADMIN` 角色，又需要通过某个自定义 Voter 的检查。多次标注允许将这些条件逐一声明：

```php
#[IsGrantedTool('ROLE_ADMIN')]
#[IsGrantedTool('TOOL_OWNER', 'toolId')]
public function deleteResource(int $toolId): void { ... }
```

**TARGET_CLASS + TARGET_METHOD 的设计**：

- **类级别**：为工具类的所有方法设置统一的权限基线
- **方法级别**：为特定方法设置精细的权限要求

当同时存在类级别和方法级别的属性时，监听器会**合并**执行，即两层权限检查都需通过。

---

## 构造方法分析

```php
public function __construct(
    public string|Expression $attribute,
    public array|string|Expression|\Closure|null $subject = null,
    public ?string $message = null,
    public ?int $exceptionCode = null,
)
```

### 参数详解

#### `$attribute`：安全属性（必填）

| 类型 | 说明 | 示例 |
|------|------|------|
| `string` | 安全角色或自定义 Voter 属性字符串 | `'ROLE_USER'`、`'TOOL_ACCESS'` |
| `Expression` | Symfony ExpressionLanguage 表达式 | `new Expression('is_granted("ROLE_ADMIN")')` |

该参数直接传递给 Symfony Security 组件的 `AuthorizationCheckerInterface::isGranted()` 方法的第一个参数。在 Symfony 的安全体系中，此值将被传递给所有已注册的 Voter 进行投票判决。

**角色检查原理**：当值以 `ROLE_` 开头时，Symfony 内置的 `RoleVoter` 会检查当前用户的 Token 中是否包含该角色。

**自定义 Voter 检查**：当值为自定义字符串（如 `'TOOL_ACCESS'`）时，需要有对应的 Voter 注册在容器中来处理该属性。

#### `$subject`：权限主体（可选）

| 类型 | 说明 | 示例 |
|------|------|------|
| `null` | 不传递主体，仅基于属性字符串检查 | — |
| `string` | 从工具方法的已解析参数数组中按键名查找值 | `'itemId'` → 查找 `$arguments['itemId']` |
| `Expression` | 使用 ExpressionLanguage 求值，可访问 `tool` 和 `args` 变量 | `new Expression('args["itemId"]')` |
| `\Closure` | 闭包函数，接收 `(array $arguments, Tool $tool)` 参数 | `fn($args, $tool) => $args['id']` |
| `array` | 多个主体引用的数组，每个元素按上述规则解析 | `['itemId', new Expression('tool.name')]` |

**主体的作用**：在 Symfony 的 Voter 系统中，`subject` 是投票的目标对象。例如，判断用户是否有权编辑某篇文章时，`$subject` 就是该文章对象，而 `$attribute` 是 `'EDIT'`。

#### `$message`：自定义拒绝消息（可选）

| 类型 | 说明 |
|------|------|
| `?string` | 当授权失败时抛出的 `AccessDeniedException` 中使用的消息文本。若为 `null`，监听器将使用 `AccessDecision::getMessage()` 或默认的 `'Access Denied.'` |

#### `$exceptionCode`：自定义异常代码（可选）

| 类型 | 说明 |
|------|------|
| `?int` | 设置 `AccessDeniedException` 的异常代码。若为 `null`，监听器默认使用 `403` |

---

## 构造方法使用的 PHP 特性

### 构造器属性提升（Constructor Property Promotion）

```php
public function __construct(
    public string|Expression $attribute,
    public array|string|Expression|\Closure|null $subject = null,
    ...
)
```

PHP 8 的构造器属性提升使得构造参数自动成为公共属性。这意味着：
- `$isGranted->attribute` 可以直接访问
- `$isGranted->subject` 可以直接访问
- 无需显式定义属性和赋值语句

### 联合类型（Union Types）

PHP 8 的联合类型用于提供灵活的参数类型支持：
- `string|Expression`：支持字符串或表达式对象
- `array|string|Expression|\Closure|null`：支持五种类型，覆盖各种主体解析策略

---

## 设计模式

### 自定义 PHP 属性模式（Custom Attribute Pattern）

PHP 8 属性是一种元编程机制，允许在声明（类、方法、属性等）上附加结构化的元数据。`IsGrantedTool` 利用此机制将安全配置直接与工具声明关联：

```
工具类声明
    ↓
#[IsGrantedTool('ROLE_ADMIN')] ← 元数据（安全配置）
    ↓
反射 API 读取属性
    ↓
监听器在运行时执行权限检查
```

**与传统方式的对比**：

| 方式 | 代码位置 | 维护性 | 灵活性 |
|------|----------|--------|--------|
| 属性声明 | 与工具代码同一文件 | ⭐⭐⭐ 高 | ⭐⭐⭐ 高 |
| YAML/XML 配置 | 单独的配置文件 | ⭐⭐ 中 | ⭐⭐ 中 |
| 硬编码在方法中 | 业务逻辑内部 | ⭐ 低 | ⭐ 低 |

### AOP 声明式安全模式

`IsGrantedTool` 是面向切面编程（AOP）在 PHP 属性中的体现。安全检查作为一个横切关注点（Cross-cutting Concern），通过声明式属性从业务逻辑中分离出来：

```
                          横切关注点（安全）
                               ↓
#[IsGrantedTool('ROLE_USER')]
public function myTool(): string     ← 核心业务逻辑
{
    return 'result';                 ← 不含任何安全代码
}
```

---

## 与 Symfony #[IsGranted] 属性的对比

`IsGrantedTool` 的设计直接借鉴了 `Symfony\Component\Security\Http\Attribute\IsGranted`：

| 特征 | IsGranted（SecurityBundle） | IsGrantedTool（AiBundle） |
|------|----------------------------|---------------------------|
| **应用场景** | HTTP 控制器 | AI 工具 |
| **拦截机制** | 内核事件（`kernel.controller_arguments`） | Agent 事件（`ToolCallArgumentsResolved`） |
| **监听器** | `IsGrantedAttributeListener` | `IsGrantedToolAttributeListener` |
| **`$attribute` 参数** | ✅ 相同 | ✅ 相同 |
| **`$subject` 参数** | ✅ 相同（支持字符串、表达式） | ✅ 扩展（额外支持 Closure） |
| **`$message` 参数** | ✅ 相同 | ✅ 相同 |
| **`$exceptionCode` 参数** | ✅ 相同（`$statusCode`） | ✅ 相同 |
| **异常** | `AccessDeniedException` / `HttpException` | `AccessDeniedException` |

**关键差异**：`IsGrantedTool` 的 `$subject` 支持 `\Closure` 类型，这是因为 AI 工具的参数解析上下文与 HTTP 请求不同——工具没有 Request 对象，但有已解析的参数数组和工具实例。Closure 提供了最大的灵活性来从这些上下文中提取主体。

---

## 使用示例

### 简单角色检查

```php
use Symfony\AI\AiBundle\Security\Attribute\IsGrantedTool;

#[IsGrantedTool('ROLE_USER', message: 'No access to this tool.')]
public function searchDatabase(string $query): array
{
    // 仅 ROLE_USER 角色可调用
    return $this->db->search($query);
}
```

### 基于参数的主体检查

```php
#[IsGrantedTool('DOCUMENT_VIEW', 'documentId', message: 'No access to this document.')]
public function getDocument(int $documentId): Document
{
    // Voter 接收 documentId 值作为 subject，判断当前用户是否有权查看
    return $this->repository->find($documentId);
}
```

### 使用 ExpressionLanguage 的主体

```php
use Symfony\Component\ExpressionLanguage\Expression;

#[IsGrantedTool('TOOL_ACCESS', new Expression('args["projectId"] ?? 0'))]
public function analyzeProject(int $projectId): array
{
    return $this->analyzer->analyze($projectId);
}
```

### 使用 Closure 的主体

```php
#[IsGrantedTool('RESOURCE_EDIT', subject: [fn($args, $tool) => $args['resourceId']])]
public function editResource(int $resourceId, string $content): void
{
    // Closure 提供了最大灵活性
}
```

### 类级别 + 方法级别的多重权限

```php
#[IsGrantedTool('ROLE_ADMIN')]  // 类级别：所有方法需要 ADMIN 角色
class AdminToolkit
{
    #[IsGrantedTool('TOOL_DELETE', 'itemId')]  // 方法级别：额外需要删除权限
    public function deleteItem(int $itemId): void { ... }

    public function listItems(): array { ... }  // 仅需类级别的 ROLE_ADMIN
}
```

---

## 与其他文件的关系

**被引用于**：
- `IsGrantedToolAttributeListener`：在运行时通过反射读取该属性并执行权限检查
- `AiBundle`：在 `symfony/security-core` 未安装时注册自动配置回调，使用该属性时抛出异常

**依赖**：
- `Symfony\AI\Platform\Tool\Tool`：用于 Closure 类型的 `$subject` 参数的类型提示
- `Symfony\Component\ExpressionLanguage\Expression`：用于表达式类型的 `$attribute` 和 `$subject`

**类似类**：
- `Symfony\Component\Security\Http\Attribute\IsGranted`：Symfony SecurityBundle 中用于 HTTP 控制器的等价属性

---

## 总结

`IsGrantedTool` 是 AiBundle 安全子系统的声明式入口点。它通过 PHP 8 属性机制，将 Symfony Security 组件的授权能力无缝引入 AI 工具调用场景。其设计的核心价值在于：

1. **声明式安全** — 权限要求直接标注在工具代码上，与业务逻辑分离，符合 AOP 原则
2. **灵活的主体解析** — 支持字符串引用、表达式求值、闭包计算三种策略，适应各种复杂的权限判定场景
3. **Symfony 生态一致性** — API 设计与 `#[IsGranted]` 高度一致，降低学习成本
4. **可重复与多层级** — IS_REPEATABLE 和 TARGET_CLASS + TARGET_METHOD 的组合允许灵活的权限叠加策略
5. **纯元数据** — 类本身不含任何逻辑，所有行为由 `IsGrantedToolAttributeListener` 在运行时解释执行
