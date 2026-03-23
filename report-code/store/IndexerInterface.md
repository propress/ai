# IndexerInterface.php 分析报告

## 概述

`IndexerInterface` 定义文档处理管道的统一入口，负责将原始输入（来源标识符、文档对象或迭代器）经过完整流程（加载→过滤→转换→向量化→存储）写入向量数据库。

## 关键方法分析

| 方法 | 说明 |
|------|------|
| `index(string\|iterable\|object $input, array $options = []): void` | 接受灵活输入类型，触发完整索引流程；`options` 支持 `chunk_size`、`platform_options` 等参数 |

## 设计模式

- **命令模式**：`index()` 封装了完整的文档处理流程，调用方无需了解内部细节
- **策略模式**：不同实现（`DocumentIndexer`、`SourceIndexer`、`ConfiguredSourceIndexer`）针对不同输入来源

## 关联关系

- `DocumentIndexer`：直接接受 `EmbeddableDocumentInterface` 或其迭代器
- `SourceIndexer`：接受来源字符串，委托 `LoaderInterface` 加载文档
- `ConfiguredSourceIndexer`：带预配置默认来源的装饰器
- `IndexCommand` 通过 `ServiceLocator` 发现并调用实现类
