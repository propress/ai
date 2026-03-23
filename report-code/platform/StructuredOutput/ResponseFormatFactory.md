# ResponseFormatFactory 分析报告

## 文件概述
`ResponseFormatFactory` 将 PHP 类名转换为 OpenAI `response_format` JSON Schema 格式定义，是结构化输出的 Schema 生成入口。

## 方法分析

### `create(string $responseClass): array`
```php
return [
    'type' => 'json_schema',
    'json_schema' => [
        'name' => u($responseClass)->afterLast('\\')->toString(), // 短类名
        'schema' => $this->schemaFactory->buildProperties($responseClass),
        'strict' => true, // 开启 additionalProperties: false
    ],
];
```
- 使用 `symfony/string` 的 `u()` 提取短类名（去除命名空间前缀）
- `strict: true` 与 `additionalProperties: false` 配合，要求模型严格遵守 Schema

## 设计模式
**工厂（Factory）**：封装 Schema 构建和格式包装，对 `PlatformSubscriber` 隐藏细节。

## 扩展点
实现 `ResponseFormatFactoryInterface` 替换此工厂，可以：
- 生成 Anthropic `tool_use` 格式（而非 `json_schema`）
- 添加缓存（避免重复构建 Schema）
- 自定义 Schema 名称格式

## 与其他文件的关系
- 实现 `ResponseFormatFactoryInterface`
- 依赖 `Contract\JsonSchema\Factory`
