# ModelCatalog 目录分析报告

## 目录职责

`ModelCatalog/` 目录包含模型目录系统，负责管理和查询可用的 AI 模型。模型目录提供了从模型名称字符串到 `Model` 对象的映射。

**目录路径**: `src/platform/src/ModelCatalog/`

---

## 包含的文件清单

| 文件 | 说明 |
|------|------|
| `ModelCatalogInterface.php` | 模型目录接口 |
| `AbstractModelCatalog.php` | 抽象基类，提供通用实现 |
| `FallbackModelCatalog.php` | 回退目录，接受任何模型名 |

---

## 接口定义

```php
interface ModelCatalogInterface
{
    public function getModel(string $modelName): Model;
    public function getModels(): array;
}
```

---

## AbstractModelCatalog

提供标准的模型解析和查询实现。

**特性**:
- 支持查询参数解析 (`model-name?temperature=0.7`)
- 支持大小变体解析 (`model:23b`)
- 自动类型转换

```php
abstract class AbstractModelCatalog implements ModelCatalogInterface
{
    protected array $models;
    
    public function getModel(string $modelName): Model
    {
        $parsed = $this->parseModelName($modelName);
        // 创建 Model 实例
    }
    
    protected function parseModelName(string $modelName): array
    {
        // 解析 'model?param=value' 格式
    }
}
```

---

## FallbackModelCatalog

接受任何模型名称，赋予所有能力。

```php
class FallbackModelCatalog extends AbstractModelCatalog
{
    public function getModel(string $modelName): Model
    {
        return new Model(
            $modelName,
            Capability::cases(), // 所有能力
            $this->parseModelName($modelName)['options']
        );
    }
}
```

---

## 典型使用场景

### 场景1：创建自定义模型目录

```php
class OpenAIModelCatalog extends AbstractModelCatalog
{
    public function __construct()
    {
        $this->models = [
            'gpt-4' => [
                'class' => Model::class,
                'capabilities' => [
                    Capability::INPUT_MESSAGES,
                    Capability::OUTPUT_TEXT,
                    Capability::OUTPUT_STREAMING,
                    Capability::OUTPUT_STRUCTURED,
                    Capability::TOOL_CALLING,
                ],
            ],
            'gpt-4-vision' => [
                'class' => Model::class,
                'capabilities' => [
                    Capability::INPUT_MESSAGES,
                    Capability::INPUT_IMAGE,
                    Capability::OUTPUT_TEXT,
                ],
            ],
        ];
    }
}
```

### 场景2：带参数的模型名

```php
// 在模型名中传递参数
$result = $platform->invoke(
    'gpt-4?temperature=0.7&max_tokens=500',
    $messages
);

// 等同于
$result = $platform->invoke('gpt-4', $messages, [
    'temperature' => 0.7,
    'max_tokens' => 500,
]);
```

### 场景3：模型大小变体

```php
// llama-3.1:70b 会查找 llama-3.1 的目录条目
$result = $platform->invoke('llama-3.1:70b', $messages);
```
