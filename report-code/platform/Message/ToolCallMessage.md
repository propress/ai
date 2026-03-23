# ToolCallMessage 分析报告

## 文件概述
ToolCallMessage 表示工具调用的结果消息，是 AI function calling 功能的重要组成部分。当 AI 助手请求调用某个工具（函数）时，应用程序执行该工具后，使用 ToolCallMessage 将执行结果返回给 AI 模型。这种消息类型使 AI 能够通过工具调用获取外部信息或执行操作，然后基于结果生成最终回复。

## 类/接口定义

### ToolCallMessage
- **类型**: final class（最终类）
- **继承/实现**: 实现 `MessageInterface` 接口
- **命名空间**: `Symfony\AI\Platform\Message`
- **职责**: 封装工具调用的结果，将执行信息传递回 AI 模型以继续对话

## 属性分析

### $toolCall
- **类型**: `ToolCall`
- **可见性**: private readonly
- **说明**: 原始的工具调用对象，包含工具 ID、名称和参数信息

### $content
- **类型**: `string`
- **可见性**: private readonly
- **说明**: 工具执行的结果内容，通常是 JSON 字符串或纯文本

## 方法分析

### __construct()
- **可见性**: public
- **参数**: 
  - `$toolCall` (`ToolCall`): 原始的工具调用对象
  - `$content` (`string`): 工具执行结果
- **返回值**: 无
- **功能说明**: 构造工具调用消息，将工具调用信息和执行结果关联起来。自动生成 UUID v7 作为消息 ID。两个参数都是必需的且不可变。
- **注意事项**: toolCall 对象必须与 AI 发起的工具调用匹配（特别是 ID）

### getRole()
- **可见性**: public
- **参数**: 无
- **返回值**: `Role` - 返回 `Role::ToolCall` 枚举值
- **功能说明**: 标识此消息为工具调用结果角色，用于消息序列化和 API 通信。实现 MessageInterface 接口要求。
- **注意事项**: 始终返回固定的 ToolCall 角色

### getToolCall()
- **可见性**: public
- **参数**: 无
- **返回值**: `ToolCall` - 原始工具调用对象
- **功能说明**: 获取关联的工具调用信息，包括工具 ID、名称和参数。这些信息用于将结果与原始请求关联。
- **注意事项**: ToolCall 对象包含工具的完整上下文信息

### getContent()
- **可见性**: public
- **参数**: 无
- **返回值**: `string` - 工具执行结果
- **功能说明**: 获取工具执行的结果内容。通常是结构化数据的字符串表示（JSON）或纯文本描述。实现 MessageInterface 接口，但返回类型更具体。
- **注意事项**: 内容应该清晰描述工具执行的结果，AI 会基于此生成最终回复

## 设计模式

### 1. Trait 组合模式
使用 `IdentifierAwareTrait` 和 `MetadataAwareTrait` 复用基础功能，保持类的简洁和专注。

### 2. 不可变对象（Immutable Object）
所有属性都是 readonly，确保消息一旦创建就不会被意外修改。这对于调试和追踪工具调用流程非常重要。

### 3. 值对象（Value Object）
ToolCallMessage 是一个纯数据载体，没有业务逻辑，只负责携带工具调用的结果信息。

## 扩展点

### 添加执行元数据
```php
$message = new ToolCallMessage($toolCall, $result);
$message->getMetadata()->set('execution_time_ms', 150);
$message->getMetadata()->set('success', true);
$message->getMetadata()->set('cache_hit', false);
```

### 错误处理
```php
// 工具执行失败时
$errorMessage = new ToolCallMessage(
    $toolCall,
    json_encode(['error' => 'API rate limit exceeded', 'code' => 429])
);
$errorMessage->getMetadata()->set('error', true);
```

## 与其他文件的关系

### 依赖关系
- **MessageInterface**: 实现的核心接口
- **IdentifierAwareTrait**: ID 管理功能
- **MetadataAwareTrait**: 元数据管理功能
- **ToolCall**: 工具调用数据结构（来自 Result 命名空间）
- **Role**: 角色枚举
- **Uuid**: Symfony UID 组件

### 被依赖关系
- **工具调用执行器**: 创建 ToolCallMessage 返回结果
- **对话管理器**: 将 ToolCallMessage 添加到对话历史
- **消息序列化器**: 将 ToolCallMessage 转换为 API 格式
- **Function Calling 工作流**: 完整的工具调用循环

### 典型工作流
1. **AssistantMessage** 包含 ToolCall 请求
2. 应用程序执行工具
3. 创建 **ToolCallMessage** 返回结果
4. 将 ToolCallMessage 加入对话
5. AI 基于结果生成新的 **AssistantMessage**

## 使用示例

```php
use Symfony\AI\Platform\Message\ToolCallMessage;
use Symfony\AI\Platform\Result\ToolCall;
use Symfony\AI\Platform\Message\AssistantMessage;

// 1. AI 请求工具调用（来自 AssistantMessage）
$assistantMessage = new AssistantMessage(
    content: null,
    toolCalls: [
        new ToolCall(
            id: 'call_abc123',
            name: 'get_weather',
            arguments: ['city' => 'Beijing', 'unit' => 'celsius']
        )
    ]
);

// 2. 执行工具
$toolCall = $assistantMessage->getToolCalls()[0];
$weatherData = executeWeatherTool($toolCall->arguments);

// 3. 创建工具调用结果消息
$toolCallMessage = new ToolCallMessage(
    toolCall: $toolCall,
    content: json_encode([
        'temperature' => 22,
        'condition' => 'sunny',
        'humidity' => 45
    ])
);

// 4. 添加元数据
$toolCallMessage->getMetadata()->set('execution_time_ms', 245);
$toolCallMessage->getMetadata()->set('api_source', 'weather_api_v2');

// 5. 获取消息信息
$role = $toolCallMessage->getRole(); // Role::ToolCall
$content = $toolCallMessage->getContent(); // JSON 结果
$originalToolCall = $toolCallMessage->getToolCall();

// 6. 完整的工具调用流程
$conversation = [
    new SystemMessage('You are a helpful assistant.'),
    new UserMessage(new Text('What is the weather in Beijing?')),
    // AI 请求工具调用
    new AssistantMessage(
        toolCalls: [
            new ToolCall(
                id: 'call_123',
                name: 'get_weather',
                arguments: ['city' => 'Beijing']
            )
        ]
    ),
    // 返回工具结果
    new ToolCallMessage(
        new ToolCall(
            id: 'call_123',
            name: 'get_weather',
            arguments: ['city' => 'Beijing']
        ),
        json_encode(['temperature' => 22, 'condition' => 'sunny'])
    ),
    // AI 基于结果生成回复
    // new AssistantMessage('北京现在天气晴朗，温度 22°C。')
];

// 7. 错误情况处理
$errorToolCall = new ToolCall(
    id: 'call_456',
    name: 'database_query',
    arguments: ['query' => 'SELECT * FROM users']
);

try {
    $result = executeDatabaseQuery($errorToolCall->arguments);
    $resultMessage = new ToolCallMessage($errorToolCall, $result);
} catch (\Exception $e) {
    $errorMessage = new ToolCallMessage(
        $errorToolCall,
        json_encode([
            'error' => $e->getMessage(),
            'type' => 'DatabaseException'
        ])
    );
    $errorMessage->getMetadata()->set('error', true);
}
```
