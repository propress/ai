# Contract/JsonSchema/Subject 目录分析报告

## 目录职责

`Contract/JsonSchema/Subject/` 目录包含描述主题类，封装反射信息供描述器使用。

**目录路径**: `src/platform/src/Contract/JsonSchema/Subject/`

---

## 包含的文件清单

| 文件 | 说明 |
|------|------|
| `ObjectSubject.php` | 对象主题，封装类反射 |
| `PropertySubject.php` | 属性主题，封装属性反射 |

---

## ObjectSubject

```php
class ObjectSubject
{
    public function __construct(
        private readonly \ReflectionClass $class,
    )
    
    public function getClass(): \ReflectionClass;
    public function getClassName(): string;
}
```

---

## PropertySubject

```php
class PropertySubject
{
    public function __construct(
        private readonly \ReflectionProperty $property,
        private readonly ObjectSubject $object,
    )
    
    public function getProperty(): \ReflectionProperty;
    public function getObject(): ObjectSubject;
    public function getName(): string;
    public function getClassName(): string;
}
```

---

## 使用方式

```php
$class = new \ReflectionClass(MyClass::class);
$objectSubject = new ObjectSubject($class);

foreach ($class->getProperties() as $property) {
    $propertySubject = new PropertySubject($property, $objectSubject);
    
    // 描述器处理属性
    $schema = $describer->describe($propertySubject);
}
```
