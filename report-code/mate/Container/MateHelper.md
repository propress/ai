# MateHelper.php 深度分析报告

## 1. 文件概述

`MateHelper.php` 是一个静态工具类，为用户配置文件（`mate/config.php`）提供便捷的 API，允许用户在不直接操作容器参数的情况下精细控制 MCP 扩展的功能启用/禁用。

**文件路径**：`src/mate/src/Container/MateHelper.php`

---

## 2. 类签名与依赖

```php
namespace Symfony\AI\Mate\Container;

use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;

class MateHelper
```

- 非 `final` 类，可被继承
- 无属性、无构造函数
- 仅包含静态方法

---

## 3. 方法级别分析

### 3.1 `disableFeatures(ContainerConfigurator $container, array $extensions): void`

**输入**：
- `$container`：`ContainerConfigurator` —— Symfony 容器配置器
- `$extensions`：`array<string, list<string>>` —— 扩展名到待禁用功能名列表的映射

**输出**：`void`

**实现逻辑**：
1. 初始化空数组 `$data`
2. 遍历每个扩展及其功能列表
3. 为每个功能构建结构：`$data[$extension][$feature] = ['enabled' => false]`
4. 调用 `$container->parameters()->set('mate.disabled_features', $data)`

**使用示例**：
```php
// mate/config.php
return static function (ContainerConfigurator $container): void {
    MateHelper::disableFeatures($container, [
        'vendor/extension' => ['badTool', 'semiBadTool'],
        'nyholm/example' => ['clock'],
    ]);
};
```

**数据流向**：`MateHelper` → 容器参数 `mate.disabled_features` → `FilteredDiscoveryLoader::isFeatureAllowed()` 过滤

---

## 4. 设计模式分析

- **静态门面模式（Static Facade）**：提供简洁的静态 API 隐藏容器参数操作细节
- **配置 DSL**：将底层参数结构转化为语义化的禁用声明

---

## 5. 在模块中的调用场景

用户在 `mate/config.php` 中调用，由 `ContainerFactory::loadUserServices()` 触发加载。最终通过 `FilteredDiscoveryLoader` 在能力发现阶段过滤被禁用的功能。

---

## 6. 可扩展性分析

- 可通过继承添加更多静态辅助方法（如 `enableFeatures`、`configureExtension` 等）
- 当前仅支持禁用，不支持启用或条件配置
- 多次调用会覆盖（非合并），用户应在一次调用中列出所有禁用项

---

## 7. 技巧与最佳实践

- 使用语义化的方法名替代直接的参数设置，提升配置文件可读性
- 数据结构设计为三层嵌套（扩展→功能→配置），为未来扩展更多属性留有余地
- 与 `FilteredDiscoveryLoader` 的协作基于约定的参数名，保持了松耦合
