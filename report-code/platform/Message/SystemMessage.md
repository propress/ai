# SystemMessage 分析报告

## 文件概述
SystemMessage 表示系统级别的提示消息，用于设定 AI 助手的行为准则、角色定位和上下文信息。它是对话的"指令层"，告诉 AI 模型应该如何回应用户，扮演什么角色，遵循什么规则。SystemMessage 通常在对话开始时发送，并在整个会话中持续影响 AI 的行为。该类支持纯文本和模板两种内容形式。

## 类/接口定义

### SystemMessage
- **类型**: final class（最终类）
- **继承/实现**: 实现 `MessageInterface` 接口
- **命名空间**: `Symfony\AI\Platform\Message`
- **职责**: 封装系统级别的提示指令，定义 AI 助手的行为和上下文

## 属性分析

### $content
- **类型**: `string|Template`
- **可见性**: private readonly
- **说明**: 系统消息内容，可以是纯文本字符串或可变参数模板。使用 readonly 确保内容不可变。

## 方法分析

### __construct()
- **可见性**: public
- **参数**: 
  - `$content` (`string|Template`): 系统提示内容，支持字符串或模板
- **返回值**: 无
- **功能说明**: 构造系统消息对象，初始化内容并自动生成 UUID v7 作为消息 ID。内容使用 readonly 修饰符确保构造后不可修改。
- **注意事项**: content 参数必需，系统消息不能为空

### getRole()
- **可见性**: public
- **参数**: 无
- **返回值**: `Role` - 返回 `Role::System` 枚举值
- **功能说明**: 标识此消息为系统角色，用于消息路由和序列化。实现 MessageInterface 接口要求。
- **注意事项**: 始终返回固定的 System 角色

### getContent()
- **可见性**: public
- **参数**: 无
- **返回值**: `string|Template` - 系统提示内容
- **功能说明**: 获取系统消息的内容。可能是纯文本字符串（直接使用）或 Template 对象（需要渲染后使用）。实现 MessageInterface 接口，但返回类型更具体。
- **注意事项**: 如果返回 Template 对象，调用者需要使用 TemplateRenderer 进行变量替换

## 设计模式

### 1. Trait 组合模式
通过 `IdentifierAwareTrait` 和 `MetadataAwareTrait` 复用 ID 和元数据管理代码，保持类本身的简洁性。

### 2. 不可变对象（Immutable Object）
使用 `readonly` 修饰符确保内容在构造后不可更改，结合 IdentifierAwareTrait 的 `withId()` 方法实现完全的不可变性。这种设计提供：
- 线程安全
- 可预测的行为
- 适合函数式编程风格

### 3. 联合类型（Union Types）
内容支持 `string|Template` 两种类型，提供灵活性：
- string: 简单场景，直接使用
- Template: 复杂场景，支持变量替换和动态生成

## 扩展点

### 使用字符串内容
```php
$message = new SystemMessage(
    'You are a helpful assistant. Always be polite and professional.'
);
```

### 使用模板内容
```php
$template = Template::string(
    'You are a {role} assistant specialized in {domain}. ' .
    'Your expertise level is {level}.'
);
$message = new SystemMessage($template);
```

### 添加元数据
```php
$message = new SystemMessage('You are a helpful assistant.');
$message->getMetadata()->set('version', '1.0');
$message->getMetadata()->set('language', 'zh-CN');
```

## 与其他文件的关系

### 依赖关系
- **MessageInterface**: 实现的核心接口
- **IdentifierAwareTrait**: ID 管理功能
- **MetadataAwareTrait**: 元数据管理功能
- **Template**: 模板内容支持
- **Role**: 角色枚举
- **Uuid**: Symfony UID 组件

### 被依赖关系
- **对话管理器**: 在会话开始时设置 SystemMessage
- **提示词工程工具**: 动态生成和组合系统提示
- **模板渲染器**: 处理 Template 类型的内容
- **消息序列化器**: 将 SystemMessage 转换为 API 请求格式

## 使用示例

```php
use Symfony\AI\Platform\Message\SystemMessage;
use Symfony\AI\Platform\Message\Template;

// 1. 简单的系统提示
$simpleMessage = new SystemMessage(
    'You are a helpful AI assistant. Respond in Chinese.'
);

// 2. 角色扮演提示
$roleMessage = new SystemMessage(
    'You are a professional Python developer with 10 years of experience. ' .
    'You provide clean, well-documented code with best practices.'
);

// 3. 使用字符串模板
$stringTemplate = Template::string(
    'You are a {profession} assistant. ' .
    'Your task is to help with {task}. ' .
    'User expertise level: {level}'
);
$templateMessage = new SystemMessage($stringTemplate);

// 4. 使用表达式模板
$expressionTemplate = Template::expression(
    '"You are a " ~ role ~ " assistant. Today is " ~ date'
);
$advancedMessage = new SystemMessage($expressionTemplate);

// 5. 带元数据的系统消息
$message = new SystemMessage('You are helpful.');
$message->getMetadata()->set('category', 'general');
$message->getMetadata()->set('priority', 'high');

// 使用消息
$role = $message->getRole(); // Role::System
$content = $message->getContent();

// 处理不同内容类型
if ($content instanceof Template) {
    // 需要渲染
    $rendered = $renderer->render($content, [
        'profession' => 'software engineer',
        'task' => 'code review',
        'level' => 'intermediate'
    ]);
    echo $rendered;
} else {
    // 直接使用字符串
    echo $content;
}

// 组合使用（典型对话场景）
use Symfony\AI\Platform\Message\UserMessage;
use Symfony\AI\Platform\Message\Content\Text;

$conversation = [
    new SystemMessage('You are a friendly chatbot.'),
    new UserMessage(new Text('Hello!')),
    // ... AI 会根据系统消息的指示回应
];
```
