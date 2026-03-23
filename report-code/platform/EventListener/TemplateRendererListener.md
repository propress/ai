# TemplateRendererListener 分析报告

## 文件概述
TemplateRendererListener 是一个事件监听器类，负责在模型调用前渲染消息模板。当请求选项中包含 template_vars 时，该监听器会遍历 MessageBag 中的所有消息，识别包含模板的消息内容，使用配置的模板渲染器将模板变量替换为实际值。它还支持对象参数的自动规范化，通过可选的 NormalizerInterface 将对象转换为数组或可渲染的格式。该类实现了 EventSubscriberInterface，提供了自注册能力。

## 类/接口定义

### TemplateRendererListener
- **类型**: final class
- **继承/实现**: implements EventSubscriberInterface
- **命名空间**: Symfony\AI\Platform\EventListener
- **职责**: 监听 InvocationEvent 事件，处理消息模板的渲染和模板变量的规范化，支持在系统消息和用户消息中使用模板引擎

## 方法分析

### __construct()
- **可见性**: public
- **参数**: 
  - `$rendererRegistry` (`TemplateRendererRegistryInterface`): 模板渲染器注册表，用于根据模板类型获取对应的渲染器
  - `$normalizer` (`?NormalizerInterface`): 可选的对象规范化器，用于将对象类型的模板变量转换为数组
- **返回值**: void
- **功能说明**: 初始化监听器，注入必需的渲染器注册表和可选的规范化器
- **注意事项**: 如果模板变量包含对象且未提供 normalizer，渲染时会抛出异常

### __invoke()
- **可见性**: public
- **参数**: 
  - `$event` (`InvocationEvent`): 包含模型调用上下文的事件对象
- **返回值**: `void` - 直接修改事件对象
- **功能说明**: 
  1. 检查选项中是否存在 template_vars，不存在则跳过
  2. 验证 template_vars 和 template_options 的类型
  3. 规范化模板变量（处理对象类型）
  4. 遍历消息并渲染包含模板的内容
  5. 更新事件的输入为渲染后的消息集合
  6. 从选项中移除 template_vars 和 template_options
- **注意事项**: 只处理 MessageBag 类型的输入，其他类型直接跳过

### getSubscribedEvents()
- **可见性**: public static
- **参数**: 无
- **返回值**: `array` - 事件订阅配置数组
- **功能说明**: 声明该监听器订阅 InvocationEvent 事件，并将 __invoke 方法作为处理器
- **注意事项**: 这是 EventSubscriberInterface 接口的要求，使监听器能够自动注册

### normalizeTemplateVars()
- **可见性**: private
- **参数**: 
  - `$templateVars` (`array<string, mixed>`): 原始模板变量数组
  - `$normalizerContext` (`array<string, mixed>`): 规范化器上下文配置
- **返回值**: `array<string, mixed>` - 规范化后的模板变量
- **功能说明**: 
  1. 验证变量键必须是字符串
  2. 对于对象类型的值（非 Stringable），使用 normalizer 转换
  3. 其他类型的值保持不变
- **注意事项**: 如果遇到非 Stringable 对象但未配置 normalizer，会抛出 InvalidArgumentException

### renderMessage()
- **可见性**: private
- **参数**: 
  - `$message` (`MessageInterface`): 要渲染的消息对象
  - `$templateVars` (`array<string, mixed>`): 模板变量
- **返回值**: `MessageInterface` - 渲染后的消息对象（可能是新对象或原对象）
- **功能说明**: 
  1. 如果是 SystemMessage 且内容是 Template，渲染并创建新的 SystemMessage
  2. 如果是 UserMessage，遍历所有内容项，将 Template 类型的内容渲染为 Text
  3. 其他类型的消息直接返回
- **注意事项**: 只处理 SystemMessage 和 UserMessage，AssistantMessage 等不处理

## 设计模式

### 事件订阅者模式（Event Subscriber Pattern）
实现 EventSubscriberInterface，提供自注册能力：
- **自描述**: 监听器自己声明订阅哪些事件
- **自动注册**: 框架可以自动扫描和注册订阅者
- **集中配置**: 事件映射集中在 getSubscribedEvents 方法中

### 责任链模式（Chain of Responsibility）
作为事件处理链中的一环，处理特定职责后传递给下一个处理器。

### 策略模式（Strategy Pattern）
通过 TemplateRendererRegistryInterface 支持多种模板引擎（Twig、Blade 等），运行时根据模板类型选择合适的渲染器。

### 适配器模式（Adapter Pattern）
通过 NormalizerInterface 将不同类型的对象适配为可在模板中使用的格式。

## 扩展点

虽然类被声明为 final，但可以通过以下方式扩展功能：

```php
// 创建自定义的模板渲染器
class CustomTemplateRenderer implements TemplateRendererInterface
{
    public function render(Template $template, array $vars): string
    {
        // 自定义渲染逻辑
        return str_replace(
            array_map(fn($k) => "{{{$k}}}", array_keys($vars)),
            array_values($vars),
            $template->getContent()
        );
    }
}

// 注册到注册表
$registry->register('custom', new CustomTemplateRenderer());

// 创建支持更多消息类型的监听器
final class ExtendedTemplateRendererListener implements EventSubscriberInterface
{
    public function __invoke(InvocationEvent $event): void
    {
        // 支持 AssistantMessage 的模板渲染
        // 支持函数调用消息的模板
        // 支持多模态内容的模板
    }
}

// 创建变量预处理监听器
final class TemplateVarPreprocessor implements EventSubscriberInterface
{
    public function __invoke(InvocationEvent $event): void
    {
        $options = $event->getOptions();
        
        if (isset($options['template_vars'])) {
            // 添加全局变量
            $options['template_vars']['timestamp'] = time();
            $options['template_vars']['user'] = $this->getCurrentUser();
            
            $event->setOptions($options);
        }
    }
    
    public static function getSubscribedEvents(): array
    {
        return [
            InvocationEvent::class => ['__invoke', 100] // 更高的优先级
        ];
    }
}
```

## 与其他文件的关系

**依赖于**:
- `InvocationEvent`: 事件对象类
- `TemplateRendererRegistryInterface`: 渲染器注册表接口
- `NormalizerInterface`: Symfony Serializer 的规范化器接口
- `MessageBag`, `SystemMessage`, `UserMessage`: 消息相关类
- `Template`, `Text`: 消息内容类型
- `InvalidArgumentException`: 异常类

**被使用于**:
- EventDispatcher/EventSubscriber 系统
- AI 平台的请求处理管道
- 支持模板的 Model 实现

**相关组件**:
- `StringToMessageBagListener`: 另一个输入预处理监听器
- Twig/Blade 等模板引擎的 Bridge 实现

## 使用示例

```php
use Symfony\AI\Platform\EventListener\TemplateRendererListener;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\Message\SystemMessage;
use Symfony\AI\Platform\Message\UserMessage;
use Symfony\AI\Platform\Message\Template;

// 场景1: 基本模板渲染
$messages = new MessageBag(
    new SystemMessage(new Template('twig', 'You are a {{ role }} assistant.')),
    new UserMessage(
        new Template('twig', 'My name is {{ name }} and I need help with {{ topic }}.')
    )
);

$response = $model->generate($messages, [
    'template_vars' => [
        'role' => 'helpful',
        'name' => 'Alice',
        'topic' => 'PHP programming'
    ]
]);
// 渲染后:
// System: "You are a helpful assistant."
// User: "My name is Alice and I need help with PHP programming."

// 场景2: 对象参数的自动规范化
class User {
    public function __construct(
        public string $name,
        public string $email,
        public array $preferences
    ) {}
}

$user = new User('Bob', 'bob@example.com', ['language' => 'zh-CN']);

$messages = new MessageBag(
    new UserMessage(
        new Template('twig', 'User {{ user.name }} ({{ user.email }}) prefers {{ user.preferences.language }}')
    )
);

$response = $model->generate($messages, [
    'template_vars' => ['user' => $user],
    'template_options' => [
        'normalizer_context' => ['groups' => ['api']]
    ]
]);
// normalizer 会将 $user 对象转换为数组，然后模板可以访问其属性

// 场景3: 混合模板和静态内容
$messages = new MessageBag(
    new SystemMessage('You are a helpful assistant.'), // 静态内容
    new UserMessage(
        new Text('Hello! '),
        new Template('twig', 'My name is {{ name }}.'),
        new Text(' Nice to meet you!')
    )
);

$response = $model->generate($messages, [
    'template_vars' => ['name' => 'Charlie']
]);
// 渲染后: "Hello! My name is Charlie. Nice to meet you!"

// 场景4: 在 Symfony 应用中配置
// config/services.yaml
// services:
//     Symfony\AI\Platform\EventListener\TemplateRendererListener:
//         arguments:
//             $rendererRegistry: '@Symfony\AI\Platform\Message\TemplateRenderer\TemplateRendererRegistryInterface'
//             $normalizer: '@serializer'
//         tags:
//             - { name: kernel.event_subscriber }

// 场景5: 错误处理
try {
    $response = $model->generate($messages, [
        'template_vars' => [
            'complexObject' => new ComplexObject(),  // 需要 normalizer
            123 => 'invalid key'  // 无效的键
        ]
    ]);
} catch (InvalidArgumentException $e) {
    // 捕获模板变量验证错误
    error_log('Template variable error: ' . $e->getMessage());
}

// 场景6: 动态构建模板变量
class ChatbotService
{
    public function chat(string $userInput, array $context): Response
    {
        $templateVars = [
            'user_input' => $userInput,
            'date' => date('Y-m-d'),
            'time' => date('H:i:s'),
        ];
        
        // 添加上下文信息
        if (isset($context['user'])) {
            $templateVars['user'] = $context['user'];
        }
        
        if (isset($context['session'])) {
            $templateVars['session_id'] = $context['session']['id'];
        }
        
        $messages = $this->buildMessages();
        
        return $this->model->generate($messages, [
            'template_vars' => $templateVars,
            'template_options' => [
                'normalizer_context' => [
                    'groups' => ['chat'],
                    'datetime_format' => 'Y-m-d H:i:s'
                ]
            ]
        ]);
    }
}
```

TemplateRendererListener 提供了强大的模板渲染能力，使开发者能够使用熟悉的模板引擎（如 Twig）来构建 AI 提示词，支持变量替换、条件逻辑、循环等高级功能，同时通过对象规范化机制无缝集成复杂的数据结构，极大地提高了提示词的可维护性和复用性。
