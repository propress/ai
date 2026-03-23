# MethodDescriber 分析报告

## 文件概述
`MethodDescriber` 专门处理"描述方法参数"的场景（Tool 的函数参数），通过解析 PHPDoc `@param` 注释提取参数说明，为函数调用工具的参数 Schema 添加 `description` 字段。

## 类定义
- **类型**: `final class`，实现 `ObjectDescriberInterface` + `PropertyDescriberInterface`

## 方法分析

### `describeObject(ObjectSubject $subject, ?array &$schema): iterable`
- 只处理 `ReflectionMethod` 类型的反射器（即方法主题，非类主题）
- 遍历方法的所有 `ReflectionParameter`，逐个 `yield` 为 `PropertySubject`

### `describeProperty(PropertySubject $subject, ?array &$schema): void`
- 只处理 `ReflectionParameter` 类型的属性（方法参数）
- 调用 `fromParameter()` 提取 PHPDoc 说明
- 若有说明则设置 `$schema['description']`

### `fromParameter(ReflectionParameter $parameter): string` *(private)*
- 从方法的 `getDocComment()` 中用正则匹配 `@param <type> $paramName <description>`
- 正则：`/@param\s+\S+\s+\$name\s+(.*)/`
- 只返回说明文本部分（去除类型前缀和参数名）

## 设计模式
**单一职责**：仅处理方法参数（Tool 场景），不干扰类属性描述。正则解析 PHPDoc 是"约定优于配置"思想：开发者写的 `@param` 注释直接成为 AI 可见的工具参数说明。

## 与其他文件的关系
- 被 `Describer` 在描述 Tool 方法时使用
- 与 `Factory::buildParameters()` 配合，将方法签名转为 Schema
