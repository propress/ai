# 目录分析报告：tests/

## 目录基本信息

| 属性 | 值 |
|------|-----|
| 路径 | `src/mcp-bundle/tests/` |
| 文件数量 | 4 |
| 测试框架 | PHPUnit 11+ |
| 职责 | 验证 MCP Bundle 的依赖注入配置和 Profiler 组件功能 |

## 目录内容

| 子目录 | 文件 | 测试类 | 覆盖目标 |
|--------|------|--------|----------|
| `DependencyInjection/` | `McpBundleTest.php` | `McpBundleTest` | `McpBundle` 的完整配置和服务注册 |
| `DependencyInjection/` | `McpPassTest.php` | `McpPassTest` | `McpPass` Compiler Pass 的服务收集逻辑 |
| `Profiler/` | `DataCollectorTest.php` | `DataCollectorTest` | `DataCollector` 的数据收集和读取 |
| `Profiler/` | `TraceableRegistryTest.php` | `TraceableRegistryTest` | `TraceableRegistry` 的 reset 行为 |

## 测试覆盖分析

### McpBundleTest（最全面的测试，~530 行）

| 测试方法 | 验证内容 |
|----------|----------|
| `testDefaultConfiguration` | 所有配置参数的默认值 |
| `testDataCollectorTagIncludesId` | DataCollector 标签的 id 属性 |
| `testCustomConfiguration` | 自定义配置值的正确性 |
| `testClientTransportsConfiguration` (DataProvider) | 各种传输配置组合下的服务注册 |
| `testServerServices` | 核心服务的注册和 Builder 方法调用 |
| `testMcpToolAttributeAutoconfiguration` | #[McpTool] 属性自动配置 |
| `testMcpPromptAttributeAutoconfiguration` | #[McpPrompt] 属性自动配置 |
| `testMcpResourceAttributeAutoconfiguration` | #[McpResource] 属性自动配置 |
| `testMcpResourceTemplateAttributeAutoconfiguration` | #[McpResourceTemplate] 属性自动配置 |
| `testHttpConfigurationDefaults` | HTTP 配置默认值和 FileSessionStore |
| `testHttpConfigurationCustom` | 自定义 HTTP 路径和 MemorySessionStore |
| `testSessionStoreFileConfiguration` | File Session Store 配置 |
| `testSessionStoreCacheConfigurationDefault` | Cache Session Store 默认配置 |
| `testSessionStoreCacheConfigurationCustom` | Cache Session Store 自定义配置 |
| `testDiscoveryDefaultConfiguration` | Discovery 默认扫描目录 |
| `testDiscoveryCustomConfiguration` | Discovery 自定义扫描/排除目录 |
| `testDiscoveryWithExcludeDirsOnly` | 只配置排除目录 |
| `testLoaderInterfaceAutoconfiguration` | LoaderInterface 自动配置标签 |
| `testRequestHandlerInterfaceAutoconfiguration` | RequestHandlerInterface 自动配置标签 |
| `testNotificationHandlerInterfaceAutoconfiguration` | NotificationHandlerInterface 自动配置标签 |

### McpPassTest

| 测试方法 | 验证内容 |
|----------|----------|
| `testCreatesServiceLocatorForAllMcpServices` | 所有4种 MCP 标签服务被收集到 ServiceLocator |
| `testDoesNothingWhenNoMcpServicesTagged` | 无标签服务时不创建 ServiceLocator |
| `testDoesNothingWhenNoServerBuilder` | 无 Builder 服务时安全跳过 |
| `testHandlesPartialMcpServices` | 部分 MCP 服务场景 |

### DataCollectorTest

| 测试方法 | 验证内容 |
|----------|----------|
| `testGetNameReturnsShortName` | 返回 'mcp' 短名称 |
| `testLateCollectPopulatesData` | lateCollect 正确填充数据 |
| `testGetTotalCountReturnsZeroForEmptyRegistry` | 空注册表返回 0 |

### TraceableRegistryTest

| 测试方法 | 验证内容 |
|----------|----------|
| `testResetCallsClear` | reset() 正确委托到 clear() |

## 测试模式和技巧

### 1. Container Builder 辅助方法
```php
private function buildContainer(array $configuration): ContainerBuilder
```
McpBundleTest 使用辅助方法创建预配置的容器，简化测试设置。

### 2. DataProvider 模式
```php
#[DataProvider('provideClientTransportsConfiguration')]
public function testClientTransportsConfiguration(array $config, array $expectedServices)
```
使用 PHPUnit DataProvider 测试多种传输配置组合。

### 3. 直接使用 Mock
TraceableRegistryTest 使用 PHPUnit Mock 验证委托行为。

### 4. 真实 Registry 集成测试
DataCollectorTest 使用真实的 `Registry` 实例而非 Mock，确保集成正确性。

## 测试覆盖盲区

- 未测试 McpCommand 的执行逻辑
- 未测试 McpController 的 HTTP 请求处理
- 未测试 RouteLoader 的路由生成
- 未测试 TraceableRegistry 的完整方法委托
- 未测试 DataCollector 注册了 MCP 能力后的数据收集
