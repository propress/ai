# Factory 分析报告

## 文件概述
Factory类是JSON Schema生成系统的入口点和门面（Facade）模式实现。它负责将PHP类和方法转换为符合JSON Schema标准的数组结构，主要用于AI平台的工具调用和参数定义。该类通过委托给ObjectDescriberInterface来完成实际的描述工作，实现了关注点分离。

## 类/接口定义
- **类型**: final class（最终类，不可继承）
- **命名空间**: `Symfony\AI\Platform\Contract\JsonSchema`
- **作者**: Christopher Hertel, Oskar Stark
- **依赖关系**: 依赖ObjectDescriberInterface接口，默认使用Describer实现
- **职责**: 作为JSON Schema生成的统一入口，提供简洁的API来构建参数和属性的Schema

## PHPStan类型定义
Factory类定义了详细的JsonSchema类型，包含以下关键元素：
- **type**: 固定为'object'
- **properties**: 属性定义数组，支持多种JSON Schema验证规则
  - 基本类型：type, description
  - 枚举和常量：enum, const
  - 字符串验证：pattern, minLength, maxLength
  - 数值验证：minimum, maximum, multipleOf, exclusiveMinimum, exclusiveMaximum
  - 数组验证：minItems, maxItems, uniqueItems, minContains, maxContains
  - 对象验证：minProperties, maxProperties, dependentRequired
  - 组合规则：anyOf
- **required**: 必填字段列表
- **additionalProperties**: 固定为false，不允许额外属性

## 方法分析

### __construct()
- **可见性**: public
- **参数**: 
  - `ObjectDescriberInterface $objectDescriber`: 对象描述器，默认为Describer实例
- **职责**: 初始化工厂，支持依赖注入自定义描述器
- **设计意图**: 允许用户替换默认的描述器实现，提供扩展性

### buildParameters()
- **可见性**: public
- **参数**:
  - `string $className`: 类的完全限定名
  - `string $methodName`: 方法名称
- **返回类型**: `JsonSchema|null`
- **功能描述**: 为指定方法的参数构建JSON Schema
- **实现细节**:
  1. 创建ObjectSubject，使用ReflectionMethod获取方法元信息
  2. 委托给objectDescriber处理描述逻辑
  3. 返回生成的schema数组
- **使用场景**: AI工具定义中，需要描述函数参数的类型和约束
- **边界情况**: 如果方法不存在会抛出ReflectionException

### buildProperties()
- **可见性**: public
- **参数**:
  - `string $className`: 类的完全限定名
- **返回类型**: `JsonSchema|null`
- **功能描述**: 为指定类的属性构建JSON Schema
- **实现细节**:
  1. 创建ObjectSubject，使用ReflectionClass获取类元信息
  2. 委托给objectDescriber处理描述逻辑
  3. 返回生成的schema数组
- **使用场景**: 当需要将PHP对象结构转换为JSON Schema时，用于数据验证或API文档生成
- **边界情况**: 如果类不存在会抛出ReflectionException

## 设计模式

### 1. 门面模式（Facade Pattern）
Factory类为复杂的JSON Schema生成系统提供了简化的接口。客户端只需调用buildParameters()或buildProperties()，无需了解内部的描述器链、Subject对象等复杂细节。

### 2. 依赖注入模式
通过构造函数注入ObjectDescriberInterface，允许用户自定义描述器实现，提高了灵活性和可测试性。默认使用Describer实现，体现了合理的默认值原则。

### 3. 策略模式（Strategy Pattern）
ObjectDescriberInterface定义了描述对象的策略接口，Factory依赖接口而非具体实现，可以在运行时切换不同的描述策略。

## 扩展点

### 自定义描述器
用户可以实现ObjectDescriberInterface接口，创建自定义的描述逻辑：

```php
class CustomDescriber implements ObjectDescriberInterface {
    public function describeObject(ObjectSubject $subject, ?array &$schema): void {
        // 自定义描述逻辑
    }
}

$factory = new Factory(new CustomDescriber());
```

### 组合描述器
可以使用装饰器模式或责任链模式组合多个描述器：

```php
class CompositeDescriber implements ObjectDescriberInterface {
    public function __construct(private array $describers) {}
    
    public function describeObject(ObjectSubject $subject, ?array &$schema): void {
        foreach ($this->describers as $describer) {
            $describer->describeObject($subject, $schema);
        }
    }
}
```

## 与其他文件的关系

### 依赖的接口和类
- **ObjectDescriberInterface**: 定义对象描述器的契约
- **Describer**: ObjectDescriberInterface的默认实现，是一个组合描述器
- **ObjectSubject**: 封装要描述的对象信息（类或方法）
- **ReflectionClass/ReflectionMethod**: PHP反射API，用于获取类和方法的元信息

### 被依赖的场景
- **ToolNormalizer**: 使用Factory为AI工具生成参数schema
- **AI Platform集成**: 各种Bridge实现使用Factory将PHP函数签名转换为AI提供商所需的JSON Schema格式

### 在系统中的位置
Factory是JSON Schema生成系统的入口点，位于整个描述器链的顶层。它协调了Subject（描述对象）、Describer（描述逻辑）和Schema（输出结果）三个核心概念。

## 使用示例

### 示例1: 为工具方法生成参数Schema
```php
use Symfony\AI\Platform\Contract\JsonSchema\Factory;

class WeatherTool {
    public function getWeather(string $city, string $unit = 'celsius'): string {
        // 实现
    }
}

$factory = new Factory();
$schema = $factory->buildParameters(WeatherTool::class, 'getWeather');

// 生成的schema:
// [
//     'type' => 'object',
//     'properties' => [
//         'city' => ['type' => 'string', 'description' => '...'],
//         'unit' => ['type' => 'string', 'description' => '...'],
//     ],
//     'required' => ['city'],
//     'additionalProperties' => false,
// ]
```

### 示例2: 为数据类生成属性Schema
```php
use Symfony\Component\Validator\Constraints as Assert;

class UserProfile {
    #[Assert\NotBlank]
    #[Assert\Email]
    public string $email;
    
    #[Assert\Length(min: 3, max: 50)]
    public string $name;
    
    #[Assert\Range(min: 18, max: 120)]
    public int $age;
}

$factory = new Factory();
$schema = $factory->buildProperties(UserProfile::class);

// schema会包含所有验证约束转换后的JSON Schema规则
```

### 示例3: 使用自定义描述器
```php
use Symfony\AI\Platform\Contract\JsonSchema\Describer\Describer;
use Symfony\AI\Platform\Contract\JsonSchema\Describer\TypeInfoDescriber;
use Symfony\AI\Platform\Contract\JsonSchema\Describer\ValidatorConstraintsDescriber;

// 只使用类型信息描述器，不包含验证约束
$customDescriber = new Describer([
    new TypeInfoDescriber(),
]);

$factory = new Factory($customDescriber);
$schema = $factory->buildParameters(MyClass::class, 'myMethod');
```

## 性能考虑
- 使用PHP反射API，在生产环境建议缓存生成的Schema
- 描述器链的执行是顺序的，可以通过减少描述器数量来优化性能
- final关键字允许PHP进行更好的优化

## 安全性
- 不执行用户输入的类名或方法名，由调用者保证安全性
- 生成的Schema不包含敏感信息（如属性值），只包含结构定义
- additionalProperties设为false可防止意外的额外字段
