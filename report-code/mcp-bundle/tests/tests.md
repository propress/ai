# 测试文件分析：tests/

## 测试概览

| 文件 | 类名 | 测试数 | 职责 |
|------|------|--------|------|
| `DependencyInjection/McpBundleTest.php` | `McpBundleTest` | 16 | 测试 Bundle 配置、服务注册、自动配置 |
| `DependencyInjection/McpPassTest.php` | `McpPassTest` | 4 | 测试编译器传递逻辑 |
| `Profiler/DataCollectorTest.php` | `DataCollectorTest` | 3 | 测试数据收集器 |
| `Profiler/TraceableRegistryTest.php` | `TraceableRegistryTest` | 1 | 测试可追踪注册中心的 reset |

## McpBundleTest

### 测试策略
使用 `buildContainer()` 辅助方法构建完整的 ContainerBuilder，模拟不同配置场景。

```php
private function buildContainer(array $configuration): ContainerBuilder
{
    $container = new ContainerBuilder();
    $container->setParameter('kernel.debug', true);
    $container->setParameter('kernel.environment', 'test');
    $container->setParameter('kernel.build_dir', 'public');
    $container->setParameter('kernel.project_dir', '/path/to/project');

    $extension = (new McpBundle())->getContainerExtension();
    $extension->load($configuration, $container);

    return $container;
}
```

**技巧：** 直接使用 `ContainerBuilder`（非编译后的容器），可以检查服务定义而非实际的服务实例。

### 测试用例详解

| 方法 | 测试目标 |
|------|----------|
| `testDefaultConfiguration` | 所有默认参数值正确 |
| `testDataCollectorTagIncludesId` | DataCollector 标签包含 `id => 'mcp'` |
| `testCustomConfiguration` | 自定义配置值正确传递 |
| `testClientTransportsConfiguration` | 四种传输组合下的服务注册状态（DataProvider） |
| `testServerServices` | Builder 的方法调用链完整 |
| `testMcpToolAttributeAutoconfiguration` | McpTool 属性自动配置注册 |
| `testMcpPromptAttributeAutoconfiguration` | McpPrompt 属性自动配置注册 |
| `testMcpResourceAttributeAutoconfiguration` | McpResource 属性自动配置注册 |
| `testMcpResourceTemplateAttributeAutoconfiguration` | McpResourceTemplate 属性自动配置注册 |
| `testHttpConfigurationDefaults` | HTTP 配置默认值（路径、会话存储） |
| `testHttpConfigurationCustom` | 自定义 HTTP 配置 |
| `testSessionStoreFileConfiguration` | 文件会话存储配置 |
| `testSessionStoreCacheConfigurationDefault` | 缓存会话存储默认配置 |
| `testSessionStoreCacheConfigurationCustom` | 缓存会话存储自定义配置 |
| `testDiscoveryDefaultConfiguration` | 发现配置默认值 |
| `testDiscoveryCustomConfiguration` | 自定义发现配置 |
| `testDiscoveryWithExcludeDirsOnly` | 仅排除目录配置 |
| `testLoaderInterfaceAutoconfiguration` | LoaderInterface 自动配置 |
| `testRequestHandlerInterfaceAutoconfiguration` | RequestHandlerInterface 自动配置 |
| `testNotificationHandlerInterfaceAutoconfiguration` | NotificationHandlerInterface 自动配置 |

### 关键断言模式

**1. 验证 Builder 方法调用链：**
```php
$methodCalls = $builderDefinition->getMethodCalls();
foreach ($methodCalls as $call) {
    if ('setEventDispatcher' === $call[0]) {
        $hasEventDispatcherCall = true;
    }
}
```

**2. 验证属性自动配置注册：**
```php
$attributeAutoconfigurators = $container->getAttributeAutoconfigurators();
$this->assertArrayHasKey(McpTool::class, $attributeAutoconfigurators);
```

**3. 验证会话存储类型：**
```php
$sessionStoreDefinition = $container->getDefinition('mcp.session.store');
$this->assertSame(InMemorySessionStore::class, $sessionStoreDefinition->getClass());
```

## McpPassTest

### 测试用例详解

| 方法 | 测试目标 |
|------|----------|
| `testCreatesServiceLocatorForAllMcpServices` | 四种标签的服务都被收集到 ServiceLocator |
| `testDoesNothingWhenNoMcpServicesTagged` | 无标记服务时不添加方法调用 |
| `testDoesNothingWhenNoServerBuilder` | 无 Builder 时不创建 ServiceLocator |
| `testHandlesPartialMcpServices` | 部分标签有服务时正确处理 |

### 关键断言模式

**验证 ServiceLocator 内容：**
```php
$serviceLocatorId = (string) $methodCalls[0][1][0];
$serviceLocatorDef = $container->getDefinition($serviceLocatorId);
$services = $serviceLocatorDef->getArgument(0);

$this->assertArrayHasKey('tool_service', $services);
$this->assertInstanceOf(ServiceClosureArgument::class, $services['tool_service']);
$this->assertInstanceOf(Reference::class, $services['tool_service']->getValues()[0]);
```

这验证了三层封装：ServiceLocator → ServiceClosureArgument → Reference。

## DataCollectorTest

### 测试用例详解

| 方法 | 测试目标 |
|------|----------|
| `testGetNameReturnsShortName` | getName() 返回 'mcp'（非完整类名） |
| `testLateCollectPopulatesData` | lateCollect() 后数据可访问 |
| `testGetTotalCountReturnsZeroForEmptyRegistry` | 空注册中心总数为 0 |

### 注意事项
- 使用真实的 `Registry` 实例（非 Mock），因为需要真实的数据结构
- `NullLogger` 用于满足 Registry 的日志依赖

## TraceableRegistryTest

### 测试用例详解

| 方法 | 测试目标 |
|------|----------|
| `testResetCallsClear` | reset() 方法委托到 clear() |

### 说明
只有一个测试，验证 `ResetInterface` 的核心行为。其他方法是纯委托（透传），无需单独测试。

## 测试覆盖分析

### 已覆盖
- ✅ 所有配置选项的默认值和自定义值
- ✅ 传输模式的条件注册
- ✅ 会话存储的三种策略
- ✅ 属性自动配置
- ✅ 接口自动配置
- ✅ 编译器传递的标签收集
- ✅ Profiler 数据收集
- ✅ ResetInterface 实现

### 未覆盖
- ❌ McpController 的 HTTP 请求处理（涉及 PSR-7 转换，需要集成测试）
- ❌ McpCommand 的 STDIO 运行（涉及进程 I/O，需要功能测试）
- ❌ RouteLoader 的路由注册（可单元测试但未覆盖）
- ❌ 模板渲染（通常通过 Profiler 功能测试覆盖）

### 测试设计模式
1. **构建器辅助**（`buildContainer()`）：封装容器构建的重复逻辑
2. **DataProvider**（`provideClientTransportsConfiguration`）：参数化测试传输组合
3. **单元隔离**：每个测试使用独立的 ContainerBuilder 实例
