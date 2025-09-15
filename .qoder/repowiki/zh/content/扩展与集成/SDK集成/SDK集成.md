# SDK集成

<cite>
**本文档中引用的文件**  
- [README.md](file://sdks/README.md)
- [python-client/README.md](file://sdks/python-client/README.md)
- [python-client/dify_client/client.py](file://sdks/python-client/dify_client/client.py)
- [nodejs-client/README.md](file://sdks/nodejs-client/README.md)
- [nodejs-client/index.js](file://sdks/nodejs-client/index.js)
- [php-client/README.md](file://sdks/php-client/README.md)
- [php-client/dify-client.php](file://sdks/php-client/dify-client.php)
</cite>

## 目录
1. [简介](#简介)
2. [SDK概览](#sdk概览)
3. [Node.js SDK集成](#nodejs-sdk集成)
4. [Python SDK集成](#python-sdk集成)
5. [PHP SDK集成](#php-sdk集成)
6. [核心API调用示例](#核心api调用示例)
7. [认证机制](#认证机制)
8. [错误处理策略](#错误处理策略)
9. [异步操作支持](#异步操作支持)
10. [性能优化建议](#性能优化建议)
11. [各SDK特性与限制](#各sdk特性与限制)
12. [最佳实践](#最佳实践)

## 简介
Dify SDK为开发者提供了便捷的接口，用于集成Dify平台的核心功能。本文档详细介绍了官方提供的Node.js、Python和PHP客户端，涵盖安装方法、初始化配置、基本使用模式以及高级功能。通过这些SDK，开发者可以轻松实现应用创建、对话交互、工作流执行等核心功能，同时支持文件上传、反馈收集、音频处理等多种扩展能力。

## SDK概览
Dify提供了多种语言的SDK客户端，包括Node.js、Python和PHP，每个SDK都封装了Dify API的核心功能，简化了开发者与Dify平台的交互过程。这些SDK支持常见的应用场景，如完成式生成、聊天交互、工作流执行等，并提供了统一的认证机制和错误处理策略。

**Section sources**
- [README.md](file://sdks/README.md)

## Node.js SDK集成

### 安装方法
通过npm安装Dify Node.js SDK：
```bash
npm install dify-client
```

### 初始化配置
```js
import { DifyClient, ChatClient, CompletionClient } from 'dify-client'

const API_KEY = 'your-api-key-here'
const user = 'random-user-id'

// 创建客户端实例
const completionClient = new CompletionClient(API_KEY)
const chatClient = new ChatClient(API_KEY)
```

### 基本使用模式
Node.js SDK提供了多种客户端类，包括`DifyClient`、`CompletionClient`、`ChatClient`和`WorkflowClient`，每个类都封装了特定功能的API调用。

```js
// 创建完成式消息
completionClient.createCompletionMessage({'query': 'Hello'}, user)

// 创建聊天消息（流式响应）
const response = await chatClient.createChatMessage({}, 'Hello', user, true)
const stream = response.data
stream.on('data', data => {
    console.log(data)
})
```

**Section sources**
- [nodejs-client/README.md](file://sdks/nodejs-client/README.md)
- [nodejs-client/index.js](file://sdks/nodejs-client/index.js)

## Python SDK集成

### 安装方法
通过pip安装Dify Python SDK：
```bash
pip install dify-client
```

### 初始化配置
```python
from dify_client import CompletionClient, ChatClient, WorkflowClient

api_key = "your_api_key"
user_id = "user_id"

# 初始化不同类型的客户端
completion_client = CompletionClient(api_key)
chat_client = ChatClient(api_key)
workflow_client = WorkflowClient(api_key)
```

### 基本使用模式
Python SDK提供了清晰的类结构，支持完成式生成、聊天交互和工作流执行等多种模式。

```python
# 完成式生成（阻塞模式）
completion_response = completion_client.create_completion_message(
    inputs={"query": "What's the weather like today?"},
    response_mode="blocking", 
    user="user_id"
)

# 聊天交互（流式模式）
chat_response = chat_client.create_chat_message(
    inputs={}, 
    query="Hello", 
    user="user_id", 
    response_mode="streaming"
)

for line in chat_response.iter_lines(decode_unicode=True):
    line = line.split('data:', 1)[-1]
    if line.strip():
        line = json.loads(line.strip())
        print(line.get('answer'))
```

**Section sources**
- [python-client/README.md](file://sdks/python-client/README.md)
- [python-client/dify_client/client.py](file://sdks/python-client/dify_client/client.py)

## PHP SDK集成

### 安装方法
通过Composer安装Dify PHP SDK：
```json
{
    "require": {
        "guzzlehttp/guzzle": "^7.9"
    },
    "autoload": {
        "files": ["path/to/dify-client.php"]
    }
}
```

### 初始化配置
```php
<?php
require 'vendor/autoload.php';

$apiKey = 'your-api-key-here';
$difyClient = new DifyClient($apiKey);
$completionClient = new CompletionClient($apiKey);
$chatClient = new ChatClient($apiKey);
```

### 基本使用模式
PHP SDK采用面向对象的设计，提供了直观的方法调用接口。

```php
// 创建完成式消息
$response = $completionClient->create_completion_message(
    array("query" => "Who are you?"), 
    "blocking", 
    "user_id"
);

// 创建聊天消息
$response = $chatClient->create_chat_message(
    array(), 
    "Who are you?", 
    "user_id", 
    "blocking", 
    $conversation_id
);

// 文件上传
$fileForUpload = [
    [
        'tmp_name' => '/path/to/file/filename.jpg',
        'name' => 'filename.jpg'
    ]
];
$response = $difyClient->file_upload("user_id", $fileForUpload);
```

**Section sources**
- [php-client/README.md](file://sdks/php-client/README.md)
- [php-client/dify-client.php](file://sdks/php-client/dify-client.php)

## 核心API调用示例

### 应用创建
通过SDK可以轻松创建和管理Dify应用。

```python
# Python示例
parameters = client.get_application_parameters(user="user_id")
print(parameters.json())
```

### 对话交互
支持多种对话模式，包括阻塞式和流式响应。

```js
// Node.js流式对话
const response = await chatClient.createChatMessage({}, query, user, true)
response.data.on('data', data => {
    console.log(data)
})
```

### 工作流执行
通过WorkflowClient可以执行复杂的工作流。

```python
# Python工作流执行
inputs = {
    "context": "previous interaction",
    "user_prompt": "What is the capital of France?"
}
response = client.run(inputs=inputs, response_mode="blocking", user=user_id)
result = json.loads(response.text)
print(result.get("data").get("outputs")["answer"])
```

### 文件上传与视觉模型
支持文件上传和视觉模型调用。

```python
# Python文件上传
with open(file_path, "rb") as file:
    files = {"file": (file_name, file, mime_type)}
    response = dify_client.file_upload("user_id", files)
    result = response.json()
    print(f'upload_file_id: {result.get("id")}')
```

**Section sources**
- [python-client/README.md](file://sdks/python-client/README.md)
- [nodejs-client/README.md](file://sdks/nodejs-client/README.md)
- [php-client/README.md](file://sdks/php-client/README.md)

## 认证机制
所有Dify SDK都采用基于API密钥的认证机制。开发者需要在初始化客户端时提供有效的API密钥，该密钥将作为Bearer Token包含在每个请求的Authorization头中。

```python
# Python认证示例
client = DifyClient(api_key="your_api_key")
```

```js
// Node.js认证示例
const client = new DifyClient('your-api-key-here')
```

```php
// PHP认证示例
$difyClient = new DifyClient('your-api-key-here')
```

API密钥应该安全存储，避免在客户端代码或版本控制系统中明文暴露。

**Section sources**
- [python-client/dify_client/client.py](file://sdks/python-client/dify_client/client.py)
- [nodejs-client/index.js](file://sdks/nodejs-client/index.js)
- [php-client/dify-client.php](file://sdks/php-client/dify-client.php)

## 错误处理策略
Dify SDK提供了统一的错误处理机制，开发者可以通过检查响应状态码和异常来处理各种错误情况。

```python
# Python错误处理
try:
    response = completion_client.create_completion_message(inputs, "blocking", user)
    response.raise_for_status()  # 抛出HTTP错误
    result = response.json()
except requests.exceptions.RequestException as e:
    print(f"请求失败: {e}")
```

```js
// Node.js错误处理
try {
    const response = await chatClient.createChatMessage(inputs, query, user)
    // 处理成功响应
} catch (error) {
    console.error('请求失败:', error.response?.data || error.message)
}
```

常见的错误类型包括认证失败、请求参数错误、资源不存在等，开发者应该根据具体的错误码实施相应的恢复策略。

**Section sources**
- [python-client/dify_client/client.py](file://sdks/python-client/dify_client/client.py)
- [nodejs-client/index.js](file://sdks/nodejs-client/index.js)

## 异步操作支持
Dify SDK全面支持异步操作，特别是流式响应模式，这对于实时应用和大响应处理非常重要。

### 流式响应处理
```python
# Python流式处理
chat_response = chat_client.create_chat_message(
    inputs={}, 
    query="Hello", 
    user="user_id", 
    response_mode="streaming"
)

for line in chat_response.iter_lines(decode_unicode=True):
    if line.strip():
        line_data = json.loads(line.strip())
        print(line_data.get('answer'))
```

```js
// Node.js流式处理
const response = await chatClient.createChatMessage({}, query, user, true)
response.data.on('data', data => {
    console.log(data)
})
response.data.on('end', () => {
    console.log('流式响应结束')
})
```

异步操作允许应用程序在等待API响应的同时保持响应性，特别适合Web应用和移动应用。

**Section sources**
- [python-client/dify_client/client.py](file://sdks/python-client/dify_client/client.py)
- [nodejs-client/index.js](file://sdks/nodejs-client/index.js)

## 性能优化建议
为了获得最佳性能，建议采用以下优化策略：

### 连接复用
尽量复用客户端实例，避免频繁创建和销毁连接。

```python
# 推荐：复用客户端实例
client = CompletionClient(api_key)
# 在整个应用生命周期中使用同一个实例
```

### 批量操作
对于大量相似请求，考虑实现批量处理机制。

### 缓存策略
对频繁访问但不经常变化的数据实施缓存。

```python
# 示例：简单缓存实现
from functools import lru_cache

@lru_cache(maxsize=128)
def get_application_parameters(user):
    return client.get_application_parameters(user)
```

### 超时设置
合理设置请求超时，避免长时间等待。

### 并发控制
在多线程或多进程环境中，合理控制并发请求数量，避免对服务器造成过大压力。

**Section sources**
- [python-client/dify_client/client.py](file://sdks/python-client/dify_client/client.py)
- [nodejs-client/index.js](file://sdks/nodejs-client/index.js)

## 各SDK特性与限制

### Node.js SDK
**特性：**
- 基于axios，支持Promise和async/await
- 完善的流式响应支持
- TypeScript类型定义

**限制：**
- 仅支持Node.js环境
- 需要额外安装axios依赖

### Python SDK
**特性：**
- 基于requests，兼容性好
- 支持同步和异步操作
- 详细的文档和示例

**限制：**
- 需要安装requests库
- 异步支持需要额外配置

### PHP SDK
**特性：**
- 基于Guzzle HTTP客户端
- 简单的文件上传接口
- 易于集成到现有PHP项目

**限制：**
- 需要PHP 7.2或更高版本
- 依赖Guzzle库

**Section sources**
- [python-client/README.md](file://sdks/python-client/README.md)
- [nodejs-client/README.md](file://sdks/nodejs-client/README.md)
- [php-client/README.md](file://sdks/php-client/README.md)

## 最佳实践
### 安全实践
- API密钥应通过环境变量或配置文件管理
- 避免在客户端代码中暴露API密钥
- 定期轮换API密钥

### 错误处理
- 实现全面的错误捕获和日志记录
- 提供用户友好的错误消息
- 实现重试机制处理临时性错误

### 性能监控
- 监控API调用延迟和成功率
- 设置适当的超时和重试策略
- 实施限流以保护后端服务

### 代码组织
- 封装SDK调用，提供清晰的接口
- 实现服务层抽象，便于测试和维护
- 使用类型提示（Python）或类型定义（TypeScript）提高代码质量

### 测试策略
- 编写单元测试验证SDK调用
- 实现集成测试确保端到端功能
- 使用模拟对象进行隔离测试

**Section sources**
- [python-client/README.md](file://sdks/python-client/README.md)
- [nodejs-client/README.md](file://sdks/nodejs-client/README.md)
- [php-client/README.md](file://sdks/php-client/README.md)