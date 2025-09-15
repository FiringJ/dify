# API集成

<cite>
**本文档引用文件**  
- [workflow.py](file://api/controllers/web/workflow.py)
- [workflow_run_service.py](file://api/services/workflow_run_service.py)
- [workflow_fields.py](file://api/fields/workflow_fields.py)
- [workflow_run_fields.py](file://api/fields/workflow_run_fields.py)
- [index.js](file://sdks/nodejs-client/index.js)
- [dify_client/__init__.py](file://sdks/python-client/dify_client/__init__.py)
- [app_config.py](file://api/configs/app_config.py)
- [constants.py](file://api/core/workflow/constants.py)
- [enums.py](file://api/core/workflow/enums.py)
- [models.py](file://api/models/workflow.py)
</cite>

## 目录
1. [简介](#简介)
2. [认证机制](#认证机制)
3. [API端点与请求方法](#api端点与请求方法)
4. [请求体结构](#请求体结构)
5. [响应格式](#响应格式)
6. [异步执行与轮询机制](#异步执行与轮询机制)
7. [客户端调用示例](#客户端调用示例)
8. [错误处理与重试策略](#错误处理与重试策略)
9. [API版本控制](#api版本控制)
10. [部署模式配置](#部署模式配置)

## 简介
Dify工作流API允许开发者通过RESTful接口触发和管理自动化工作流。该API支持异步执行模式，适用于复杂任务编排、数据处理和AI驱动的业务流程。本文档详细说明如何集成Dify工作流API，涵盖认证、请求结构、响应处理、错误恢复和部署配置。

## 认证机制
Dify API使用API Key进行身份验证。API Key需通过HTTP请求头传递：

```
Authorization: Bearer <API_KEY>
```

API Key可在Dify控制台的“设置”->“API密钥”中生成和管理。每个API Key关联特定工作区权限，确保调用者具备执行工作流的授权。

**Section sources**
- [workflow.py](file://api/controllers/web/workflow.py#L25-L45)
- [app_config.py](file://api/configs/app_config.py#L15-L30)

## API端点与请求方法
工作流执行API采用标准RESTful设计，使用POST方法触发执行：

```
POST /api/workflows/<workflow_id>/run
```

URL路径中的`<workflow_id>`为工作流唯一标识符，可在Dify控制台获取。该端点接受JSON格式的请求体，并返回异步任务标识。

**Section sources**
- [workflow.py](file://api/controllers/web/workflow.py#L50-L70)
- [models.py](file://api/models/workflow.py#L10-L25)

## 请求体结构
请求体包含工作流执行所需的所有输入参数，主要由以下部分组成：

```json
{
  "inputs": {
    "field1": "value1",
    "field2": "value2"
  },
  "query": "用户查询文本",
  "files": [
    {
      "type": "image",
      "transfer_method": "remote_url",
      "url": "https://example.com/image.jpg"
    }
  ],
  "response_mode": "streaming",
  "user": "user_id"
}
```

- **inputs**: 工作流节点所需的输入变量，结构与工作流定义匹配
- **query**: 可选的用户输入文本，用于对话式工作流
- **files**: 附加文件列表，支持远程URL或预上传文件ID
- **response_mode**: 响应模式，支持"blocking"（同步）和"streaming"（流式）
- **user**: 调用者用户标识，用于审计和权限控制

**Section sources**
- [workflow_fields.py](file://api/fields/workflow_fields.py#L10-L50)
- [workflow_run_service.py](file://api/services/workflow_run_service.py#L30-L60)

## 响应格式
API返回标准化的JSON响应，包含任务标识和执行状态：

```json
{
  "task_id": "task-123",
  "workflow_run_id": "run-456",
  "status": "queued",
  "created_at": 1700000000
}
```

- **task_id**: 任务队列标识，用于查询执行状态
- **workflow_run_id**: 工作流执行实例唯一ID
- **status**: 当前状态，可能值包括"queued"、"running"、"succeeded"、"failed"
- **created_at**: 执行创建时间戳

**Section sources**
- [workflow_run_fields.py](file://api/fields/workflow_run_fields.py#L5-L35)
- [workflow_run_service.py](file://api/services/workflow_run_service.py#L80-L100)

## 异步执行与轮询机制
工作流默认以异步模式执行。调用者需通过状态查询接口轮询执行结果：

```
GET /api/workflow-runs/<workflow_run_id>
```

推荐轮询策略：
- 初始延迟：1秒
- 最大轮询间隔：5秒
- 超时时间：300秒
- 状态检查：持续轮询直到状态为"succeeded"或"failed"

对于流式响应模式，可通过SSE（Server-Sent Events）接收实时输出。

**Section sources**
- [workflow.py](file://api/controllers/web/workflow.py#L120-L150)
- [constants.py](file://api/core/workflow/constants.py#L5-L20)
- [enums.py](file://api/core/workflow/enums.py#L10-L30)

## 客户端调用示例

### Python客户端
```python
from dify_client import Client

client = Client(api_key="your-api-key")
response = client.create_workflow_run(
    workflow_id="your-workflow-id",
    inputs={"text": "Hello World"},
    timeout=300
)

# 轮询结果
result = client.get_workflow_run_status(response['workflow_run_id'])
```

### Node.js客户端
```javascript
const { DifyClient } = require('dify-client');

const client = new DifyClient({ apiKey: 'your-api-key' });
const response = await client.createWorkflowRun({
  workflowId: 'your-workflow-id',
  inputs: { text: 'Hello World' },
  timeout: 300
});

// 轮询结果
const status = await client.getWorkflowRunStatus(response.workflow_run_id);
```

**Section sources**
- [dify_client/__init__.py](file://sdks/python-client/dify_client/__init__.py#L20-L80)
- [index.js](file://sdks/nodejs-client/index.js#L15-L70)

## 错误处理与重试策略
API可能返回以下HTTP状态码：
- `400 Bad Request`: 输入参数无效
- `401 Unauthorized`: 认证失败
- `404 Not Found`: 工作流不存在
- `429 Too Many Requests`: 请求频率超限
- `500 Internal Server Error`: 服务器内部错误

建议实现指数退避重试策略：
- 初始重试延迟：1秒
- 退避因子：2
- 最大重试次数：3次
- 对4xx客户端错误不进行重试

**Section sources**
- [workflow.py](file://api/controllers/web/workflow.py#L180-L220)
- [workflow_run_service.py](file://api/services/workflow_run_service.py#L120-L150)

## API版本控制
Dify API通过URL路径实现版本控制：

```
/api/v1/workflows/<workflow_id>/run
```

- 当前稳定版本：v1
- 向后兼容性：保证同一主版本内接口兼容
- 版本升级：新功能在次版本中添加，不破坏现有接口
- 弃用策略：旧版本在发布新主版本后维持6个月支持期

**Section sources**
- [app_config.py](file://api/configs/app_config.py#L40-L55)
- [workflow.py](file://api/controllers/web/workflow.py#L10-L20)

## 部署模式配置
### SaaS部署
- Endpoint: `https://api.dify.ai`
- 认证：标准API Key
- 速率限制：根据订阅计划

### 私有部署
- Endpoint: `https://<your-domain>/api`
- 认证：API Key或企业SSO集成
- 配置：通过`API_BASE_URL`环境变量设置
- 网络：支持VPC内网访问和SSL配置

**Section sources**
- [app_config.py](file://api/configs/app_config.py#L60-L80)
- [workflow.py](file://api/controllers/web/workflow.py#L30-L40)