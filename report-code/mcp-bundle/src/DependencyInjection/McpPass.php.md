# 文件分析报告：src/DependencyInjection/McpPass.php

## 文件基本信息

| 属性 | 值 |
|------|-----|
| 文件路径 | `src/mcp-bundle/src/DependencyInjection/McpPass.php` |
| 命名空间 | `Symfony\AI\McpBundle\DependencyInjection` |
| 类名 | `McpPass` |
| 实现接口 | `CompilerPassInterface` |
| 使用 Trait | `PriorityTaggedServiceTrait` |
| 类型 | `final class` |
| 职责 | 收集所有标记的 MCP 服务并创建 ServiceLocator 注入到 Builder |

## 类签名

```php
final class McpPass implements CompilerPassInterface
{
    use PriorityTaggedServiceTrait;

    public function process(ContainerBuilder $container): void
}
```

## 设计模式

### 1. Compiler Pass 模式（Symfony DI 编译阶段处理）
这是 Symfony DependencyInjection 组件的核心扩展点。Compiler Pass 在容器编译阶段执行，允许对已注册的服务定义进行修改。

**好处：**
- 在所有服务定义注册完成后执行，可以获取完整的服务集合
- 可以跨 Bundle 收集服务（任何 Bundle 注册的 MCP 服务都会被收集）
- 在编译时完成服务组装，运行时无额外开销

### 2. Service Locator 模式
```php
$serviceLocatorRef = ServiceLocatorTagPass::register($container, $serviceReferences);
$container->getDefinition('mcp.server.builder')->addMethodCall('setContainer', [$serviceLocatorRef]);
```

使用 Symfony 的 `ServiceLocatorTagPass` 创建一个 **Service Locator**（服务定位器），只包含 MCP 相关的服务。

**好处：**
- 比注入完整容器更安全（只暴露需要的服务）
- 支持延迟加载（服务在首次使用时才实例化）
- Service Locator 是 Symfony 推荐的"受控"服务定位模式

### 3. 标签收集模式（Tag Collection Pattern）
```php
$mcpTags = ['mcp.tool', 'mcp.prompt', 'mcp.resource', 'mcp.resource_template'];
foreach ($mcpTags as $tag) {
    $taggedServices = $container->findTaggedServiceIds($tag);
    $allMcpServices = array_merge($allMcpServices, $taggedServices);
}
```

遍历所有 MCP 标签，收集所有标记的服务。

**好处：**
- 开发者只需给服务打标签（或使用属性自动打标签），无需显式注册
- 支持四种类型的 MCP 能力
- 扩展性极强：添加新的 MCP 服务只需打正确的标签

## 方法详细分析

### `process(ContainerBuilder $container): void`

**输入：**
- `$container`：Symfony 容器构建器，包含所有已注册的服务定义

**输出：**
- 无直接返回值（副作用：修改 `mcp.server.builder` 的服务定义）

**逻辑流程：**
```
process(ContainerBuilder $container)
│
├─ 步骤 1: 检查前置条件
│   └─ mcp.server.builder 服务是否存在？
│       ├─ 否 → 直接返回（Bundle 未完全配置）
│       └─ 是 → 继续
│
├─ 步骤 2: 收集所有 MCP 标签服务
│   ├─ findTaggedServiceIds('mcp.tool')
│   ├─ findTaggedServiceIds('mcp.prompt')
│   ├─ findTaggedServiceIds('mcp.resource')
│   └─ findTaggedServiceIds('mcp.resource_template')
│   └─ 合并到 $allMcpServices
│
├─ 步骤 3: 检查是否有 MCP 服务
│   └─ $allMcpServices 为空？
│       ├─ 是 → 直接返回（无需创建 Service Locator）
│       └─ 否 → 继续
│
├─ 步骤 4: 创建服务引用映射
│   └─ 为每个服务 ID 创建 Reference 对象
│       └─ $serviceReferences[$serviceId] = new Reference($serviceId)
│
├─ 步骤 5: 注册 Service Locator
│   └─ ServiceLocatorTagPass::register($container, $serviceReferences)
│       └─ 创建一个 ServiceLocator 服务定义
│       └─ 返回对 ServiceLocator 的引用
│
└─ 步骤 6: 注入到 Builder
    └─ mcp.server.builder->addMethodCall('setContainer', [$serviceLocatorRef])
        └─ Builder 在构建 Server 时使用此 ServiceLocator 解析服务
```

## Service Locator 的作用

Builder 使用 Service Locator 来解析 MCP 能力的处理器（handler）。当 MCP SDK 的自动发现机制扫描到带有 `#[McpTool]`、`#[McpPrompt]` 等属性的类/方法时，会使用 Service Locator 来获取这些类的实例。

```
Builder::build()
├─ 自动发现扫描文件系统
│   └─ 找到带 MCP 属性的类
├─ 注册到 Registry
│   └─ handler 可能是 string（服务 ID）
└─ 当需要执行 handler 时
    └─ ServiceLocator->get($serviceId) → 获取实例
```

## 技巧分析

### 1. 防御性前置条件检查
```php
if (!$container->hasDefinition('mcp.server.builder')) {
    return;
}
```
**为什么这么做：**
- Compiler Pass 在所有 Bundle 加载完成后执行
- 如果 McpBundle 因某种原因未完全配置（如传输都未启用），`mcp.server.builder` 可能不存在
- 安全地跳过处理，避免错误

### 2. 空服务集合快速返回
```php
if ([] === $allMcpServices) {
    return;
}
```
**为什么这么做：**
- 如果没有任何 MCP 服务被标记，不需要创建 Service Locator
- 避免创建空的 Service Locator 浪费资源
- 使用 `[] === $allMcpServices` 而非 `empty($allMcpServices)` 符合项目编码规范

### 3. 使用 `ServiceLocatorTagPass::register()` 而非手动创建
```php
$serviceLocatorRef = ServiceLocatorTagPass::register($container, $serviceReferences);
```
**为什么这么做：**
- `ServiceLocatorTagPass::register()` 是 Symfony 提供的标准工具
- 自动处理 ServiceLocator 的注册、命名和优化
- 每个 Reference 会被自动包装为 `ServiceClosureArgument`（延迟加载）

### 4. `PriorityTaggedServiceTrait` 的引入
虽然当前代码中没有直接使用 `PriorityTaggedServiceTrait` 的方法，但引入它表明：
- 未来可能需要按优先级排序 MCP 服务
- 或者是历史遗留，保留以备后用

## 被调用场景

1. **`McpBundle::build()`** → `$container->addCompilerPass(new McpPass())`
2. Symfony 容器编译阶段自动调用 `process()` 方法
3. 在所有 Bundle 的 `loadExtension()` 执行完毕后

## 与 MCP SDK 的关系

McpPass 创建的 Service Locator 通过 `Builder::setContainer()` 注入到 MCP SDK 中。SDK 的内部使用 PSR-11 `ContainerInterface` 来解析服务：

```php
// MCP SDK Builder 内部
public function setContainer(ContainerInterface $container): self
{
    $this->container = $container;
    return $this;
}
```

这使得 MCP SDK 可以使用 Symfony 的 DI 容器来实例化和管理 MCP 能力的处理器。

## 可扩展性

### 可扩展
- 添加新的 MCP 服务类型只需定义新的标签，并在此 Pass 中添加收集逻辑
- 自定义 Compiler Pass 可以在 McpPass 之前或之后执行，修改 MCP 服务定义

### 不可扩展
- Service Locator 的创建方式是固定的
- 注入到 Builder 的方法名 `setContainer` 是固定的

### 可能的扩展场景
- 添加 MCP 服务的验证逻辑（检查 handler 签名是否正确）
- 添加 MCP 服务的优先级排序
- 添加基于条件的服务过滤（如环境相关的服务）
