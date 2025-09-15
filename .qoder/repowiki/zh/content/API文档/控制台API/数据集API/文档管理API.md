# 文档管理API

<cite>
**本文档中引用的文件**  
- [document.py](file://api/controllers/service_api/dataset/document.py)
- [dataset_service.py](file://api/services/dataset_service.py)
- [file_service.py](file://api/services/file_service.py)
- [document_fields.py](file://api/fields/document_fields.py)
- [dataset.py](file://api/models/dataset.py)
- [DocumentAddByFileApi](file://api/controllers/service_api/dataset/document.py#L224-L351)
- [DocumentListApi](file://api/controllers/service_api/dataset/document.py#L410-L441)
- [DocumentApi](file://api/controllers/service_api/dataset/document.py#L526-L606)
</cite>

## 目录
1. [简介](#简介)
2. [核心端点](#核心端点)
3. [文档上传](#文档上传)
4. [文档列表](#文档列表)
5. [获取文档详情](#获取文档详情)
6. [更新文档](#更新文档)
7. [删除文档](#删除文档)
8. [异步处理状态轮询](#异步处理状态轮询)
9. [分段策略与嵌入模型](#分段策略与嵌入模型)
10. [错误处理与故障排除](#错误处理与故障排除)
11. [代码示例](#代码示例)

## 简介
Dify文档管理API提供了一套完整的接口，用于管理知识库中的文档。支持通过多种方式上传文档，包括本地文件上传、网页抓取和Notion同步。API支持对文档进行创建、读取、更新和删除（CRUD）操作，并提供了异步处理状态查询机制。文档在上传后会经过解析、清洗、分段和索引等处理流程，最终用于检索增强生成（RAG）场景。

**Section sources**
- [document.py](file://api/controllers/service_api/dataset/document.py#L1-L677)

## 核心端点
文档管理API的核心端点如下：

| HTTP方法 | URL路径 | 描述 |
|---------|--------|------|
| POST | /datasets/{dataset_id}/documents | 通过文件或文本创建新文档 |
| GET | /datasets/{dataset_id}/documents | 列出数据集中的所有文档 |
| GET | /datasets/{dataset_id}/documents/{document_id} | 获取特定文档的详细信息 |
| PATCH | /datasets/{dataset_id}/documents/{document_id} | 更新现有文档（通过文件或文本） |
| DELETE | /datasets/{dataset_id}/documents/{document_id} | 删除指定文档 |

**Section sources**
- [document.py](file://api/controllers/service_api/dataset/document.py#L224-L606)

## 文档上传
文档可以通过多部分表单数据（multipart/form-data）上传。上传时需要指定`dataset_id`，并可选择性地提供处理规则。

### 请求格式
```http
POST /datasets/{dataset_id}/document/create-by-file HTTP/1.1
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW
Authorization: Bearer <your_api_token>

------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="file"; filename="example.pdf"
Content-Type: application/pdf

<文件二进制内容>
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="data"
Content-Type: application/json

{
  "doc_form": "text_model",
  "doc_language": "Chinese",
  "process_rule": {
    "mode": "custom",
    "rules": {
      "pre_processing_rules": [
        {"id": "remove_extra_spaces", "enabled": true},
        {"id": "remove_urls", "enabled": false}
      ],
      "segmentation": {
        "delimiter": "\n",
        "max_tokens": 512
      }
    }
  }
}
------WebKitFormBoundary7MA4YWxkTrZu0gW--
```

### 支持的文档类型
- **PDF**: .pdf
- **Word**: .docx, .doc
- **Excel**: .xlsx, .xls
- **文本**: .txt, .md, .csv
- **网页**: 通过URL抓取

**Section sources**
- [document.py](file://api/controllers/service_api/dataset/document.py#L224-L351)
- [file_service.py](file://api/services/file_service.py#L1-L50)

## 文档列表
获取数据集中所有文档的列表，支持分页和搜索。

### 请求示例
```http
GET /datasets/123e4567-e89b-12d3-a456-426614174000/documents?page=1&limit=20&keyword=report HTTP/1.1
Authorization: Bearer <your_api_token>
```

### 响应格式
```json
{
  "data": [
    {
      "id": "doc-123",
      "name": "annual_report_2023.pdf",
      "created_at": 1700000000,
      "indexing_status": "completed",
      "tokens": 15000,
      "segment_count": 30,
      "hit_count": 45
    }
  ],
  "has_more": false,
  "limit": 20,
  "total": 1,
  "page": 1
}
```

**Section sources**
- [document.py](file://api/controllers/service_api/dataset/document.py#L410-L441)

## 获取文档详情
获取特定文档的详细信息，包括元数据、处理状态和配置。

### 请求示例
```http
GET /datasets/123e4567-e89b-12d3-a456-426614174000/documents/doc-123?metadata=all HTTP/1.1
Authorization: Bearer <your_api_token>
```

### 响应格式
```json
{
  "id": "doc-123",
  "name": "annual_report_2023.pdf",
  "created_at": 1700000000,
  "indexing_status": "completed",
  "completed_at": 1700000100,
  "tokens": 15000,
  "segment_count": 30,
  "data_source_type": "upload_file",
  "data_source_info": {
    "file_ids": ["file-456"]
  },
  "dataset_process_rule": {
    "mode": "automatic",
    "rules": "{\"pre_processing_rules\":[{\"id\":\"remove_extra_spaces\",\"enabled\":true}],\"segmentation\":{\"delimiter\":\"\\n\",\"max_tokens\":512}}"
  },
  "doc_metadata": {
    "author": "John Doe",
    "created_date": "2023-12-01"
  }
}
```

**Section sources**
- [document.py](file://api/controllers/service_api/dataset/document.py#L526-L606)

## 更新文档
通过上传新文件来更新现有文档。

### 请求示例
```http
POST /datasets/{dataset_id}/documents/{document_id}/update-by-file HTTP/1.1
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW
Authorization: Bearer <your_api_token>

------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="file"; filename="updated_report.pdf"
Content-Type: application/pdf

<更新后的文件二进制内容>
------WebKitFormBoundary7MA4YWxkTrZu0gW--
```

**Section sources**
- [document.py](file://api/controllers/service_api/dataset/document.py#L319-L351)

## 删除文档
删除指定的文档。

### 请求示例
```http
DELETE /datasets/123e4567-e89b-12d3-a456-426614174000/documents/doc-123 HTTP/1.1
Authorization: Bearer <your_api_token>
```

### 响应
- **204 No Content**: 文档删除成功
- **403 Forbidden**: 文档已归档，无法删除
- **404 Not Found**: 文档不存在

**Section sources**
- [document.py](file://api/controllers/service_api/dataset/document.py#L583-L606)

## 异步处理状态轮询
文档上传后会异步处理，可通过`task_id`查询处理进度。

### 状态轮询端点
```http
GET /datasets/{dataset_id}/documents/{document_id}/indexing-status HTTP/1.1
Authorization: Bearer <your_api_token>
```

### 状态响应
```json
{
  "data": [
    {
      "id": "doc-123",
      "indexing_status": "completed",
      "processing_started_at": "2023-11-15T10:00:00Z",
      "parsing_completed_at": "2023-11-15T10:01:00Z",
      "cleaning_completed_at": "2023-11-15T10:02:00Z",
      "splitting_completed_at": "2023-11-15T10:03:00Z",
      "completed_at": "2023-11-15T10:04:00Z",
      "completed_segments": 30,
      "total_segments": 30
    }
  ]
}
```

**Section sources**
- [document.py](file://api/controllers/service_api/dataset/document.py#L439-L472)

## 分段策略与嵌入模型
文档处理支持多种分段策略和嵌入模型选择。

### 分段策略
- **固定分段**: 按固定token数量分段
- **语义分段**: 按语义边界（如段落）分段
- **层次分段**: 结合章节和段落的层次结构分段

### 嵌入模型参数
在创建文档时可通过`embedding_model`和`embedding_model_provider`参数指定嵌入模型：
```json
{
  "embedding_model_provider": "openai",
  "embedding_model": "text-embedding-ada-002"
}
```

**Section sources**
- [document.py](file://api/controllers/service_api/dataset/document.py#L81-L114)
- [dataset_service.py](file://api/services/dataset_service.py#L1472-L1501)

## 错误处理与故障排除
### 常见错误响应

| HTTP状态码 | 错误码 | 描述 | 解决方案 |
|----------|-------|------|---------|
| 413 | file_too_large | 请求体过大 | 检查文件大小是否超过系统限制 |
| 422 | document_indexing | 文档处理失败 | 检查文件格式是否支持，查看错误详情 |
| 400 | invalid_url | 无效URL | 验证URL格式和可访问性 |
| 403 | archived_document_immutable | 已归档文档不可修改 | 取消归档后再操作 |
| 409 | dataset_in_use | 数据集正在被使用 | 先从应用中移除数据集 |

### 故障排除指南
1. **文档上传失败**: 检查文件大小、格式和网络连接
2. **处理超时**: 大文件可能需要更长时间处理，建议轮询状态
3. **索引失败**: 检查嵌入模型配置和API密钥
4. **权限错误**: 验证API令牌和数据集访问权限

**Section sources**
- [error.py](file://api/controllers/service_api/dataset/error.py#L0-L48)
- [document.py](file://api/controllers/service_api/dataset/document.py#L583-L606)

## 代码示例
### cURL示例
```bash
# 上传文件
curl -X POST "https://api.dify.ai/v1/datasets/123e4567-e89b-12d3-a456-426614174000/document/create-by-file" \
  -H "Authorization: Bearer your-api-token" \
  -H "Content-Type: multipart/form-data" \
  -F "file=@/path/to/document.pdf" \
  -F 'data={"doc_language":"Chinese"}'

# 查询文档状态
curl -X GET "https://api.dify.ai/v1/datasets/123e4567-e89b-12d3-a456-426614174000/documents/doc-123/indexing-status" \
  -H "Authorization: Bearer your-api-token"
```

### Python客户端示例
```python
import requests

class DifyClient:
    def __init__(self, api_key, base_url="https://api.dify.ai"):
        self.api_key = api_key
        self.base_url = base_url
        self.headers = {"Authorization": f"Bearer {api_key}"}

    def upload_document(self, dataset_id, file_path, doc_language="Chinese"):
        url = f"{self.base_url}/v1/datasets/{dataset_id}/document/create-by-file"
        
        with open(file_path, 'rb') as f:
            files = {'file': f}
            data = {'data': json.dumps({'doc_language': doc_language})}
            response = requests.post(url, headers=self.headers, files=files, data=data)
            
        return response.json()

    def get_document_status(self, dataset_id, document_id):
        url = f"{self.base_url}/v1/datasets/{dataset_id}/documents/{document_id}/indexing-status"
        response = requests.get(url, headers=self.headers)
        return response.json()

# 使用示例
client = DifyClient("your-api-token")
result = client.upload_document("dataset-123", "/path/to/document.pdf")
print(f"文档上传成功，ID: {result['document']['id']}")

# 轮询处理状态
import time
while True:
    status = client.get_document_status("dataset-123", result['document']['id'])
    indexing_status = status['data'][0]['indexing_status']
    print(f"处理状态: {indexing_status}")
    
    if indexing_status == "completed":
        print("文档处理完成！")
        break
    elif indexing_status == "error":
        print(f"处理失败: {status['data'][0]['error']}")
        break
        
    time.sleep(5)  # 每5秒查询一次
```

**Section sources**
- [document.py](file://api/controllers/service_api/dataset/document.py#L224-L351)
- [document.py](file://api/controllers/service_api/dataset/document.py#L439-L472)