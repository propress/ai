# Decart Bridge 分析报告

## 概述

Decart Bridge 将 [Decart AI](https://decart.ai) 平台集成到 Symfony AI Platform 中，专注于**多模态生成**能力，包括文本生成图像（T2I）、文本生成视频（T2V）、图像生成图像（I2I）、图像生成视频（I2V）以及视频生成视频（V2V）。所有生成结果均以 `BinaryResult` 形式返回（直接包含响应的二进制内容及 `Content-Type`），无文本输出。

---

## 目录结构

```
Decart/
├── Decart.php                       # 模型标识类（继承 Model，final）
├── DecartClient.php                 # HTTP 客户端，处理生成与编辑请求
├── DecartResultConverter.php        # 将 HTTP 响应转换为 BinaryResult
├── ModelCatalog.php                 # 静态模型目录（lucy 系列模型）
├── PlatformFactory.php              # Platform 工厂方法
└── Contract/
    ├── DecartContract.php           # Contract 工厂，注册 ImageNormalizer 和 VideoNormalizer
    ├── ImageNormalizer.php          # 将 Image 内容序列化为 API 格式
    └── VideoNormalizer.php          # 将 Video 内容序列化为 API 格式
```

---

## 关键设计模式

### 1. 生成与编辑双路由
`DecartClient::request()` 根据模型能力选择两条请求路径：
- **生成**（TEXT_TO_IMAGE 或 TEXT_TO_VIDEO）→ `generate()`：以 `prompt` 字段提交文本
- **编辑**（IMAGE_TO_IMAGE、IMAGE_TO_VIDEO、VIDEO_TO_VIDEO）→ `edit()`：同时提交 `prompt` 和 `data`（文件句柄）

两者均调用同一端点 `POST {hostUrl}/generate/{model_name}`，通过 `x-api-key` Header 认证。

### 2. 动态 Content-Type
`DecartResultConverter` 从 HTTP 响应 Header 中读取 `content-type`（而非硬编码 `audio/mpeg` 等），使结果格式自适应不同模型的输出类型（图像或视频）。

### 3. 双媒体类型 Normalizer
`DecartContract` 同时注册 `ImageNormalizer` 和 `VideoNormalizer`，分别将 `Image` 和 `Video` 内容对象序列化为 `{type: input_image/input_video, input_image/input_video: {data, path, format}}` 结构。

### 4. 可配置端点
`DecartClient` 支持通过构造函数参数 `$hostUrl` 自定义 API 基础地址（默认为 `https://api.decart.ai/v1`），`PlatformFactory` 同样暴露此参数，便于测试或私有部署。

---

## 组件关系图

```
PlatformFactory
    └── Platform
            ├── DecartClient          ← HTTP 请求（生成 / 编辑）
            ├── DecartResultConverter ← 转换 HTTP 响应
            ├── ModelCatalog
            └── DecartContract
                    ├── ImageNormalizer ← 序列化 Image 内容
                    └── VideoNormalizer ← 序列化 Video 内容

DecartClient → RawHttpResult
DecartResultConverter → BinaryResult（动态 content-type）
```

---

## 支持的模型（Lucy 系列）

| 模型 ID              | 能力                              |
|----------------------|-----------------------------------|
| `lucy-dev-i2v`       | IMAGE_TO_VIDEO, VIDEO_TO_VIDEO    |
| `lucy-pro-t2i`       | TEXT_TO_IMAGE                     |
| `lucy-pro-t2v`       | TEXT_TO_VIDEO, IMAGE_TO_VIDEO     |
| `lucy-pro-i2i`       | IMAGE_TO_IMAGE                    |
| `lucy-pro-i2v`       | IMAGE_TO_VIDEO                    |
| `lucy-pro-v2v`       | VIDEO_TO_VIDEO                    |
| `lucy-pro-flf2v`     | IMAGE_TO_VIDEO（首末帧到视频）    |

---

## 外部依赖

- **Symfony HttpClient**（`EventSourceHttpClient`）
- **Decart REST API**：`https://api.decart.ai/v1`
