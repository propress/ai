# AbstractModelCatalog 分析报告

## 文件概述
`AbstractModelCatalog` 是所有 Bridge 模型目录的基类，实现了模型名称解析（支持查询参数传递选项、`:` 变体匹配）和模型实例化逻辑。

## 类定义
- **类型**: `abstract class`，实现 `ModelCatalogInterface`

## 方法分析

### `getModel(string $modelName): Model`
完整流程：
1. 调用 `parseModelName()` 解析名称、查询参数选项、目录键
2. 在 `$this->models` 中查找，不存在抛出 `ModelNotFoundException`
3. 实例化模型类，检查是否继承 `Model`

### `parseModelName(string $modelName): array` *(protected)*
支持两种特殊语法：
- **查询参数**：`gpt-4o?temperature=0.7&max_tokens=100` → 解析为 `options` 数组
- **变体匹配**：`llama3.1:70b` → 若完整名称不存在，取 `:` 前的基础名 `llama3.1` 查找

### `convertScalarStrings(array $data): array` *(private, static)*
递归将查询参数字符串转为正确 PHP 类型：
- `'true'` → `true`，`'false'` → `false`
- `'42'` → `42`（int），`'3.14'` → `3.14`（float）

**技巧**：`parse_str()` 将查询字符串解析为数组，但所有值都是字符串，`convertScalarStrings` 负责类型恢复，使选项能正确传递给 HTTP 请求。

## 设计模式
**模板方法（Template Method）**：子类只需声明 `$this->models` 数组（模型注册表），所有查找/实例化/解析逻辑由基类提供。

## 与其他文件的关系
- 被所有 Bridge 的 ModelCatalog 继承（如 `OpenAi\ModelCatalog`）
- `$this->models` 格式：`array<string, array{class: class-string<Model>, capabilities: list<Capability>}>`
