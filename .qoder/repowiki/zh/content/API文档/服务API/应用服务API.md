# 应用服务API

<cite>
**本文档中引用的文件**  
- [app.py](file://api/controllers/service_api/app/app.py)
- [completion.py](file://api/controllers/service_api/app/completion.py)
- [conversation.py](file://api/controllers/service_api/app/conversation.py)
- [workflow.py](file://api/controllers/service_api/app/workflow.py)
</cite>

## 目录
1. [简介](#简介)
2. [API概览](#api概览)
3. [认证机制](#认证机制)
4. [应用信息与元数据](#应用信息与元数据)
5. [补全应用API](#补全应用api)
6. [对话应用API](#对话应用api)
7. [工作流应用API](#工作流应用api)
8. [错误处理](#错误处理)
9. [异步操作与状态管理](#异步操作与状态管理)
10. [代码示例](#代码示例)

## 简介

Dify应用服务API提供了一组RESTful接口，用于与Dify平台中的各类应用进行交互。通过这些API，开发者可以创建、配置和管理对话应用、补全应用和工作流应用。本文档详细说明了所有相关端点的使用方法，包括HTTP方法、URL模式、请求/响应结构和认证要求。

API支持多种应用模式，包括：
- **补全应用**：基于输入生成文本内容
- **对话应用**：支持多轮对话交互
- **工作流应用**：执行复杂的工作流任务
- **高级聊天应用**：结合对话和工作流功能

**Section sources**
- [app.py](file://api/controllers/service_api/app/app.py#L1-L95)
- [completion.py](file://api/controllers/service_api/app/completion.py#L1-L255)
- [conversation.py](file://api/controllers/service_api/app/conversation.py#L1-L248)
- [workflow.py](file://api/controllers/service_api/app/workflow.py#L1-L313)

## API概览

Dify应用服务API提供了一系列端点，用于管理应用的生命周期和交互。所有API端点都遵循RESTful设计原则，使用标准的HTTP方法和状态码。

### 支持的应用模式

| 应用模式 | 描述 |
|---------|------|
| `completion` | 补全应用，用于生成文本内容 |
| `chat` | 基本对话应用，支持多轮对话 |
| `agent_chat` | 代理对话应用，具有智能决策能力 |
| `workflow` | 工作流应用，执行预定义的工作流 |
| `advanced_chat` | 高级聊天应用，结合对话和工作流功能 |

### 通用请求头

所有API请求都需要包含以下头信息：

```http
Authorization: Bearer <API_TOKEN>
Content-Type: application/json
```

### 响应结构

成功的API响应通常包含以下结构：

```json
{
  "result": "success",
  "data": { /* 响应数据 */ }
}
```

错误响应结构：

```json
{
  "code": "error_code",
  "message": "错误描述"
}
```

**Section sources**
- [app.py](file://api/controllers/service_api/app/app.py#L1-L95)
- [completion.py](file://api/controllers/service_api/app/completion.py#L1-L255)

## 认证机制

Dify应用服务API使用Bearer Token进行认证。每个应用都有一个唯一的API令牌，用于验证请求的合法性。

### 认证流程

1. 在Dify控制台中获取应用的API令牌
2. 在每个API请求的Authorization头中包含该令牌
3. 服务器验证令牌的有效性

### 令牌验证

API使用`@validate_app_token`装饰器来验证应用令牌。该装饰器会：
- 验证令牌格式
- 检查令牌是否与请求的应用匹配
- 验证令牌是否有效

```python
@validate_app_token
def get(self, app_model: App):
    # 处理经过认证的请求
    pass
```

对于需要用户上下文的API，还可以指定用户参数的获取位置：

```python
@validate_app_token(fetch_user_arg=FetchUserArg(fetch_from=WhereisUserArg.JSON, required=True))
def post(self, app_model: App, end_user: EndUser):
    # 处理包含用户信息的请求
    pass
```

**Section sources**
- [app.py](file://api/controllers/service_api/app/app.py#L1-L95)
- [completion.py](file://api/controllers/service_api/app/completion.py#L1-L255)
- [conversation.py](file://api/controllers/service_api/app/conversation.py#L1-L248)

## 应用信息与元数据

### 获取应用信息

获取应用的基本信息，包括名称、描述、标签和模式。

**端点**: `GET /info`  
**认证**: 是  
**响应码**: 
- 200: 成功
- 401: 未授权
- 404: 应用未找到

**响应示例**:
```json
{
  "name": "客服助手",
  "description": "智能客服对话应用",
  "tags": ["客服", "AI"],
  "mode": "chat",
  "author_name": "张三"
}
```

### 获取应用元数据

获取应用的配置和设置信息。

**端点**: `GET /meta`  
**认证**: 是  
**响应码**: 
- 200: 成功
- 401: 未授权
- 404: 应用未找到

### 获取应用参数

获取应用的输入参数和配置。

**端点**: `GET /parameters`  
**认证**: 是  
**响应码**: 
- 200: 成功
- 401: 未授权
- 404: 应用未找到

根据应用模式，返回不同的参数结构：
- 对于工作流和高级聊天应用，返回工作流配置
- 对于其他应用，返回应用模型配置

**Section sources**
- [app.py](file://api/controllers/service_api/app/app.py#L1-L95)

## 补全应用API

补全应用API用于生成文本内容，支持阻塞和流式响应模式。

### 创建补全

生成基于输入的文本内容。

**端点**: `POST /completion-messages`  
**认证**: 是  
**请求体**:

```json
{
  "inputs": {
    "variable1": "value1"
  },
  "query": "生成一段关于AI的描述",
  "files": [],
  "response_mode": "blocking"
}
```

**参数说明**:
- `inputs`: 输入参数字典
- `query`: 查询字符串
- `files`: 附件列表
- `response_mode`: 响应模式（blocking/streaming）

**响应码**:
- 200: 成功
- 400: 参数错误
- 401: 未授权
- 500: 服务器错误

### 停止补全任务

停止正在运行的补全任务。

**端点**: `POST /completion-messages/{task_id}/stop`  
**认证**: 是  
**路径参数**:
- `task_id`: 要停止的任务ID

**响应码**:
- 200: 成功
- 401: 未授权
- 404: 任务未找到

**Section sources**
- [completion.py](file://api/controllers/service_api/app/completion.py#L1-L255)

## 对话应用API

对话应用API支持多轮对话交互，包括消息发送、会话管理和变量操作。

### 发送聊天消息

在对话会话中发送消息。

**端点**: `POST /chat-messages`  
**认证**: 是  
**请求体**:

```json
{
  "inputs": {
    "name": "张三"
  },
  "query": "你好",
  "conversation_id": "uuid",
  "response_mode": "streaming",
  "auto_generate_name": true
}
```

**参数说明**:
- `conversation_id`: 现有会话ID（可选）
- `auto_generate_name`: 是否自动生成会话名称

**响应码**:
- 200: 成功
- 400: 参数错误
- 401: 未授权
- 404: 会话未找到

### 停止聊天消息生成

停止正在生成的聊天消息。

**端点**: `POST /chat-messages/{task_id}/stop`  
**认证**: 是  
**路径参数**:
- `task_id`: 要停止的任务ID

### 会话管理

#### 列出会话

获取用户的会话列表。

**端点**: `GET /conversations`  
**认证**: 是  
**查询参数**:
- `last_id`: 分页起始ID
- `limit`: 返回数量
- `sort_by`: 排序方式

#### 删除会话

删除指定的会话。

**端点**: `DELETE /conversations/{c_id}`  
**认证**: 是  
**路径参数**:
- `c_id`: 会话ID

#### 重命名会话

修改会话名称。

**端点**: `POST /conversations/{c_id}/name`  
**认证**: 是  
**请求体**:
```json
{
  "name": "新会话名称",
  "auto_generate": false
}
```

### 会话变量管理

#### 获取会话变量

获取会话中的变量列表。

**端点**: `GET /conversations/{c_id}/variables`  
**认证**: 是

#### 更新会话变量

修改会话变量的值。

**端点**: `PUT /conversations/{c_id}/variables/{variable_id}`  
**认证**: 是  
**请求体**:
```json
{
  "value": "新值"
}
```

**Section sources**
- [conversation.py](file://api/controllers/service_api/app/conversation.py#L1-L248)

## 工作流应用API

工作流应用API用于执行和管理工作流任务。

### 执行工作流

运行工作流应用。

**端点**: `POST /workflows/run`  
**认证**: 是  
**请求体**:

```json
{
  "inputs": {
    "input1": "value1"
  },
  "files": [],
  "response_mode": "blocking"
}
```

### 按ID执行工作流

执行特定ID的工作流版本。

**端点**: `POST /workflows/{workflow_id}/run`  
**认证**: 是  
**路径参数**:
- `workflow_id`: 工作流ID

### 获取工作流运行详情

获取工作流运行的详细信息。

**端点**: `GET /workflows/run/{workflow_run_id}`  
**认证**: 是  
**响应结构**:

```json
{
  "id": "运行ID",
  "workflow_id": "工作流ID",
  "status": "状态",
  "inputs": {},
  "outputs": {},
  "error": "错误信息",
  "total_steps": 5,
  "total_tokens": 1000,
  "created_at": 1700000000,
  "finished_at": 1700000060,
  "elapsed_time": 60.5
}
```

### 停止工作流任务

停止正在运行的工作流任务。

**端点**: `POST /workflows/tasks/{task_id}/stop`  
**认证**: 是  
**路径参数**:
- `task_id`: 任务ID

### 获取工作流日志

获取工作流执行日志。

**端点**: `GET /workflows/logs`  
**认证**: 是  
**查询参数**:
- `keyword`: 关键词搜索
- `status`: 运行状态过滤
- `created_at__before`: 开始时间前
- `created_at__after`: 开始时间后
- `page`: 页码
- `limit`: 每页数量

**Section sources**
- [workflow.py](file://api/controllers/service_api/app/workflow.py#L1-L313)

## 错误处理

API使用标准的HTTP状态码和自定义错误码来处理错误情况。

### 通用错误码

| 状态码 | 错误码 | 描述 |
|-------|-------|------|
| 400 | bad_request | 请求参数错误 |
| 401 | unauthorized | 未授权，API令牌无效 |
| 404 | not_found | 资源未找到 |
| 429 | rate_limit_exceeded | 请求频率超限 |
| 500 | internal_server_error | 服务器内部错误 |

### 特定错误类型

- `app_unavailable`: 应用不可用
- `not_chat_app`: 不是聊天应用
- `not_workflow_app`: 不是工作流应用
- `completion_request_error`: 补全请求错误
- `conversation_completed`: 会话已完成
- `provider_not_initialize`: 服务提供商未初始化
- `provider_quota_exceeded`: 服务提供商配额超限
- `model_currently_not_support`: 模型当前不支持

### 错误响应示例

```json
{
  "code": "unauthorized",
  "message": "Unauthorized - invalid API token"
}
```

**Section sources**
- [completion.py](file://api/controllers/service_api/app/completion.py#L1-L255)
- [conversation.py](file://api/controllers/service_api/app/conversation.py#L1-L248)
- [workflow.py](file://api/controllers/service_api/app/workflow.py#L1-L313)

## 异步操作与状态管理

Dify API支持异步操作，允许客户端启动长时间运行的任务并查询其状态。

### 异步响应模式

支持两种响应模式：
- **阻塞模式** (`blocking`): 等待任务完成并返回结果
- **流式模式** (`streaming`): 实时返回生成的文本片段

```json
{
  "response_mode": "streaming"
}
```

### 任务状态管理

通过任务ID可以管理和监控异步任务：

1. 启动任务，获取任务ID
2. 使用任务ID查询任务状态
3. 必要时停止任务执行

### 停止任务

所有类型的异步任务都支持停止操作：

```http
POST /completion-messages/{task_id}/stop
POST /chat-messages/{task_id}/stop
POST /workflows/tasks/{task_id}/stop
```

停止操作通过`AppQueueManager.set_stop_flag`实现，设置停止标志后，任务会在下一个检查点终止。

**Section sources**
- [completion.py](file://api/controllers/service_api/app/completion.py#L1-L255)
- [conversation.py](file://api/controllers/service_api/app/conversation.py#L1-L248)
- [workflow.py](file://api/controllers/service_api/app/workflow.py#L1-L313)

## 代码示例

### Python客户端示例

```python
import requests
import json

class DifyClient:
    def __init__(self, api_key, base_url="https://api.dify.ai"):
        self.api_key = api_key
        self.base_url = base_url
        self.headers = {
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json"
        }
    
    def create_completion(self, app_id, inputs, query, response_mode="blocking"):
        url = f"{self.base_url}/v1/apps/{app_id}/completion-messages"
        payload = {
            "inputs": inputs,
            "query": query,
            "response_mode": response_mode
        }
        response = requests.post(url, headers=self.headers, json=payload)
        return response.json()
    
    def send_chat_message(self, app_id, inputs, query, conversation_id=None):
        url = f"{self.base_url}/v1/apps/{app_id}/chat-messages"
        payload = {
            "inputs": inputs,
            "query": query,
            "response_mode": "streaming"
        }
        if conversation_id:
            payload["conversation_id"] = conversation_id
        response = requests.post(url, headers=self.headers, json=payload)
        return response.json()

# 使用示例
client = DifyClient("your-api-key")
result = client.create_completion(
    app_id="app-123",
    inputs={"name": "张三"},
    query="写一封欢迎邮件",
    response_mode="blocking"
)
print(result)
```

### Node.js客户端示例

```javascript
const axios = require('axios');

class DifyClient {
    constructor(apiKey, baseUrl = 'https://api.dify.ai') {
        this.apiKey = apiKey;
        this.baseUrl = baseUrl;
        this.client = axios.create({
            baseURL: baseUrl,
            headers: {
                'Authorization': `Bearer ${apiKey}`,
                'Content-Type': 'application/json'
            }
        });
    }

    async createCompletion(appId, inputs, query, responseMode = 'blocking') {
        try {
            const response = await this.client.post(`/v1/apps/${appId}/completion-messages`, {
                inputs,
                query,
                response_mode: responseMode
            });
            return response.data;
        } catch (error) {
            throw new Error(`API请求失败: ${error.response?.data?.message || error.message}`);
        }
    }

    async getConversationList(appId, lastId = null, limit = 20) {
        const params = {};
        if (lastId) params.last_id = lastId;
        if (limit) params.limit = limit;

        const response = await this.client.get(`/v1/apps/${appId}/conversations`, { params });
        return response.data;
    }
}

// 使用示例
const client = new DifyClient('your-api-key');
client.createCompletion('app-123', {name: '张三'}, '生成一段文本')
    .then(result => console.log(result))
    .catch(error => console.error(error));
```

**Section sources**
- [completion.py](file://api/controllers/service_api/app/completion.py#L1-L255)
- [conversation.py](file://api/controllers/service_api/app/conversation.py#L1-L248)
- [workflow.py](file://api/controllers/service_api/app/workflow.py#L1-L313)