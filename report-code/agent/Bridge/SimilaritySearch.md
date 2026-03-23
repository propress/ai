# Bridge/SimilaritySearch/SimilaritySearch.php 分析报告

## 概述

`SimilaritySearch` 注册为 `similarity_search` 工具，将搜索词向量化后在向量存储中执行语义相似度搜索，返回匹配文档的元数据 JSON 字符串。公开 `$usedDocuments` 属性供外部访问最近检索结果。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Toolbox/Attribute/AsTool` | 工具注册注解 |
| `Store/VectorizerInterface` + `StoreInterface` | 向量化与检索（来自 Store 模块）|
