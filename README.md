# Hua AI Agent

Hua AI Agent 是一个面向 AI 应用开发的 Spring Boot + Vue 全栈项目，集成了 AI 恋爱顾问、RAG 知识库问答、ReAct 智能体、工具调用、SSE 流式输出和 MCP 地图服务。

项目后端基于 Java 21、Spring Boot 3.5、Spring AI 和 Spring AI Alibaba，前端基于 Vue 3、Vite 与 Vue Router。前端通过 SSE 实时展示模型回复与智能体执行进度。

## 功能特性

- **AI 恋爱顾问**：围绕单身、恋爱、已婚场景提供对话建议，按 `chatId` 维护最近 20 条消息的会话上下文。
- **流式对话**：恋爱顾问和超级智能体均支持 SSE 流式输出，前端可以边生成边显示，并支持主动停止请求。
- **RAG 知识库**：读取 `src/main/resources/document/*.md`，按文件名中的状态标签（单身、恋爱、已婚）补充元数据，支持 SimpleVectorStore 和 PGVector 两种向量存储方案。
- **ReAct 超级智能体**：智能体按“思考 -> 选择工具 -> 执行 -> 汇总”的流程处理复杂任务，最多执行 20 步，单任务默认最长 5 分钟。
- **重复调用检测**：连续出现相同或高度相似的工具调用时，注入破循环提示；仍未恢复时强制结束任务，避免无限循环。
- **内置工具链**：支持网页搜索、网页抓取、资源下载、文件读写、终端命令、PDF 生成和任务终止。
- **MCP 集成**：项目已配置高德地图 MCP Server，并封装了 MCP 工具调用能力。
- **统一响应与异常处理**：普通接口使用 `BaseResponse` 包装，流式接口独立处理错误，避免连接中途异常导致响应不可读。

## 技术栈

| 层级 | 技术 |
| --- | --- |
| 后端 | Java 21、Spring Boot 3.5、Spring Web |
| AI | Spring AI、Spring AI Alibaba、DashScope |
| 智能体 | ReAct、Tool Calling、多步骤执行 |
| RAG | Markdown Reader、Embedding、SimpleVectorStore、PGVector |
| MCP | Spring AI MCP Client、高德地图 MCP Server |
| 数据与工具 | PostgreSQL + pgvector、Jsoup、Hutool、iText |
| 前端 | Vue 3、Vite 5、Vue Router、Axios、Lucide Vue |
| 文档 | Knife4j / Springdoc OpenAPI |

## 项目结构

```text
hua-ai-agent/
├── src/main/java/com/huahua/huaaiagent/
│   ├── agent/                  # ReAct 智能体、工具调用与循环检测
│   ├── app/                    # LoveApp 等业务应用
│   ├── advisor/                # AI Advisor 与日志增强
│   ├── chatmemory/             # 基于文件的对话记忆实现
│   ├── config/                 # CORS、JSON 等配置
│   ├── controller/             # HTTP 与 SSE 接口
│   ├── rag/                    # 文档加载、向量存储与 RAG 配置
│   ├── tools/                  # 智能体可调用的本地工具
│   └── HuaAiAgentApplication.java
├── src/main/resources/
│   ├── application.yml         # 主配置，默认激活 local profile
│   ├── application-local.yml   # 本地敏感配置，已被 Git 忽略
│   ├── document/               # 恋爱知识库 Markdown 文档
│   └── mcp-servers.json        # MCP Server 配置
├── src/test/                   # 后端测试与集成示例
├── HuaAIAgent-front/           # Vue 3 前端
├── tmp/                        # 工具生成的文件、下载资源和 PDF
├── pom.xml
└── package.json                # 根目录依赖，目前用于安装高德 MCP Server
```

## 环境要求

- JDK 21
- Maven 3.9+
- Node.js 18+
- npm 9+
- PostgreSQL（使用 PGVector 功能时需要）
- DashScope API Key
- SearchAPI API Key（网页搜索工具使用）
- 高德地图 API Key（MCP 地图服务使用）

## 配置

后端默认激活 `local` profile，因此启动前需要准备：

```text
src/main/resources/application-local.yml
```

该文件已加入 `.gitignore`，不要将真实密钥提交到仓库。可参考下面的结构，通过环境变量注入敏感信息：

```yaml
spring:
  ai:
    dashscope:
      api-key: ${DASHSCOPE_API_KEY}
      chat:
        options:
          model: qwen-plus
  datasource:
    url: jdbc:postgresql://localhost:5432/hua_ai_agent
    username: ${POSTGRES_USERNAME:postgres}
    password: ${POSTGRES_PASSWORD}

search-api:
  api-key: ${SEARCH_API_KEY}

# 需要将 Markdown 文档初始化到 PGVector 时开启
# app:
#   vector-store:
#     initialize: true
```

配置说明：

- `spring.ai.dashscope.api-key`：DashScope 模型和 Embedding 服务使用的密钥。
- `spring.ai.dashscope.chat.options.model`：对话模型名称，可按账号可用模型调整。
- `spring.datasource.*`：PGVector 使用的 PostgreSQL 连接信息。
- `search-api.api-key`：SearchAPI 的百度搜索接口密钥。
- `app.vector-store.initialize=true`：启动时把知识库文档写入 PGVector。启用前需确保数据库已安装 `vector` 扩展并准备好 `vector_store` 表。

根目录还需要安装 MCP Server 依赖：

```bash
npm install
```

[`src/main/resources/mcp-servers.json`](src/main/resources/mcp-servers.json) 中目前使用本机绝对路径启动高德地图 MCP Server。如果项目目录不同，请同步修改 `args` 中的路径，并将 `AMAP_MAPS_API_KEY` 替换为真实密钥。

前端开发配置位于 [`HuaAIAgent-front/.env`](HuaAIAgent-front/.env)：

```dotenv
VITE_API_BASE_URL=/api
VITE_PROXY_TARGET=http://localhost:8123
```

开发环境默认由 Vite 将 `/api` 请求代理到后端，浏览器侧无需直接处理跨域。

## 启动项目

### 1. 启动后端

在项目根目录执行：

```bash
mvn spring-boot:run
```

后端地址：

```text
http://localhost:8123/api
```

健康检查：

```bash
curl http://localhost:8123/api/health
```

接口文档：

```text
http://localhost:8123/api/doc.html
```

### 2. 启动前端

另开一个终端：

```bash
cd HuaAIAgent-front
npm install
npm run dev
```

前端地址：

```text
http://localhost:5173
```

主要页面：

| 路径 | 页面 |
| --- | --- |
| `/` | AI 应用中心 |
| `/love` | AI 恋爱大师 |
| `/manus` | AI 超级智能体 |

### 3. 构建

后端打包：

```bash
mvn clean package -DskipTests
```

前端构建：

```bash
cd HuaAIAgent-front
npm run build
```

## 接口说明

所有接口都位于 `/api` 上下文路径下。

| 方法 | 路径 | 参数 | 返回 |
| --- | --- | --- | --- |
| GET | `/api/health` | 无 | 统一 JSON 响应 |
| GET | `/api/ai/love_app/chat/sync` | `message`, `chatId` | 同步对话 JSON |
| GET | `/api/ai/love_app/chat/sse` | `message`, `chatId` | SSE 文本流 |
| GET | `/api/ai/love_app/chat/sse/emitter` | `message`, `chatId` | `SseEmitter` 流 |
| GET | `/api/ai/manus/chat` | `message` | SSE 事件流 |

同步对话示例：

```bash
curl "http://localhost:8123/api/ai/love_app/chat/sync?message=我该如何开始一段关系&chatId=demo-session"
```

流式对话示例：

```bash
curl -N "http://localhost:8123/api/ai/love_app/chat/sse?message=最近有些感情困惑&chatId=demo-session"
```

超级智能体示例：

```bash
curl -N "http://localhost:8123/api/ai/manus/chat?message=制定一份周末约会计划并生成PDF"
```

智能体接口会发送以下 SSE 事件：

- `agent_progress`：当前执行了哪个工具，仅包含面向用户的简短进度。
- `agent_final`：最终回答。
- `agent_error`：执行失败时的错误信息。

> 当前 HTTP 控制器直接开放的是基础流式对话和智能体接口。RAG、MCP 等能力已在服务层实现，可通过测试或新增控制器继续暴露。

## 知识库

知识库文件位于：

```text
src/main/resources/document/
```

当前文档按恋爱状态分为：

- `恋爱常见问题和回答 - 单身篇.md`
- `恋爱常见问题和回答 - 恋爱篇.md`
- `恋爱常见问题和回答 - 已婚篇.md`

文档加载器会读取文件名中的两字状态标签并写入 `status` 元数据。新增 Markdown 文档时建议保持相同命名方式，以便按状态进行过滤和检索。

## 工具列表

| 工具 | 方法 | 说明 |
| --- | --- | --- |
| 网页搜索 | `searchWeb` | 调用 SearchAPI 搜索百度结果 |
| 网页抓取 | `scrapeWebPage` | 使用 Jsoup 获取网页 HTML |
| 资源下载 | `downloadResource` | 下载图片等资源到 `tmp/download` |
| 文件读取 | `readFile` | 读取 `tmp/file` 下的文本文件 |
| 文件写入 | `writeFile` | 将内容写入 `tmp/file` |
| 终端命令 | `executeTerminalCommand` | 在 Windows `cmd.exe` 中执行命令 |
| PDF 生成 | `generatePDF` | 生成中文 PDF 到 `tmp/pdf` |
| 任务终止 | `doTerminate` | 通知智能体结束当前任务 |

工具生成的文件统一位于项目根目录的 `tmp/` 下。

## 测试与前端联调

后端测试：

```bash
mvn test
```

部分后端测试会真实调用 DashScope、SearchAPI、数据库或 MCP 服务，运行前需完成本地配置并确保网络可用。

前端测试与构建：

```bash
cd HuaAIAgent-front

# 验证 SSE 增量解析
npm run test:sse

# 构建生产资源
npm run build
```

没有可用后端时，可以使用前端自带的假后端：

```bash
cd HuaAIAgent-front
npm run mock
```

随后在另一个终端启动 `npm run dev`，即可联调流式展示效果。运行真实浏览器检查：

```bash
cd HuaAIAgent-front
npm run test:e2e
```

`test:e2e` 需要本机安装 Chrome 或 Edge，也可以通过 `E2E_BROWSER_PATH` 指定浏览器路径。

## 常见问题

### 启动时报缺少配置项

确认 `src/main/resources/application-local.yml` 存在，并包含 DashScope、SearchAPI 和 PostgreSQL 配置。`application.yml` 默认激活 `local` profile。

### 前端请求失败或跨域

确认后端运行在 `8123` 端口，并优先使用 Vite 代理。若直接将 `VITE_API_BASE_URL` 配置为完整后端地址，则需要同时确认后端 CORS 配置允许当前前端来源。

### SSE 无法实时输出

流式接口链路需要使用 `text/event-stream`，并关闭反向代理的响应缓冲。Vite 配置已经通过 `Accept-Encoding: identity` 和 `Cache-Control: no-cache` 改善本地代理行为；部署到 Nginx 等服务时应关闭对应缓冲。

### MCP Server 无法启动

依次确认根目录已执行 `npm install`、`mcp-servers.json` 中的绝对路径正确，并且 `AMAP_MAPS_API_KEY` 已替换为有效密钥。

## 安全提示

- 不要提交 `application-local.yml`、真实 API Key、数据库密码或其他敏感配置。
- `executeTerminalCommand` 会执行本机终端命令，`readFile` 和 `writeFile` 会访问本地文件系统。请仅在可信环境中使用智能体工具，不要将未受限的接口直接暴露到公网。
- 生产环境应增加鉴权、请求频率限制、命令白名单、文件路径校验和审计日志。
- 网页抓取与资源下载会访问外部 URL，应防范 SSRF、恶意文件和超大响应。

## 许可证

当前仓库未包含开源许可证文件。如需对外分发或商用，请先补充明确的许可证和第三方服务使用条款。
