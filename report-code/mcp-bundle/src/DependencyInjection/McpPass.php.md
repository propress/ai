# 文件分析：src/DependencyInjection/McpPass.php

## 文件基本信息

| 属性 | 值 |
|------|-----|
| 文件路径 | `src/mcp-bundle/src/DependencyInjection/McpPass.php` |
| 完整类名 | `Symfony\AI\McpBundle\DependencyInjection\McpPass` |
| 实现接口 | `Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface` |
| 使用 Trait | `PriorityTaggedServiceTrait` |
| 修饰符 | `final` |
| 行数 | 51 行 |
| 职责 | 将所有 MCP 能力服务收集到 ServiceLocator 并注入 Builder |

## 功能概述

`McpPass` 是 Symfony DI 容器的 **编译器传递（Compiler Pass）**，它在容器编译阶段运行，负责：
1. 查找容器中所有标记为 `mcp.tool`、`mcp.prompt`、`mcp.resource`、`mcp.resource_template` 标签的服务
2. 将这些服务打包成一个 ServiceLocator（惰性服务定位器）
3. 通过 `setContainer()` 方法将 ServiceLocator 注入到 `mcp.server.builder`

## 设计模式

### 1. 编译器传递模式（Compiler Pass Pattern）
Symfony DI 容器的核心扩展机制。在容器冻结（freeze/compile）之前，CompilerPass 可以检查和修改所有已注册的服务定义。

**为什么这么做：**
- 需要在编译时收集分散在整个应用中的 MCP 能力类（可能在不同 Bundle、不同目录下）
- 普通的服务定义无法实现"收集所有带某标签的服务"这种动态聚合逻辑
- 编译器传递在容器冻结前运行，修改后的定义会被优化和缓存

### 2. 服务定位器模式（Service Locator Pattern）
```php
$serviceLocatorRef = ServiceLocatorTagPass::register($container, $serviceReferences);
$container->getDefinition('mcp.server.builder')->addMethodCall('setContainer', [$serviceLocatorRef]);
```

使用 `ServiceLocatorTagPass::register()` 创建一个惰性服务定位器，而不是直接将所有服务注入。

**为什么这么做（关键技巧）：**
- **惰性加载**：ServiceLocator 中的服务只在被真正获取时才会实例化（通过 ServiceClosureArgument 包装）
- **内存优化**：即使注册了 100 个 Tool，如果某次请求只调用了 1 个，只有那 1 个会被实例化
- **编译优化**：Symfony 会将 ServiceLocator 优化为专用的容器类方法
- **安全边界**：ServiceLocator 限制了可访问的服务范围，不会暴露整个容器

### 3. 标签收集模式（Tag Collection Pattern）
```php
$mcpTags = ['mcp.tool', 'mcp.prompt', 'mcp.resource', 'mcp.resource_template'];
foreach ($mcpTags as $tag) {
    $taggedServices = $container->findTaggedServiceIds($tag);
    $allMcpServices = array_merge($allMcpServices, $taggedServices);
}
```

**为什么这么做：**
- 统一管理四种不同类型的 MCP 能力服务
- 一次性处理所有标签，避免为每种标签写一个 CompilerPass

## 方法详细分析

### process(ContainerBuilder $container): void

**输入：** `ContainerBuilder $container` — Symfony DI 容器构建器

**输出：** 无返回值（通过修改容器定义产生副作用）

**逻辑流程：**

```
1. 检查 mcp.server.builder 是否存在
   │ 不存在 → return（可能传输未启用，此时无需处理）
   ▼
2. 遍历四种 MCP 标签，收集所有标记的服务 ID
   │ mcp.tool → [tool_service1, tool_service2, ...]
   │ mcp.prompt → [prompt_service1, ...]
   │ mcp.resource → [resource_service1, ...]
   │ mcp.resource_template → [template_service1, ...]
   ▼
3. 检查是否有任何 MCP 服务
   │ 没有 → return
   ▼
4. 为每个服务创建 Reference
   │ ['tool_service1' => new Reference('tool_service1'), ...]
   ▼
5. 使用 ServiceLocatorTagPass::register() 创建 ServiceLocator
   │ → 返回 ServiceLocator 的 Reference
   ▼
6. 给 mcp.server.builder 添加 setContainer() 方法调用
   │ $builder->setContainer($serviceLocator)
   ▼
完成
```

**防御性编程：**
1. 第 1 步检查 `mcp.server.builder` 存在性——如果用户没有启用任何传输，services.php 不会注册这些服务
2. 第 3 步检查是否有 MCP 服务——如果应用中没有定义任何 Tool/Prompt/Resource，则无需注入空的 ServiceLocator

## PriorityTaggedServiceTrait

该 Pass 引入了 `PriorityTaggedServiceTrait`，虽然在当前代码中没有直接使用优先级排序功能，但该 Trait 提供了 `findAndSortTaggedServices()` 方法。这是预留的扩展能力——如果未来 MCP 能力需要按优先级排序（例如 Tool 的展示顺序），只需简单修改收集逻辑。

## 调用流程

```
Symfony 容器编译过程
    │
    ├── McpBundle::build($container)
    │       └── $container->addCompilerPass(new McpPass())
    │
    ├── ... 其他 Bundle 注册服务定义 ...
    │
    ├── 用户代码中带有 MCP 属性的类被自动配置:
    │   #[McpTool] → 自动添加 mcp.tool 标签
    │   #[McpPrompt] → 自动添加 mcp.prompt 标签
    │   #[McpResource] → 自动添加 mcp.resource 标签
    │   #[McpResourceTemplate] → 自动添加 mcp.resource_template 标签
    │
    ├── 容器编译阶段执行 CompilerPass:
    │       └── McpPass::process($container)
    │           ├── 收集所有 mcp.* 标签的服务
    │           ├── 创建 ServiceLocator
    │           └── 注入到 mcp.server.builder
    │
    └── 容器冻结（dump 到 PHP 缓存文件）
```

## 在哪些场景下被调用

1. **首次请求（缓存未热）**：容器需要编译
2. **`bin/console cache:clear`**：清除缓存后重新编译
3. **`bin/console cache:warmup`**：预热缓存时编译
4. **开发环境**：每次代码变更后自动重新编译

## 可替换/可扩展点

1. **自定义 CompilerPass**：可以注册额外的 CompilerPass 来修改已标记服务的属性（如添加优先级、条件过滤等）
2. **标签扩展**：如果 MCP SDK 新增了能力类型（如 `McpSampling`），只需在 `$mcpTags` 数组中添加新标签，并在 `McpBundle` 中注册对应的属性自动配置
3. **ServiceLocator 替换**：可以通过覆盖 `mcp.server.builder` 的 `setContainer` 调用来注入自定义的服务定位器

## 外部知识

### Symfony Compiler Pass
- 编译器传递是 Symfony DI 容器最强大的扩展点
- 在 `ContainerBuilder::compile()` 阶段按优先级顺序执行
- 可以修改任何服务定义，甚至删除或替换服务
- 文档：https://symfony.com/doc/current/service_container/compiler_passes.html

### Symfony ServiceLocator
- 轻量级的依赖注入容器，只包含指定的服务
- 实现了 PSR-11 `ContainerInterface`
- 每个服务被 `ServiceClosureArgument` 包装，确保惰性加载
- 优于直接注入容器（反模式）——只暴露需要的服务
- 文档：https://symfony.com/doc/current/service_container/service_locators.html

### MCP SDK Builder::setContainer()
- SDK 的 Builder 在实例化能力 handler 时使用这个 ServiceLocator
- 当 MCP 客户端请求调用某个 Tool 时，Builder 通过 ServiceLocator 获取对应的服务实例
- 这实现了真正的按需加载——只有被调用的 Tool 才会被实例化
