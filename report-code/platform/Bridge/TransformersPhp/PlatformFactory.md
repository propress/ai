# PlatformFactory.php 分析报告

## 概述

`PlatformFactory` 创建 TransformersPhp Bridge 的 `Platform` 实例，在实例化前通过类存在性检查验证 `codewithkyrian/transformers` 依赖是否已安装，提供清晰的错误信息引导用户安装缺失依赖。

## 关键方法分析

### `create(ModelCatalogInterface $modelCatalog, ?EventDispatcherInterface $eventDispatcher): Platform`

**依赖检查**：通过 `class_exists(Transformers::class)` 验证 `codewithkyrian/transformers` 包是否可用，若未安装则抛出 `RuntimeException` 并附带安装命令提示：
```
composer require codewithkyrian/transformers
```

**平台构建**：
```php
return new Platform([new ModelClient()], [new ResultConverter()], $modelCatalog, eventDispatcher: $eventDispatcher);
```
- 无 API Key 参数（本地推理无需认证）
- 无 HTTP 客户端（无网络调用）
- 无 `Contract` 参数（无需序列化契约）

## 设计模式

- **守卫子句（Guard Clause）**：在方法开头立即检查前提条件，失败时早期返回/抛出
- **最简工厂**：方法签名最简洁（仅 2 个可选参数），体现本地推理无需凭证或 HTTP 配置的特点

## 与其他类的关系

- 相较于其他 Bridge 的工厂类，此类参数列表最短（无 apiKey、无 httpClient、无 Contract）
- 创建的 `Platform` 仅注册 `ModelClient` 和 `ResultConverter`，无任何 HTTP 相关组件
