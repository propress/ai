# Document 分析报告

## 文件概述
Document 类继承自 File 类，专门用于表示文档内容（如 PDF、Word、Excel 等）。它通过类型系统区分文档与其他文件类型，支持文档分析、内容提取等 AI 功能。Document 继承了 File 的所有文件处理能力，可以处理各种文档格式，支持从本地文件、二进制数据和 Data URL 创建，是构建文档理解和分析功能的基础组件。

## 类/接口定义

### Document
- **类型**: final class（最终类）
- **继承/实现**: 继承 `File` 类，间接实现 `ContentInterface` 接口
- **命名空间**: `Symfony\AI\Platform\Message\Content`
- **职责**: 表示文档内容，提供类型标识和文件处理能力

## 设计模式

### 1. 继承专用化（Inheritance Specialization）
Document 通过继承 File 获得所有文件处理功能，自身专注于类型标识。这种模式的好处：
- **代码复用**: 避免重复实现文件处理逻辑
- **类型区分**: 通过类型系统识别文档
- **语义明确**: Document 比 File 更清晰表达意图
- **未来扩展**: 可以添加文档特定的方法（如页数、提取文本等）

### 2. 标记类（Marker Class）
类体为空，纯粹用于类型识别，是一种轻量级的类型系统设计。

## 与其他文件的关系

### 依赖关系
- **File**: 父类，提供所有文件处理功能
- **ContentInterface**: 通过 File 间接实现

### 被依赖关系
- **UserMessage**: 可以包含 Document 内容
- **文档分析器**: 提取和分析文档内容
- **多模态处理器**: 路由文档到相应的处理器
- **RAG 系统**: 文档检索和增强生成

## 使用示例

```php
use Symfony\AI\Platform\Message\Content\Document;
use Symfony\AI\Platform\Message\Content\DocumentUrl;
use Symfony\AI\Platform\Message\UserMessage;
use Symfony\AI\Platform\Message\Content\Text;

// 1. 从 PDF 文件创建
$pdf = Document::fromFile('/path/to/document.pdf');

// 2. 从 Word 文档创建
$docx = Document::fromFile('/path/to/report.docx');

// 3. 从二进制数据创建
$docData = file_get_contents('/path/to/file.pdf');
$doc = new Document($docData, 'application/pdf');

// 4. 从 Data URL 创建
$dataUrl = 'data:application/pdf;base64,JVBERi0x...';
$doc = Document::fromDataUrl($dataUrl);

// 5. 在消息中使用
$message = new UserMessage(
    new Text('请总结这份文档的要点：'),
    Document::fromFile('/path/to/report.pdf')
);

// 6. 多文档分析
$message = new UserMessage(
    new Text('比较这三份文档：'),
    Document::fromFile('/docs/version1.pdf'),
    Document::fromFile('/docs/version2.pdf'),
    Document::fromFile('/docs/version3.pdf')
);

// 7. 获取文档信息（继承自 File）
$format = $doc->getFormat(); // 'application/pdf'
$filename = $doc->getFilename(); // 'document.pdf'
$binary = $doc->asBinary(); // 原始文档数据
$base64 = $doc->asBase64(); // Base64 编码
$dataUrl = $doc->asDataUrl(); // Data URL

// 8. 文档格式验证
function validateDocument(Document $doc): bool
{
    $allowedFormats = [
        'application/pdf',                                      // PDF
        'application/msword',                                   // DOC
        'application/vnd.openxmlformats-officedocument.wordprocessingml.document', // DOCX
        'application/vnd.ms-excel',                            // XLS
        'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet', // XLSX
        'application/vnd.ms-powerpoint',                       // PPT
        'application/vnd.openxmlformats-officedocument.presentationml.presentation', // PPTX
        'text/plain',                                          // TXT
        'text/csv',                                            // CSV
        'text/markdown',                                       // MD
    ];
    
    return in_array($doc->getFormat(), $allowedFormats);
}

// 9. 类型检测和处理
foreach ($message->getContent() as $content) {
    if ($content instanceof Document) {
        echo "文档: " . $content->getFilename() . "\n";
        echo "格式: " . $content->getFormat() . "\n";
        echo "大小: " . strlen($content->asBinary()) . " 字节\n";
    }
}

// 10. 文档大小检查
function validateDocumentSize(Document $doc, int $maxSizeMB = 50): void
{
    $size = strlen($doc->asBinary());
    $maxSize = $maxSizeMB * 1024 * 1024;
    
    if ($size > $maxSize) {
        throw new \InvalidArgumentException(
            "Document too large: " . round($size / 1024 / 1024, 2) . " MB"
        );
    }
}

// 11. 准备文档上传
function prepareDocumentForApi(Document $doc): array
{
    return [
        'type' => 'document',
        'format' => $doc->getFormat(),
        'data' => $doc->asBase64(),
        'filename' => $doc->getFilename(),
    ];
}

// 12. 文档类型检测
function getDocumentType(Document $doc): string
{
    return match ($doc->getFormat()) {
        'application/pdf' => 'PDF',
        'application/msword',
        'application/vnd.openxmlformats-officedocument.wordprocessingml.document' 
            => 'Word',
        'application/vnd.ms-excel',
        'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet' 
            => 'Excel',
        'application/vnd.ms-powerpoint',
        'application/vnd.openxmlformats-officedocument.presentationml.presentation' 
            => 'PowerPoint',
        'text/plain' => 'Text',
        'text/csv' => 'CSV',
        'text/markdown' => 'Markdown',
        default => 'Unknown',
    };
}

// 13. 混合文档和文本
$message = new UserMessage(
    new Text('文档分析请求：'),
    Document::fromFile('/reports/annual_report.pdf'),
    new Text('请重点关注财务数据部分')
);

// 14. 文档页数估算（PDF）
function estimatePageCount(Document $doc): ?int
{
    if ($doc->getFormat() !== 'application/pdf') {
        return null;
    }
    
    // 简单估算：平均每页约 50KB
    $size = strlen($doc->asBinary());
    return (int) ceil($size / (50 * 1024));
}

// 15. 批量文档处理
$documents = [
    Document::fromFile('/docs/file1.pdf'),
    Document::fromFile('/docs/file2.docx'),
    Document::fromFile('/docs/file3.xlsx'),
];

$stats = [
    'total_count' => count($documents),
    'total_size' => 0,
    'types' => [],
];

foreach ($documents as $doc) {
    $stats['total_size'] += strlen($doc->asBinary());
    $type = getDocumentType($doc);
    $stats['types'][$type] = ($stats['types'][$type] ?? 0) + 1;
}

// 16. 文档与 URL 的选择
// 本地文档
$localDoc = Document::fromFile('/path/to/local.pdf');

// 远程文档
$remoteDoc = new DocumentUrl('https://example.com/document.pdf');

// 两者都可以用于消息
$message = new UserMessage(
    new Text('分析这些文档'),
    $localDoc,
    $remoteDoc
);

// 17. 安全的文档处理
function createSafeDocument(string $path): Document
{
    // 检查文件存在
    if (!file_exists($path)) {
        throw new \InvalidArgumentException("File not found: $path");
    }
    
    // 创建文档
    $doc = Document::fromFile($path);
    
    // 验证格式
    if (!validateDocument($doc)) {
        throw new \InvalidArgumentException("Unsupported document format");
    }
    
    // 验证大小
    validateDocumentSize($doc, 50);
    
    return $doc;
}
```

## 最佳实践

### 1. 格式白名单
```php
const ALLOWED_DOCUMENT_FORMATS = [
    'application/pdf',
    'application/msword',
    'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
    'text/plain',
];

function isAllowedFormat(Document $doc): bool
{
    return in_array($doc->getFormat(), ALLOWED_DOCUMENT_FORMATS);
}
```

### 2. 大小限制
```php
// 不同文档类型可以有不同的大小限制
function validateByType(Document $doc): void
{
    $limits = [
        'application/pdf' => 50,  // 50 MB
        'text/plain' => 10,       // 10 MB
        'text/csv' => 20,         // 20 MB
    ];
    
    $format = $doc->getFormat();
    $limit = $limits[$format] ?? 25;
    
    validateDocumentSize($doc, $limit);
}
```

### 3. 延迟加载
```php
// fromFile 使用闭包延迟加载
$doc = Document::fromFile('/path/to/large_document.pdf');

// 文件内容仅在需要时读取
$content = $doc->asBinary();
```

### 4. 临时文件管理
```php
function processDocument(Document $doc): void
{
    if ($path = $doc->asPath()) {
        // 直接使用原始路径
        processDocumentFile($path);
    } else {
        // 创建临时文件
        $ext = match ($doc->getFormat()) {
            'application/pdf' => '.pdf',
            'text/plain' => '.txt',
            default => '',
        };
        
        $tempPath = tempnam(sys_get_temp_dir(), 'doc_') . $ext;
        file_put_contents($tempPath, $doc->asBinary());
        
        try {
            processDocumentFile($tempPath);
        } finally {
            unlink($tempPath);
        }
    }
}
```
