# DocumentUrl 分析报告

## 文件概述
DocumentUrl 类用于表示通过 URL 引用的远程文档内容。与 Document 类（存储文档数据）不同，DocumentUrl 仅存储文档的 URL 地址，适用于文档已托管在公开服务器上的场景。这种设计避免了大文件的上传和传输，对于已有 URL 的文档（如云存储、CDN 上的文件）更高效，是处理远程文档的轻量级解决方案。

## 类/接口定义

### DocumentUrl
- **类型**: final class（最终类）
- **继承/实现**: 实现 `ContentInterface` 接口
- **命名空间**: `Symfony\AI\Platform\Message\Content`
- **职责**: 存储和提供远程文档的 URL 地址

## 属性分析

### $url
- **类型**: `string`
- **可见性**: private readonly
- **说明**: 文档的 URL 地址，使用 readonly 确保不可变性

## 方法分析

### __construct()
- **可见性**: public
- **参数**: 
  - `$url` (`string`): 文档的 URL 地址
- **返回值**: 无
- **功能说明**: 构造文档 URL 对象，接受一个 URL 字符串并存储。
- **注意事项**: 不验证 URL 有效性或文档可访问性

### getUrl()
- **可见性**: public
- **参数**: 无
- **返回值**: `string` - 文档的 URL 地址
- **功能说明**: 获取存储的文档 URL。
- **注意事项**: 返回原始 URL，不做任何处理

## 设计模式

### 值对象（Value Object）
DocumentUrl 是不可变的值对象，只封装 URL 字符串，没有业务逻辑。

### 引用模式（Reference Pattern）
通过 URL 引用外部资源，而不存储实际数据，节省内存和传输成本。

## 使用示例

```php
use Symfony\AI\Platform\Message\Content\DocumentUrl;
use Symfony\AI\Platform\Message\Content\Document;
use Symfony\AI\Platform\Message\UserMessage;
use Symfony\AI\Platform\Message\Content\Text;

// 1. 创建文档 URL
$docUrl = new DocumentUrl('https://example.com/report.pdf');

// 2. 从云存储引用
$cloudDoc = new DocumentUrl('https://storage.example.com/documents/annual_report.pdf');

// 3. 在消息中使用
$message = new UserMessage(
    new Text('请分析这份报告'),
    new DocumentUrl('https://example.com/financial_report.pdf')
);

// 4. 混合本地和远程文档
$message = new UserMessage(
    new Text('比较这两份文档'),
    Document::fromFile('/local/doc.pdf'),
    new DocumentUrl('https://example.com/remote_doc.pdf')
);

// 5. 获取 URL
$url = $docUrl->getUrl();
echo $url;

// 6. URL 验证
function validateDocumentUrl(DocumentUrl $docUrl): bool
{
    $url = $docUrl->getUrl();
    
    if (!filter_var($url, FILTER_VALIDATE_URL)) {
        return false;
    }
    
    $ext = pathinfo(parse_url($url, PHP_URL_PATH), PATHINFO_EXTENSION);
    return in_array(strtolower($ext), ['pdf', 'docx', 'doc', 'xlsx', 'txt']);
}

// 7. 批量文档 URL
$docUrls = array_map(
    fn($url) => new DocumentUrl($url),
    [
        'https://example.com/doc1.pdf',
        'https://example.com/doc2.docx',
        'https://example.com/doc3.xlsx',
    ]
);
```
