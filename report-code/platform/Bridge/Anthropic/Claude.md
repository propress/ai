# Anthropic/Claude 分析报告

## 文件概述
`Claude` 是所有 Claude 系列模型的标记子类，提供版本常量和默认 `max_tokens` 注入。

## 模型版本常量
```php
const HAIKU_3 = 'claude-3-haiku-20240307';
const HAIKU_35 = 'claude-3-5-haiku-latest';
const SONNET_3 = 'claude-3-sonnet-20240229';
const SONNET_35 = 'claude-3-5-sonnet-latest';
const SONNET_37 = 'claude-3-7-sonnet-latest';
const SONNET_4 = 'claude-sonnet-4-20250514';
const OPUS_3 = 'claude-3-opus-20240229';
const OPUS_4 = 'claude-opus-4-20250514';
// ...
```

## 默认 max_tokens
```php
if (!isset($options['max_tokens'])) {
    $options['max_tokens'] = 1000;
}
```
Anthropic API 要求 `max_tokens` 为**必填**字段（OpenAI 则可选），Claude 模型在此统一注入默认值 1000，防止漏传导致 API 报错。

## 与其他文件的关系
- 所有 Anthropic Contract Normalizer 检查 `instanceof Claude`
- `ModelCatalog` 为所有 Claude 模型注册此类
