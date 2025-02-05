# 使用 OpenAI 库调用 Vertex AI 模型的方法

## 概述

借助 **Chat Completions API**，您可以通过 Python 和 REST 版本的 OpenAI 库向 Vertex AI 模型发送请求。如果您已经在使用 OpenAI 库，可以通过此 API 在调用 OpenAI 模型和 Vertex AI 托管模型之间无缝切换，以便比较输出、成本和可扩展性，而无需修改现有代码。对于初次使用的开发者，建议直接调用 **Gemini API** 以获得更高效的支持。

👉 [WildCard | 一分钟注册，轻松订阅海外线上服务](https://bbtdd.com/WildCard)

## 支持的模型

Chat Completions API 支持以下模型：

- **Gemini 模型**：Google 提供的高性能 AI 模型。
- **Model Garden 的自部署模型**：您可以自行部署和管理的 AI 模型。

## 身份验证步骤

### Python 环境配置

1. 安装 OpenAI SDK：
   bash
   pip install openai
   
2. 修改客户端设置或环境配置，以使用 Google 身份验证和 Vertex AI 端点。根据您的需求，选择调用 Gemini 模型或 Model Garden 的自部署模型。

### 获取 Google 凭据

使用 `google-auth` Python SDK 以编程方式获取 Google 凭据：
python
import google.auth
creds, project = google.auth.default()

默认情况下，访问令牌的持续时间为 1 小时。您可以延长其有效期或定期刷新令牌，并更新 `openai.api_key` 变量。

### 初始化客户端
python
client = openai.OpenAI()


## 使用 Chat Completions API 调用模型

### 调用 Gemini 模型

1. 设置 `MODEL_ID` 变量：
   python
   MODEL_ID = "google/gemini-1.5-flash-002"
   
2. 发送请求：
   python
   response = client.chat.completions.create(
       model=MODEL_ID,
       messages=[{"role": "user", "content": "Why is the sky blue?"}]
   )
   print(response.choices[0].message.content)
   

### 调用 Model Garden 的自部署模型

1. 设置 `ENDPOINT` 变量：
   python
   ENDPOINT = "your-endpoint-url"
   
2. 发送请求：
   bash
   curl -X POST \
     -H "Authorization: Bearer $(gcloud auth print-access-token)" \
     -H "Content-Type: application/json" \
     https://us-central1-aiplatform.googleapis.com/v1beta1/projects/${PROJECT_ID}/locations/us-central1/endpoints/${ENDPOINT}/chat/completions \
     -d '{
       "stream": true,
       "messages": [{
         "role": "user",
         "content": "Write a story about a magic backpack."
       }]
     }'
   

## 支持的参数

Chat Completions API 支持以下 OpenAI 参数：

| 参数               | 描述                                                                 |
|--------------------|----------------------------------------------------------------------|
| `messages`         | 支持多种消息类型，如系统消息、用户消息、助手消息等。                   |
| `model`            | 指定模型名称。                                                       |
| `max_tokens`       | 限制生成内容的最大令牌数。                                           |
| `stream`           | 启用流式传输响应。                                                   |
| `temperature`      | 控制生成内容的创意性。                                               |

传递不受支持的参数时，系统会自动忽略。

## 刷新凭据

以下示例展示了如何自动刷新凭据：

python
class OpenAICredentialsRefresher:
    def __init__(self, **kwargs):
        self.client = openai.OpenAI(**kwargs, api_key="DUMMY")
        self.creds, self.project = google.auth.default(
            scopes=["https://www.googleapis.com/auth/cloud-platform"]
        )

    def __getattr__(self, name):
        if not self.creds.valid:
            auth_req = google.auth.transport.requests.Request()
            self.creds.refresh(auth_req)
            self.client.api_key = self.creds.token
        return getattr(self.client, name)


## 后续步骤

- 查看 OpenAI 兼容语法调用 **Inference API** 的示例。
- 了解 **Function Calling API** 的使用方法。
- 深入掌握 Gemini API 的功能和应用场景。