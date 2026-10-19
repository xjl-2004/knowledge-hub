# Knowledge Hub · 企业级知识库管理平台

一个面向企业内部的**知识库管理 + RAG 智能问答 + 知识图谱**一体化平台。支持文档上传解析、异步构建检索索引、全文/语义/图谱多路检索、以及带引用溯源的 AI 问答。

后端采用 NestJS 模块化划分，通过 RabbitMQ 解耦文档发布后的重处理管线；前端为 React + Ant Design 管理台。

## 核心特性

- **文档全生命周期**：上传 → 解析（PDF / DOCX / PPTX / XLSX / 纯文本）→ 编辑 → 审核 → 发布 / 归档 / 下架，支持正文转 Markdown 落库。
- **异步处理管线**：文档发布后经 RabbitMQ 触发 RAG 分块嵌入、知识图谱抽取、ES 索引构建，主流程与重处理解耦。
- **混合检索（Hybrid Retrieval）**：向量语义召回 + ES 全文检索 + 重排序（Reranker），支持查询改写。
- **RAG 问答**：流式对话、多轮上下文、长短记忆、联网搜索兜底，回答附带来源引用。
- **知识图谱**：基于 Neo4j 抽取实体与关系，提供图谱检索、实体 / 关系浏览与可视化。
- **RBAC 权限体系**：用户 / 角色 / 权限 / 团队多维授权，配套审核工作台。
- **认证与安全**：JWT 鉴权、邮箱激活、密码重置（Redis 验证码）。

## 技术栈

**前端**
- React 19 + TypeScript + Vite 8
- Ant Design 6、ECharts 6、React Router 7
- react-markdown（回答渲染）、AI SDK React（流式对话）
- oxlint

**后端**
- NestJS 11 + TypeScript
- 数据层：TypeORM（PostgreSQL / pgvector）、Mongoose（MongoDB）
- 检索层：Elasticsearch 8.17（IK 中文分词）、Neo4j（知识图谱）
- 消息队列：RabbitMQ（amqp）
- 缓存 / 验证码：Redis（ioredis）
- AI：LangChain / AI SDK，Embedding（OpenAI 兼容，未配 Key 时降级本地哈希向量）
- 对象存储：RustFS（S3 兼容）
- 文档解析：pdf-parse、mammoth、exceljs、officeparser、turndown
- 其他：JWT / Passport、bcryptjs、snowflake-id、nodemailer

## 系统架构

```
                    ┌──────────────────────────────┐
   浏览器  ───────▶ │  Frontend (React + AntD)     │
                    └──────────────┬───────────────┘
                                   │ /api 代理
                    ┌──────────────▼───────────────┐
                    │  Backend (NestJS)            │
                    │  auth / user / role / team   │
                    │  document / search / graph   │
                    │  ai (RAG chat)               │
                    └───┬───────┬───────┬───────┬──┘
                        │       │       │       │
              ┌─────────▼─┐ ┌───▼───┐ ┌─▼────┐ ┌▼──────┐
              │ PostgreSQL│ │MongoDB│ │ Redis│ │ Neo4j │
              │ +pgvector │ └───────┘ └──────┘ └───────┘
              └───────────┘
                        │ 发布后异步管线
              ┌─────────▼─────────┐
              │  RabbitMQ         │
              └───┬──────┬────────┘
                  │      │
          ┌───────▼──┐ ┌─▼──────────┐
          │  RAG     │ │  KG 抽取   │
          │ 分块+嵌入│ │ +图谱构建  │
          └────┬─────┘ └─────┬──────┘
               │             │
        ┌──────▼──────┐ ┌────▼──────┐
        │Elasticsearch│ │  Neo4j    │
        └─────────────┘ └───────────┘
```

## 目录结构

```
knowledge-hub/
├── knowledge-hub-backend/    # NestJS 后端
│   ├── src/
│   │   ├── auth/             # JWT 鉴权、邮箱激活、密码重置
│   │   ├── user/ role/ team/ # RBAC 用户、角色、权限、团队
│   │   ├── document/         # 文档 CRUD、解析器、审核
│   │   ├── pipeline/         # 分块、嵌入、抽取、图谱、索引
│   │   ├── mq/               # RabbitMQ 发布/消费
│   │   ├── ai/               # RAG 对话、混合检索、重排、记忆
│   │   ├── search/ graph/    # 全文检索 / 知识图谱
│   │   └── storage/ redis/   # 对象存储 / 缓存
│   ├── docker-compose.yml    # 中间件编排
│   └── .env.example
└── knowledge-hub-frontend/   # React 管理台
    └── src/pages/            # 登录、文档、检索、图谱、后台管理
```

## 快速开始

### 0. 前置依赖

- Node.js 20+
- pnpm
- Docker Desktop

### 1. 启动中间件

```bash
cd knowledge-hub-backend
docker compose up -d
```

### 2. 启动后端

```bash
cd knowledge-hub-backend
pnpm install
copy .env.example .env        # Windows；macOS/Linux 用 cp .env.example .env
pnpm start:dev
```

后端默认运行在 `http://localhost:3000`。

### 3. 启动前端

```bash
cd knowledge-hub-frontend
pnpm install
pnpm dev
```

浏览器打开 `http://localhost:5173`，开发时 `/api` 会代理到 `3000`。

### 4. 预置账号

| 账号 | 密码 | 用途 |
| --- | --- | --- |
| user | 123456 | 文档与检索 |
| admin | 123456 | 系统管理 |
| reviewer | 123456 | 审核 |

## 依赖服务端口

| 服务 | 端口 | 说明 |
| --- | --- | --- |
| Backend | 3000 | NestJS API |
| Frontend | 5173 | Vite 开发服务器 |
| PostgreSQL (pgvector) | 5432 | 主库 / 向量存储 |
| MongoDB | 27017 | 文档内容存储 |
| Redis | 6379 | 缓存 / 验证码 |
| RabbitMQ | 5672 / 15672 | 消息队列 / 管理台 |
| Elasticsearch | 9200 | 全文检索 |
| Kibana | 5601 | ES 控制台 |
| Neo4j | 7474 / 7687 | 图谱 / Bolt |
| RustFS | 9000 / 9001 | 对象存储 / 控制台 |
| pgAdmin | 8088 | PostgreSQL 控制台 |
| mongo-express | 8081 | MongoDB 控制台 |
| RedisInsight | 5540 | Redis 控制台 |

## 主要接口（节选）

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| POST | `/ai/chat` | RAG 流式问答 |
| POST | `/rag/search` | 仅检索（不生成） |
| POST | `/search` | 全文检索 |
| POST | `/documents` | 新建文档 |
| POST | `/documents/:id/upload` | 上传并解析 |
| GET | `/graph/...` | 图谱检索 / 实体 / 关系 |

## 说明

- 仓库中 `test-files/` 含含个人信息的测试样例，已通过 `.gitignore` 排除，不随仓库发布。
- 所有敏感配置（数据库密码、API Key 等）请写入本地 `.env`，不要提交到仓库。

## License

UNLICENSED（私有项目，未开源授权）。
