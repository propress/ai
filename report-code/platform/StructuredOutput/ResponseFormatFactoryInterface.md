# ResponseFormatFactoryInterface 分析报告

## 文件概述
定义响应格式工厂的接口，规定输出格式为 OpenAI `json_schema` 结构。

## 接口定义
```php
interface ResponseFormatFactoryInterface {
    public function create(string $responseClass): array; // {type, json_schema: {name, schema, strict}}
}
```

## 扩展点
这是结构化输出最重要的扩展点之一：不同 AI 提供商要求不同的结构化输出格式，可通过实现此接口并注入 `PlatformSubscriber` 来支持特定提供商的格式。

## 与其他文件的关系
- 被 `PlatformSubscriber` 依赖注入
- 实现类：`ResponseFormatFactory`
