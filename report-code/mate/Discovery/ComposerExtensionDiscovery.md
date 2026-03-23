# ComposerExtensionDiscovery.php 深度分析报告

## 1. 文件概述

`ComposerExtensionDiscovery.php` 负责从 Composer 安装的包中发现 MCP 扩展配置。它读取 `vendor/composer/installed.json` 和根项目的 `composer.json`，提取 `extra.ai-mate` 配置段，返回结构化的扩展数据（扫描目录、包含文件、指令文件路径）。

**文件路径**：`src/mate/src/Discovery/ComposerExtensionDiscovery.php`

---

## 2. 类签名与依赖

```php
namespace Symfony\AI\Mate\Discovery;

use Psr\Log\LoggerInterface;

/**
 * @phpstan-type ExtensionData array{
 *     dirs: string[],
 *     includes: string[],
 *     instructions?: string,
 * }
 */
final class ComposerExtensionDiscovery
```

### 属性

| 属性 | 类型 | 可见性 | 说明 |
|------|------|--------|------|
| `$rootDir` | `string` | `private` | 项目根目录 |
| `$logger` | `LoggerInterface` | `private` | PSR-3 日志接口 |
| `$installedPackages` | `?array` | `private` | 缓存的已安装包数据 |

### 构造函数

```php
public function __construct(
    private string $rootDir,
    private LoggerInterface $logger,
)
```

---

## 3. 方法级别分析

### 3.1 `discover(array|false $includeFilter = false): array`

**输入**：`$includeFilter` —— `array|false`，白名单过滤；`false` 表示包含所有包

**输出**：`array<string, ExtensionData>` —— 按包名索引的扩展数据

**实现逻辑**：
1. 调用 `getInstalledPackages()` 获取所有已安装包
2. 遍历每个包，检查 `extra.ai-mate` 配置是否存在
3. 跳过 `extension: false` 的包（显式排除）
4. 若提供白名单，仅包含列表中的包
5. 调用 `extractScanDirs()`、`extractIncludeFiles()`、`extractInstructions()` 提取配置
6. 返回以包名为键的扩展数据映射

### 3.2 `discoverRootProject(): array`

**输入**：无

**输出**：`ExtensionData` —— 根项目的扩展数据

**实现逻辑**：
1. 读取 `{rootDir}/composer.json`
2. 解析 JSON，提取 `extra.ai-mate` 配置
3. 构建并返回 `{dirs, includes, instructions}` 结构
4. 文件缺失或解析失败时返回默认空结构

### 3.3 `getInstalledPackages(): array`（私有）

**输出**：`array<string, array{name: string, extra: array<string, mixed>}>`

**实现逻辑**：
- 读取 `vendor/composer/installed.json`
- 兼容 Composer v1（直接数组）和 v2（`{"packages": [...]}` 包装）格式
- 验证 `name` 字段为字符串
- 结果按包名索引并缓存到 `$installedPackages`

### 3.4 `extractScanDirs(array $package, string $packageName): array`（私有）

**输入**：包元数据、包名

**输出**：`string[]` —— 相对于 vendor 目录的扫描路径

**安全措施**：
- 拒绝包含 `..` 的路径（防止路径遍历）
- 拒绝空字符串
- 验证目录实际存在于 `vendor/{packageName}/{dir}`
- 无配置时默认为包根目录

### 3.5 `extractIncludeFiles(array $package, string $packageName): array`（私有）

**输入**：包元数据、包名

**输出**：`string[]` —— 包含文件的**绝对路径**

**特点**：
- 接受字符串（转数组）或数组格式
- 返回完整绝对路径（非相对路径）
- 同样进行路径遍历防护和存在性验证

### 3.6 `extractInstructions(array $package, string $packageName): ?string`（私有）

**输入**：包元数据、包名

**输出**：`?string` —— 指令文件的相对路径

**逻辑**：验证为非空字符串、无路径遍历、文件存在

### 3.7 `extractAiMateConfig(array $composer, string $key): array`（私有）

**输入**：Composer 配置数组、配置键名

**输出**：`string[]` —— 过滤后的字符串值数组

**作用**：通用数组配置读取器，从 `extra.ai-mate.{key}` 提取值

### 3.8 `extractAiMateConfigString(array $composer, string $key): ?string`（私有）

**输入**：Composer 配置数组、配置键名

**输出**：`?string` —— 字符串配置值或 null

**作用**：通用字符串配置读取器

---

## 4. 设计模式分析

- **发现者模式（Discovery Pattern）**：自动扫描已安装包的元数据，无需手动注册
- **缓存模式**：`$installedPackages` 属性缓存避免重复读取 JSON 文件
- **防御性编程**：路径遍历检查、类型验证、文件存在性验证
- **适配器模式**：兼容 Composer v1 和 v2 的 `installed.json` 格式

---

## 5. 在模块中的调用场景

- `ContainerFactory::loadExtensions()` —— 发现启用的扩展并加载配置
- `ContainerFactory::loadUserServices()` —— 发现根项目配置
- `AgentInstructionsAggregator` —— 间接使用扩展数据中的 instructions 路径
- `DiscoverCommand` —— 列出和管理已发现的扩展

---

## 6. 可扩展性分析

- `ExtensionData` 类型定义使用 PHPStan type，便于未来添加新的配置字段
- `extra.ai-mate` 的键值对设计支持灵活扩展新配置项
- 白名单机制 (`$includeFilter`) 支持选择性发现

---

## 7. 技巧与最佳实践

- **安全第一**：所有文件路径经过严格验证，拒绝 `..`，防止路径遍历攻击
- **兼容性**：同时支持 Composer v1/v2 格式，确保广泛兼容
- **优雅降级**：缺失文件或无效配置仅记录警告，不阻断流程
- **关注点分离**：提取逻辑拆分为多个私有方法，每个方法职责单一
