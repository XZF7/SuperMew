# SuperMew 项目总结

## 一、快速启动

### 前置条件
- Python 3.12+
- uv（包管理）
- Docker Desktop

### 启动步骤

```bash
# 1. 安装依赖（首次）
uv sync

# 2. 启动 Docker Desktop（手动打开应用）

# 3. 启动基础服务（PostgreSQL + Redis + Milvus）
docker compose up -d

# 4. 启动后端
uv run python backend/app.py
```

访问地址：
- 前端页面：http://127.0.0.1:8000/
- API 文档：http://127.0.0.1:8000/docs

### 环境变量（.env 关键配置）

```env
ARK_API_KEY=你的火山引擎Key
MODEL=doubao-seed-2-0-pro-260215
GRADE_MODEL=doubao-seed-2-0-lite-260215
FAST_MODEL=doubao-seed-1-6-flash-250828
BASE_URL=https://ark.cn-beijing.volces.com/api/v3

EMBEDDING_MODEL=BAAI/bge-m3
EMBEDDING_DEVICE=cpu
DENSE_EMBEDDING_DIM=1024

MILVUS_HOST=127.0.0.1
MILVUS_PORT=19530
MILVUS_COLLECTION=embeddings_collection

DATABASE_URL=postgresql+psycopg2://postgres:postgres@127.0.0.1:5432/langchain_app
REDIS_URL=redis://127.0.0.1:6379/0

JWT_SECRET_KEY=your-secret
ADMIN_INVITE_CODE=supermew-admin-2026
JWT_ALGORITHM=HS256
JWT_EXPIRE_MINUTES=1440
PASSWORD_PBKDF2_ROUNDS=310000
```

git add .
git commit -m "你的提交信息"
git push origin main

---

## 二、技术栈

| 层次 | 技术 |
|------|------|
| 后端框架 | FastAPI + Uvicorn |
| AI 框架 | LangChain Agents + LangGraph |
| LLM | 火山引擎 ARK（豆包系列模型） |
| 本地嵌入 | BAAI/bge-m3（HuggingFace，本地推理） |
| 稀疏检索 | 手写 BM25（中英混合分词，统计持久化） |
| 向量数据库 | Milvus（HNSW 稠密索引 + 稀疏倒排索引） |
| 混合检索融合 | RRF（Reciprocal Rank Fusion） |
| 精排 | Jina Reranker v3（可选） |
| 关系数据库 | PostgreSQL + SQLAlchemy ORM |
| 缓存 | Redis（会话缓存 + 父文档缓存） |
| 鉴权 | JWT + PBKDF2-SHA256 密码哈希 + RBAC |
| 前端 | Vue 3（CDN）+ marked + highlight.js |
| 流式输出 | SSE（Server-Sent Events）+ ReadableStream |
| 容器化 | Docker Compose |
| 包管理 | uv + pyproject.toml |

---

## 三、简历级别撰写

### 项目名称
**SuperMew — 基于 LangChain Agent 的混合检索 RAG 智能问答系统**

### 项目描述（一句话）
基于 FastAPI + LangChain + Milvus 构建的全栈 RAG 智能问答系统，支持文档上传、混合检索、流式对话与用户权限管理。

### 核心亮点（简历条目格式）

- 实现**三级滑动窗口分块（L1/L2/L3）+ Auto-merging 检索**策略，仅叶子块写入 Milvus，父块存 PostgreSQL，检索时自动向上合并，兼顾精度与上下文完整性

- 构建**稠密 + BM25 稀疏双塔混合检索**，使用 Milvus Hybrid Search + RRF 融合排序，接入 Jina Reranker 精排，相比纯向量检索召回质量显著提升

- 设计**跨线程异步事件调度架构**（Global Loop Capture + call_soon_threadsafe），解决 LangChain 同步工具在 ThreadPoolExecutor 中无法向 asyncio 队列推送事件的问题，实现 RAG 检索过程实时推送前端

- 实现**流式输出（SSE）+ 前端打字机效果**，基于 `agent.astream(stream_mode="messages")` 逐 token 推送，前端 ReadableStream 解析，支持 AbortController 随时终止生成

- 实现**BM25 统计增量持久化**，词表与文档频次落盘至 `bm25_state.json`，文档入库/删除时增量更新，与 Milvus 向量库保持一致，避免重启后统计失效

- 构建**完整用户体系**：JWT 鉴权 + RBAC 权限控制（admin/user）+ PBKDF2-SHA256 密码存储，管理员可管理知识库，普通用户仅可访问自己的会话

- 引入 **Redis 多维度缓存**（会话消息、会话列表、父文档），配合 PostgreSQL 持久化，高频读取走缓存，写入后主动失效保证一致性

- 实现**查询重写体系**（Step-Back / HyDE / Complex 路由）+ 相关性评分门控，低质量召回自动触发重写二次检索，提升复杂问题回答准确率

### 技术关键词（用于 ATS 筛选）
Python · FastAPI · LangChain · LangGraph · Milvus · PostgreSQL · Redis · RAG · Hybrid Search · BM25 · HuggingFace · SSE · JWT · Docker · Vue 3

后端框架

FastAPI — Python Web 框架，负责接收前端请求、返回响应，类似 Spring Boot / Express
Uvicorn — 运行 FastAPI 的服务器，相当于 Tomcat
AI 相关

LangChain Agents — AI 框架，让 LLM 能自主决定"用哪个工具"来回答问题
LangGraph — LangChain 的扩展，把 Agent 的执行流程组织成有向图，更可控
火山引擎 ARK（豆包） — 实际调用的大语言模型，负责理解问题、生成回答
检索相关

BAAI/bge-m3 — 本地运行的嵌入模型，把文字转成向量数字，用于语义相似度计算
BM25 — 传统关键词检索算法，按词频匹配，弥补向量检索对关键词不敏感的缺陷
Milvus — 向量数据库，专门存储和检索向量，支持同时做稠密+稀疏混合检索
RRF — 融合算法，把两路检索结果合并排序，不需要手动调权重
Jina Reranker — 精排模型，对召回结果再做一次更精准的排序
数据存储

PostgreSQL — 关系型数据库，存用户信息、聊天记录、父级文档块
SQLAlchemy — Python 操作数据库的 ORM 库，不用写 SQL，用 Python 对象操作数据库
Redis — 内存缓存数据库，存热点数据（会话、父文档），加速读取
安全

JWT — 登录后颁发的 Token，每次请求带上它来证明身份，无需 Session
PBKDF2-SHA256 — 密码哈希算法，存数据库的不是明文密码而是哈希值
RBAC — 基于角色的权限控制，admin 和 user 能做的事不一样
前端

Vue 3（CDN） — 前端框架，负责页面渲染和交互，通过 CDN 引入不需要打包
marked — 把 Markdown 文本转成 HTML，用于渲染 AI 回答
highlight.js — 代码高亮库，让代码块有颜色
通信 & 部署

SSE（Server-Sent Events） — 服务器向浏览器单向推送数据的协议，实现打字机流式效果
Docker Compose — 一键启动多个服务（PostgreSQL、Redis、Milvus）的容器编排工具
uv — 比 pip 更快的 Python 包管理工具