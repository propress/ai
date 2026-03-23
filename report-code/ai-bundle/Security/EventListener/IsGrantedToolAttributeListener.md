# IsGrantedToolAttributeListener 分析报告

## 文件概述

`IsGrantedToolAttributeListener` 是 Symfony AI AiBundle 模块的安全事件监听器，负责在 AI 工具被调用之前执行授权检查。它监听 `ToolCallArgumentsResolved` 事件，通过反射读取工具类和方法上的 `#[IsGrantedTool]` 属性，然后利用 Symfony Security 组件的 `AuthorizationCheckerInterface` 验证当前用户是否具有相应权限。若授权失败，则抛出 `AccessDeniedException` 中断工具调用。

该监听器是 AiBundle 安全子系统的核心运行时组件，实现了基于事件的面向切面编程（AOP）模式，将安全这一横切关注点从工具业务逻辑中完全分离。

**文件路径**: `src/ai-bundle/src/Security/EventListener/IsGrantedToolAttributeListener.php`

**命名空间**: `Symfony\AI\AiBundle\Security\EventListener`

**作者**: Valtteri R <valtzu@gmail.com>

---

## 类定义

```php
namespace Symfony\AI\AiBundle\Security\EventListener;

use Symfony\AI\Agent\Toolbox\Event\ToolCallArgumentsResolved;
use Symfony\AI\AiBundle\Security\Attribute\IsGrantedTool;
use Symfony\Component\ExpressionLanguage\Expression;
use Symfony\Component\ExpressionLanguage\ExpressionLanguage;
use Symfony\Component\Security\Core\Authorization\AccessDecision;
use Symfony\Component\Security\Core\Authorization\AuthorizationCheckerInterface;
use Symfony\Component\Security\Core\Exception\AccessDeniedException;
use Symfony\Component\Security\Core\Exception\RuntimeException;

class IsGrantedToolAttributeListener
{
    public function __construct(
        private readonly AuthorizationCheckerInterface $authChecker,
        private ?ExpressionLanguage $expressionLanguage = null,
    ) {
    }

    public function __invoke(ToolCallArgumentsResolved $event): void { ... }

    private function getIsGrantedSubject(
        string|Expression|\Closure $subjectRef,
        object $tool,
        array $arguments
    ): mixed { ... }
}
```

### 关键特征

| 属性 | 说明 |
|------|------|
| **类型** | class（非 final，可继承） |
| **命名空间** | `Symfony\AI\AiBundle\Security\EventListener` |
| **监听事件** | `ToolCallArgumentsResolved`（Agent Toolbox 事件） |
| **核心依赖** | `AuthorizationCheckerInterface`（Symfony Security） |
| **可选依赖** | `ExpressionLanguage`（Symfony ExpressionLanguage） |
| **职责** | 在工具调用前拦截并执行 `#[IsGrantedTool]` 属性声明的授权检查 |

---

## 构造方法

```php
public function __construct(
    private readonly AuthorizationCheckerInterface $authChecker,
    private ?ExpressionLanguage $expressionLanguage = null,
)
```

### 参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `$authChecker` | `AuthorizationCheckerInterface` | 是 | Symfony 安全组件的授权检查器，负责执行 Voter 投票和角色检查 |
| `$expressionLanguage` | `?ExpressionLanguage` | 否 | 表达式语言引擎，用于求值 `Expression` 类型的 subject。若未注入，在首次需要时惰性创建 |

**`$authChecker` 的作用**：这是 Symfony Security 组件的核心接口，其 `isGranted(string $attribute, mixed $subject = null)` 方法将安全属性和主体传递给已注册的所有 Voter 进行投票。最终根据 `AccessDecisionManager` 的策略（affirmative/consensus/unanimous）决定是否授权。

**`$expressionLanguage` 的惰性初始化**：

```php
$this->expressionLanguage ??= new ExpressionLanguage();
```

使用 `??=`（空合并赋值）在首次需要时创建实例。这意味着：
- 若通过 DI 注入了 `ExpressionLanguage` 服务（带自定义提供者/函数），则使用注入版本
- 若注入为 `null`（服务不存在），则按需创建默认实例
- 若所有 `#[IsGrantedTool]` 属性都不使用 Expression 类型 subject，则永不创建

---

## `__invoke` 方法（事件处理器）

```php
public function __invoke(ToolCallArgumentsResolved $event): void
```

### 输入

| 参数 | 类型 | 说明 |
|------|------|------|
| `$event` | `ToolCallArgumentsResolved` | 工具调用参数已解析事件，携带工具实例、元数据和已解析的参数 |

### 输出

- **正常**：`void`（无返回值，事件继续传播，工具正常执行）
- **异常**：抛出 `AccessDeniedException`（授权失败，中断工具执行）

### 详细执行流程

#### 步骤 1：获取工具实例和反射信息

```php
$tool = $event->getTool();
$class = new \ReflectionClass($tool);
$method = $class->getMethod($event->getMetadata()->getReference()->getMethod());
```

- `$event->getTool()` 返回工具对象实例（PHP 对象）
- 通过 `ReflectionClass` 获取类的反射信息
- 通过 `$event->getMetadata()->getReference()->getMethod()` 获取即将执行的方法名称（默认为 `__invoke`）
- 获取该方法的反射对象

#### 步骤 2：收集 `#[IsGrantedTool]` 属性

```php
$classAttributes = $class->getAttributes(IsGrantedTool::class);
$methodAttributes = $method->getAttributes(IsGrantedTool::class);

if (!$classAttributes && !$methodAttributes) {
    return;
}
```

- 分别从类级别和方法级别收集 `#[IsGrantedTool]` 属性
- **早期返回优化**：若工具没有标注任何 `#[IsGrantedTool]` 属性，立即返回，避免不必要的授权检查开销

#### 步骤 3：遍历属性并执行授权检查

```php
$arguments = $event->getArguments();

foreach (array_merge($classAttributes, $methodAttributes) as $attr) {
    $attribute = $attr->newInstance();
    $subject = null;

    if ($subjectRef = $attribute->subject) {
        if (\is_array($subjectRef)) {
            foreach ($subjectRef as $refKey => $ref) {
                $subject[\is_string($refKey) ? $refKey : (string) $ref] = 
                    $this->getIsGrantedSubject($ref, $tool, $arguments);
            }
        } else {
            $subject = $this->getIsGrantedSubject($subjectRef, $tool, $arguments);
        }
    }
    // ... 授权检查（见下文）
}
```

**属性合并顺序**：`array_merge($classAttributes, $methodAttributes)` 先检查类级别属性，再检查方法级别属性。所有属性都必须通过，任一失败即抛出异常。

**`$attr->newInstance()`**：调用反射 API 的 `newInstance()` 方法，实际执行属性类的构造函数，创建 `IsGrantedTool` 实例。

**Subject 解析**：
- 若 `$subject` 为 `null`：不传递主体
- 若 `$subject` 为数组：逐个解析数组元素，构建关联数组
- 若 `$subject` 为标量/对象：委托 `getIsGrantedSubject()` 解析

**数组 Subject 的键名处理**：

```php
$subject[\is_string($refKey) ? $refKey : (string) $ref] = ...
```

- 若原数组有字符串键（关联数组），保留原始键名
- 若原数组为索引数组，使用引用值的字符串形式作为键名

#### 步骤 4：AccessDecision 兼容层

```php
$accessDecision = null;
// bc layer
if (class_exists(AccessDecision::class)) {
    $accessDecision = new AccessDecision();
    $accessDecision->isGranted = false;
    $decision = &$accessDecision->isGranted;
}
```

这是一个向后兼容层（BC layer），用于支持 Symfony Security 7.3+ 引入的 `AccessDecision` 类：

- 若 `AccessDecision` 类存在（Symfony 7.3+），创建实例并通过引用绑定 `$decision` 变量
- 若类不存在（较早版本），直接使用 `isGranted()` 的布尔返回值
- `$decision = &$accessDecision->isGranted` 使用引用赋值，确保 `$decision` 和 `$accessDecision->isGranted` 指向同一内存

#### 步骤 5：执行授权检查

```php
if (!$decision = $this->authChecker->isGranted($attribute->attribute, $subject, $accessDecision)) {
    $message = $attribute->message 
        ?: (class_exists(AccessDecision::class, false) ? $accessDecision->getMessage() : 'Access Denied.');

    $e = new AccessDeniedException($message, code: $attribute->exceptionCode ?? 403);
    $e->setAttributes([$attribute->attribute]);
    $e->setSubject($subject);
    if ($accessDecision && method_exists($e, 'setAccessDecision')) {
        $e->setAccessDecision($accessDecision);
    }

    throw $e;
}
```

**授权检查调用**：`$this->authChecker->isGranted()` 触发 Symfony Security 的 Voter 投票机制。

**拒绝消息的优先级**：
1. 属性中自定义的 `$message`
2. `AccessDecision::getMessage()` 提供的消息（Symfony 7.3+）
3. 默认消息 `'Access Denied.'`

**异常构建**：
- 创建 `AccessDeniedException` 并设置异常代码（默认 403）
- 附加安全属性列表（`setAttributes`）
- 附加主体信息（`setSubject`）
- 若支持，附加 `AccessDecision` 对象（`setAccessDecision`）

---

## `getIsGrantedSubject` 方法（主体解析）

```php
/**
 * @param array<string, mixed> $arguments
 */
private function getIsGrantedSubject(
    string|Expression|\Closure $subjectRef,
    object $tool,
    array $arguments
): mixed
```

### 输入

| 参数 | 类型 | 说明 |
|------|------|------|
| `$subjectRef` | `string\|Expression\|\Closure` | 主体引用，三种类型对应三种解析策略 |
| `$tool` | `object` | 当前工具实例 |
| `$arguments` | `array<string, mixed>` | 已解析的工具调用参数（键为参数名，值为参数值） |

### 输出

| 返回值 | 类型 | 说明 |
|--------|------|------|
| 解析后的主体 | `mixed` | 传递给 `AuthorizationCheckerInterface::isGranted()` 的 subject 参数 |

### 三种解析策略

#### 策略 1：Closure 解析

```php
if ($subjectRef instanceof \Closure) {
    return $subjectRef($arguments, $tool);
}
```

闭包直接被调用，接收两个参数：
- `$arguments`：已解析的工具参数数组
- `$tool`：工具对象实例

**使用场景**：需要复杂的主体计算逻辑，例如从多个参数组合生成主体对象。

**示例**：

```php
#[IsGrantedTool('EDIT', subject: fn($args, $tool) => new ResourceSubject($args['id'], $tool))]
```

#### 策略 2：Expression 解析

```php
if ($subjectRef instanceof Expression) {
    $this->expressionLanguage ??= new ExpressionLanguage();

    return $this->expressionLanguage->evaluate($subjectRef, [
        'tool' => $tool,
        'args' => $arguments,
    ]);
}
```

使用 Symfony ExpressionLanguage 组件求值表达式。表达式上下文中可用的变量：

| 变量名 | 类型 | 说明 |
|--------|------|------|
| `tool` | `object` | 当前工具实例 |
| `args` | `array<string, mixed>` | 已解析的工具调用参数 |

**使用场景**：在属性声明中直接嵌入轻量级的求值逻辑。

**示例**：

```php
#[IsGrantedTool('TOOL_ACCESS', new Expression('args["itemId"] ?? 0'))]
```

#### 策略 3：字符串参数引用

```php
if (!\array_key_exists($subjectRef, $arguments)) {
    throw new RuntimeException(\sprintf(
        'Could not find the subject "%s" for the #[IsGranted] attribute. '
        . 'Try adding a "$%s" argument to your tool method.',
        $subjectRef, $subjectRef
    ));
}

return $arguments[$subjectRef];
```

字符串被视为工具参数数组中的键名，直接从已解析参数中查找对应的值。

**使用场景**：最简单的情况——主体就是工具方法的某个参数值。

**异常处理**：若参数名不存在于参数数组中，抛出 `RuntimeException` 并提供清晰的错误提示。

**示例**：

```php
#[IsGrantedTool('DOCUMENT_VIEW', 'documentId')]
public function viewDocument(int $documentId): Document { ... }
// 'documentId' → $arguments['documentId'] → 传递给 Voter 的 subject
```

---

## 设计模式

### 事件监听器模式（Event Listener Pattern）

```
ToolCallArgumentsResolved 事件
    ↓ Symfony EventDispatcher
IsGrantedToolAttributeListener::__invoke()
    ├── 授权通过 → 事件继续传播 → 工具执行
    └── 授权失败 → throw AccessDeniedException → 工具不执行
```

监听器通过 Symfony EventDispatcher 与 Agent Toolbox 解耦。工具执行流程不需要知道安全检查的存在，安全逻辑也不需要了解工具的业务实现。

### 策略模式（Strategy Pattern）

`getIsGrantedSubject()` 方法根据 `$subjectRef` 的类型选择不同的解析策略：

```
subjectRef 类型
    ├── \Closure    → 直接调用闭包
    ├── Expression  → ExpressionLanguage 求值
    └── string      → 参数数组键值查找
```

三种策略封装在同一方法中，通过 `instanceof` 检查进行分派。这为主体解析提供了从简单到复杂的全范围支持。

### AOP 模式（面向切面编程）

安全检查作为横切关注点，通过事件机制织入工具调用流程：

```
                    横切关注点（安全）
                         ↓
工具方法声明 ──→ #[IsGrantedTool] 属性 ──→ 反射读取
    ↓                                        ↓
工具调用 ──→ ToolCallArgumentsResolved ──→ 监听器检查 ──→ 工具执行
                                              ↓ (失败)
                                    AccessDeniedException
```

**与传统 AOP 框架的对比**：不使用代码生成或代理类，而是利用 PHP 原生反射 API + Symfony 事件系统实现"运行时织入"。

### 装饰器模式的变体

从效果上看，监听器为工具调用添加了一层安全装饰：

```
原始行为：    Agent → 工具调用 → 执行
装饰后：      Agent → 工具调用 → [安全检查] → 执行
```

不同于传统装饰器（包装对象），这里通过事件拦截实现了相同的效果。

---

## 完整调用流程

```
1. AI Agent 接收到模型的工具调用请求
    ↓
2. Agent Toolbox 反序列化工具调用参数
    ↓
3. 参数解析完成，触发 ToolCallArgumentsResolved 事件
    ↓
4. Symfony EventDispatcher 将事件分发给已注册的监听器
    ↓
5. IsGrantedToolAttributeListener::__invoke() 被调用
    ↓
6. 通过反射获取工具类和目标方法的 ReflectionClass / ReflectionMethod
    ↓
7. 读取类级别和方法级别的 #[IsGrantedTool] 属性
    ├── 无属性 → 立即返回（快速路径）
    └── 有属性 → 继续
    ↓
8. 遍历每个 #[IsGrantedTool] 属性：
    ├── 8a. 解析 subject（Closure / Expression / 字符串）
    ├── 8b. 构建 AccessDecision 对象（BC 层）
    ├── 8c. 调用 $authChecker->isGranted($attribute, $subject)
    │        ↓
    │   Symfony Security 内部：
    │   ├── AccessDecisionManager 收集所有 Voter
    │   ├── 每个 Voter 对 ($attribute, $subject) 投票
    │   └── 根据策略（affirmative/consensus/unanimous）汇总投票结果
    │        ↓
    ├── 8d. 授权通过 → 继续下一个属性
    └── 8e. 授权失败 → 构建并抛出 AccessDeniedException
    ↓
9. 所有属性检查通过 → 方法正常返回
    ↓
10. 事件继续传播 → 工具方法被执行
```

---

## 依赖注入容器中的注册

### 服务定义（services.php）

```php
->set('ai.security.is_granted_attribute_listener', IsGrantedToolAttributeListener::class)
    ->args([
        service('security.authorization_checker'),
        service('expression_language')->nullOnInvalid(),
    ])
    ->tag('kernel.event_listener')
```

| 配置项 | 值 | 说明 |
|--------|-----|------|
| **服务 ID** | `ai.security.is_granted_attribute_listener` | 在 DI 容器中的唯一标识 |
| **类** | `IsGrantedToolAttributeListener` | 监听器类 |
| **参数 1** | `security.authorization_checker` | Symfony Security 的授权检查器服务 |
| **参数 2** | `expression_language`（可为 null） | ExpressionLanguage 服务，`nullOnInvalid()` 表示服务不存在时注入 null |
| **标签** | `kernel.event_listener` | 标记为事件监听器，Symfony 自动注册到 EventDispatcher |

### 事件绑定机制

标签 `kernel.event_listener` 使 Symfony 通过 `__invoke` 方法的参数类型提示自动推断监听的事件类型：

```php
public function __invoke(ToolCallArgumentsResolved $event): void
```

Symfony 的 `RegisterListenersPass` 编译器传递会检测到参数类型为 `ToolCallArgumentsResolved`，自动将监听器注册到该事件上。

### 优雅降级（AiBundle.php）

当 `symfony/security-core` 未安装时，AiBundle 在容器编译阶段处理降级：

```php
if (!ContainerBuilder::willBeAvailable(
    'symfony/security-core',
    AuthorizationCheckerInterface::class,
    ['symfony/ai-bundle']
)) {
    $builder->removeDefinition('ai.security.is_granted_attribute_listener');
    $builder->registerAttributeForAutoconfiguration(
        IsGrantedTool::class,
        static fn () => throw new InvalidArgumentException(
            'Using #[IsGrantedTool] attribute requires additional dependencies. '
            . 'Try running "composer install symfony/security-core".'
        ),
    );
}
```

**降级策略**：

| 操作 | 说明 |
|------|------|
| `removeDefinition()` | 从容器中完全移除监听器服务定义 |
| `registerAttributeForAutoconfiguration()` | 注册属性自动配置回调，当容器编译时发现某个服务类使用了 `#[IsGrantedTool]` 属性，立即抛出 `InvalidArgumentException` |

**效果**：
- 若未安装 `symfony/security-core`：监听器不存在，使用属性时在容器编译阶段就会得到明确的错误提示
- 若已安装 `symfony/security-core`：监听器正常注册，属性在运行时被读取和执行

---

## 与 Symfony Security 组件的集成

### AuthorizationCheckerInterface

```
IsGrantedToolAttributeListener
    ↓ isGranted($attribute, $subject)
AuthorizationCheckerInterface（通常是 AuthorizationChecker 实现）
    ↓
AccessDecisionManager
    ↓ decide($token, [$attribute], $subject)
Voter 1: vote($token, $subject, [$attribute]) → GRANT/DENY/ABSTAIN
Voter 2: vote($token, $subject, [$attribute]) → GRANT/DENY/ABSTAIN
Voter N: vote($token, $subject, [$attribute]) → GRANT/DENY/ABSTAIN
    ↓
决策策略（affirmative / consensus / unanimous）
    ↓
最终结果：true / false
```

### 自定义 Voter 示例

```php
use Symfony\Component\Security\Core\Authorization\Voter\Voter;

class ToolAccessVoter extends Voter
{
    protected function supports(string $attribute, mixed $subject): bool
    {
        return 'TOOL_ACCESS' === $attribute;
    }

    protected function voteOnAttribute(string $attribute, mixed $subject, TokenInterface $token): bool
    {
        $user = $token->getUser();
        // 自定义授权逻辑
        return $user->hasToolAccess($subject);
    }
}
```

### AccessDeniedException 详解

当授权失败时抛出的异常包含丰富的上下文信息：

```php
$e = new AccessDeniedException($message, code: $exceptionCode ?? 403);
$e->setAttributes([$attribute]);   // 被检查的安全属性
$e->setSubject($subject);          // 被检查的主体
$e->setAccessDecision($accessDecision); // 完整的投票决策（Symfony 7.3+）
```

这些信息可被上层代码用于：
- 日志记录：记录被拒绝的具体权限和资源
- 审计追踪：跟踪授权失败事件
- 错误展示：向用户展示详细的拒绝原因

---

## 测试验证

### 测试文件

`src/ai-bundle/tests/Security/IsGrantedToolAttributeListenerTest.php`

### 测试覆盖场景

| 测试方法 | 场景 | 验证内容 |
|----------|------|----------|
| `testItWillThrowWhenNotGranted` | 授权失败 | 方法级属性、Expression 主体、类级属性均抛出 `AccessDeniedException` |
| `testItWillNotThrowWhenGranted` | 授权通过 | `ROLE_USER` 检查通过，工具正常执行 |
| `testItWillProvideArgumentAsSubject` | 字符串主体 | 方法级和类级属性均正确传递参数值作为 subject |
| `testItWillEvaluateSubjectExpression` | Expression 主体 | ExpressionLanguage 正确求值 `args["itemId"]` |

### 测试夹具（Fixture）

**ToolWithIsGrantedOnMethod**：方法级属性测试

```php
class ToolWithIsGrantedOnMethod
{
    #[IsGrantedTool('ROLE_USER', message: 'No access to simple tool.')]
    public function simple(): bool { ... }

    #[IsGrantedTool('test:permission', 'itemId', message: 'No access to argumentAsSubject tool.')]
    public function argumentAsSubject(int $itemId): int { ... }

    #[IsGrantedTool('test:permission', new Expression('args["itemId"]'), message: 'No access to expressionAsSubject tool.')]
    public function expressionAsSubject(int $itemId): int { ... }
}
```

**ToolWithIsGrantedOnClass**：类级属性测试

```php
#[IsGrantedTool('test:permission', new Expression('args["itemId"] ?? 0'), message: 'No access to ToolWithIsGrantedOnClass tool.')]
class ToolWithIsGrantedOnClass
{
    public function __invoke(int $itemId): void { ... }
}
```

---

## 扩展点

### 1. 自定义 Voter

通过实现 `Voter` 抽象类或 `VoterInterface`，开发者可以创建任意复杂的授权逻辑：

```php
// 基于工具参数的细粒度权限控制
class AiToolVoter extends Voter
{
    protected function supports(string $attribute, mixed $subject): bool
    {
        return str_starts_with($attribute, 'AI_TOOL_');
    }

    protected function voteOnAttribute(string $attribute, mixed $subject, TokenInterface $token): bool
    {
        // 基于用户配额、工具类型、参数内容等的复杂判定
        return $this->policyEngine->evaluate($token->getUser(), $attribute, $subject);
    }
}
```

### 2. ExpressionLanguage 扩展

注入自定义的 `ExpressionLanguage` 实例以添加自定义函数：

```php
$expressionLanguage = new ExpressionLanguage();
$expressionLanguage->register('tool_owner', ...);
// 然后在属性中使用：
// #[IsGrantedTool('EDIT', new Expression('tool_owner(args["id"])'))]
```

### 3. Closure 的灵活性

Closure 类型的 subject 允许最大程度的自定义：

```php
#[IsGrantedTool('MANAGE', subject: fn($args, $tool) => [
    'resource_type' => get_class($tool),
    'resource_id' => $args['id'],
    'operation' => 'manage',
])]
```

### 4. 监听器继承

`IsGrantedToolAttributeListener` 不是 `final` 类，允许子类化以扩展行为（如添加日志、审计等）。

---

## 外部知识：关键概念

### Symfony Voter 系统

Voter 是 Symfony Security 中的核心授权机制。每个 Voter 针对特定类型的权限检查进行投票：

- `GRANT`（授予）：Voter 认为应该授权
- `DENY`（拒绝）：Voter 认为应该拒绝
- `ABSTAIN`（弃权）：Voter 不处理此类权限

`AccessDecisionManager` 收集所有投票后，根据配置的策略决定最终结果。

### PHP 8 属性与反射

```php
// 获取类上的属性
$class->getAttributes(IsGrantedTool::class);
// 获取方法上的属性
$method->getAttributes(IsGrantedTool::class);
// 实例化属性对象
$attr->newInstance();
```

反射 API 的 `getAttributes()` 返回 `ReflectionAttribute[]`，`newInstance()` 执行属性类的构造函数并返回实例。

### ToolCallArgumentsResolved 事件

该事件由 Agent Toolbox 在工具调用参数反序列化完成后、工具方法实际执行前触发。这是插入安全检查的理想时机——参数已可用，但工具尚未执行。

---

## 与其他文件的关系

**依赖**：
- `IsGrantedTool`：读取并解释此属性类的实例
- `ToolCallArgumentsResolved`：监听的事件类（来自 Agent 组件）
- `AuthorizationCheckerInterface`：Symfony Security 的授权接口
- `ExpressionLanguage`：可选的表达式求值引擎
- `AccessDecision`：Symfony 7.3+ 的投票决策对象
- `AccessDeniedException`：授权失败时抛出的异常

**被引用于**：
- `services.php`：作为 `ai.security.is_granted_attribute_listener` 服务注册
- `AiBundle.php`：在 `symfony/security-core` 不可用时移除此服务定义

**关联测试**：
- `IsGrantedToolAttributeListenerTest`：完整的单元测试
- `ToolWithIsGrantedOnClass`、`ToolWithIsGrantedOnMethod`：测试夹具

---

## 总结

`IsGrantedToolAttributeListener` 是 AiBundle 安全子系统的运行时引擎，将声明式的 `#[IsGrantedTool]` 属性转化为实际的授权检查行为。其设计的核心价值在于：

1. **事件驱动的 AOP** — 通过监听 `ToolCallArgumentsResolved` 事件，在不修改工具代码的情况下织入安全检查
2. **多策略主体解析** — 支持字符串引用、表达式求值、闭包调用三种策略，覆盖从简单到复杂的所有主体提取场景
3. **Symfony 安全体系的完整集成** — 复用 Voter 投票机制、AccessDecisionManager 决策策略、ExpressionLanguage 表达式引擎
4. **向后兼容** — AccessDecision BC 层确保在不同版本的 Symfony Security 组件下都能正确工作
5. **优雅降级** — 与 AiBundle 配合，在 `symfony/security-core` 未安装时提供清晰的编译期错误而非运行时崩溃
6. **零侵入** — 工具类完全不需要感知安全检查的存在，所有逻辑由监听器和属性声明驱动
