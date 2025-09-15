# Node.js SDK

<cite>
**本文档中引用的文件**  
- [README.md](file://sdks/nodejs-client/README.md)
- [index.js](file://sdks/nodejs-client/index.js)
- [index.d.ts](file://sdks/nodejs-client/index.d.ts)
- [package.json](file://sdks/nodejs-client/package.json)
</cite>

## 目录
1. [简介](#简介)
2. [安装与初始化](#安装与初始化)
3. [核心功能使用](#核心功能使用)
4. [TypeScript类型支持](#typescript类型支持)
5. [异步处理与流式响应](#异步处理与流式响应)
6. [错误处理机制](#错误处理机制)
7. [文件上传与多模态支持](#文件上传与多模态支持)
8. [工作流触发](#工作流触发)
9. [中间件集成示例](#中间件集成示例)
10. [性能优化建议](#性能优化建议)

## 简介
Dify Node.js SDK 是 Dify 平台提供的官方 Node.js 客户端库，旨在简化开发者在 Node.js 应用中集成 Dify AI 功能的过程。该 SDK 封装了与 Dify API 的通信逻辑，支持完成消息生成、聊天交互、工作流执行、文件上传等核心功能。SDK 基于 `axios` 实现 HTTP 请求，支持模块化导入和流式响应处理，适用于 Express、Koa 等主流 Node.js 框架。

**Section sources**
- [README.md](file://sdks/nodejs-client/README.md#L0-L9)
- [index.js](file://sdks/nodejs-client/index.js#L0-L360)

## 安装与初始化
### NPM 安装
通过 npm 安装 Dify Node.js SDK：

```bash
npm install dify-client
```

### 模块导入
支持 ES6 模块语法导入：

```js
import { DifyClient, ChatClient, CompletionClient, WorkflowClient } from 'dify-client';
```

### 客户端初始化
初始化客户端时需提供 API 密钥和可选的基础 URL。

```js
const API_KEY = 'your-api-key-here';
const client = new DifyClient(API_KEY);
const chatClient = new ChatClient(API_KEY, 'https://api.dify.ai/v1');
```

`DifyClient` 是基础类，`ChatClient`、`CompletionClient` 和 `WorkflowClient` 继承自它，分别用于处理聊天、补全和工作流任务。基础 URL 默认为 `https://api.dify.ai/v1`，可根据部署环境自定义。

**Section sources**
- [README.md](file://sdks/nodejs-client/README.md#L10-L48)
- [index.js](file://sdks/nodejs-client/index.js#L78-L85)
- [index.d.ts](file://sdks/nodejs-client/index.d.ts#L32-L34)

## 核心功能使用
### 创建应用与获取参数
通过 `getApplicationParameters` 方法获取应用配置参数：

```js
client.getApplicationParameters('user-id');
```

### 发送对话消息
使用 `ChatClient` 发送聊天消息：

```js
const response = await chatClient.createChatMessage(
  { name: 'Alice' },
  'Hello, how are you?',
  'user-123',
  false,
  null,
  []
);
```

支持指定输入变量、用户标识、会话 ID 和文件附件。

### 触发工作流
使用 `WorkflowClient` 或 `CompletionClient` 触发工作流：

```js
const workflowClient = new WorkflowClient(API_KEY);
workflowClient.run({ input: 'data' }, 'user-123', false);
```

### 文件上传
支持通过 `fileUpload` 方法上传文件：

```js
const formData = new FormData();
formData.append('file', fileBuffer, 'image.png');
formData.append('user', 'user-123');
client.fileUpload(formData);
```

**Section sources**
- [index.js](file://sdks/nodejs-client/index.js#L100-L188)
- [index.js](file://sdks/nodejs-client/index.js#L190-L250)
- [index.js](file://sdks/nodejs-client/index.js#L252-L360)

## TypeScript类型支持
SDK 提供完整的 TypeScript 类型定义，位于 `index.d.ts` 文件中。主要类型包括：

- `DifyFile`: 文件类型，支持本地上传和远程 URL
- `User`: 用户标识类型
- `RequestMethods`: HTTP 请求方法枚举
- 各客户端类的构造函数和方法签名均带有类型注解

使用 TypeScript 可获得智能提示和编译时类型检查，提升开发效率和代码健壮性。

**Section sources**
- [index.d.ts](file://sdks/nodejs-client/index.d.ts#L0-L107)

## 异步处理与流式响应
所有 API 调用均返回 `Promise`，需使用 `async/await` 或 `.then()` 处理异步结果。

### 流式响应处理
对于流式生成场景（如实时聊天），设置 `stream: true`：

```js
const response = await chatClient.createChatMessage(
  {}, 'Query', 'user-123', true
);
const stream = response.data;
stream.on('data', (chunk) => console.log(chunk));
stream.on('end', () => console.log('Stream ended'));
```

SDK 内部根据 `stream` 参数自动设置 `response_type` 为 `stream` 或 `json`，并正确处理数据流。

**Section sources**
- [index.js](file://sdks/nodejs-client/index.js#L87-L100)
- [README.md](file://sdks/nodejs-client/README.md#L30-L38)

## 错误处理机制
SDK 通过 `axios` 抛出 HTTP 错误，建议使用 `try-catch` 捕获：

```js
try {
  const response = await client.getApplicationParameters('user-123');
} catch (error) {
  if (error.response) {
    // 服务器响应错误
    console.error('Status:', error.response.status);
    console.error('Data:', error.response.data);
  } else if (error.request) {
    // 请求未收到响应
    console.error('Request:', error.request);
  } else {
    // 其他错误
    console.error('Error:', error.message);
  }
}
```

错误对象包含 `response`、`request` 和 `message` 字段，便于定位问题。

**Section sources**
- [index.js](file://sdks/nodejs-client/index.js#L87-L100)

## 文件上传与多模态支持
SDK 支持图像等文件上传，用于多模态模型处理。

### 本地文件上传
```js
const file = new DifyLocalFile({
  type: 'image',
  transfer_method: 'local_file',
  upload_file_id: 'file-123'
});
completionClient.createCompletionMessage(inputs, user, false, [file]);
```

### 远程文件上传
```js
const remoteFile = new DifyRemoteFile({
  type: 'image',
  transfer_method: 'remote_url',
  url: 'https://example.com/image.png'
});
```

上传时需设置 `Content-Type: multipart/form-data`，SDK 已在 `fileUpload` 方法中自动配置。

**Section sources**
- [index.d.ts](file://sdks/nodejs-client/index.d.ts#L15-L28)
- [index.js](file://sdks/nodejs-client/index.js#L145-L155)

## 工作流触发
通过 `WorkflowClient.run()` 方法触发工作流执行：

```js
const result = await workflowClient.run(
  { input_var: 'value' },
  'user-123',
  false
);
```

支持阻塞模式（blocking）和流式模式（streaming），通过 `response_mode` 参数控制。可使用 `stop()` 方法终止正在运行的工作流任务。

**Section sources**
- [index.js](file://sdks/nodejs-client/index.js#L330-L360)
- [index.d.ts](file://sdks/nodejs-client/index.d.ts#L100-L107)

## 中间件集成示例
### Express 集成
```js
const express = require('express');
const { ChatClient } = require('dify-client');

const app = express();
app.use(express.json());

const chatClient = new ChatClient('YOUR_API_KEY');

app.post('/chat', async (req, res) => {
  try {
    const { query, user } = req.body;
    const response = await chatClient.createChatMessage({}, query, user);
    res.json(response.data);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

### Koa 集成
```js
const Koa = require('koa');
const Router = require('koa-router');
const { CompletionClient } = require('dify-client');

const app = new Koa();
const router = new Router();
const client = new CompletionClient('API_KEY');

router.post('/completion', async (ctx) => {
  try {
    const { inputs, user } = ctx.request.body;
    const response = await client.createCompletionMessage(inputs, user);
    ctx.body = response.data;
  } catch (error) {
    ctx.status = 500;
    ctx.body = { error: error.message };
  }
});
```

**Section sources**
- [index.js](file://sdks/nodejs-client/index.js#L190-L250)

## 性能优化建议
### 连接池配置
`axios` 支持 HTTP Agent 配置，可启用连接池：

```js
import { Agent } from 'https';
const httpsAgent = new Agent({ keepAlive: true, maxSockets: 50 });
// 在 sendRequest 中传入 headerParams 或扩展客户端
```

### 请求批处理
对于高频请求，建议实现请求批处理队列，避免瞬时高并发。可结合 `Promise.all` 进行并发控制：

```js
const requests = queries.map(query => 
  chatClient.createChatMessage({}, query, 'user-123')
);
const results = await Promise.all(requests);
```

### 重试机制
建议在应用层实现指数退避重试：

```js
async function retryAsync(fn, retries = 3) {
  try {
    return await fn();
  } catch (error) {
    if (retries > 0) {
      await new Promise(r => setTimeout(r, 2 ** (4 - retries) * 1000));
      return retryAsync(fn, retries - 1);
    }
    throw error;
  }
}
```

**Section sources**
- [index.js](file://sdks/nodejs-client/index.js#L87-L100)