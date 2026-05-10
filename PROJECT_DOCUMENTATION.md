# SuperBizAgent 项目完整操作文档

> **版本**: v1.0.0  
> **最后更新**: 2026-04-28  
> **作者**: chief

---

## 📖 目录

1. [项目概述](#项目概述)
2. [技术架构](#技术架构)
3. [项目结构详解](#项目结构详解)
4. [核心模块说明](#核心模块说明)
5. [环境配置](#环境配置)
6. [快速开始](#快速开始)
7. [API 接口文档](#api-接口文档)
8. [运维指南](#运维指南)
9. [常见问题](#常见问题)

---

## 项目概述

SuperBizAgent 是一个基于 Spring Boot + AI Agent 的企业级智能问答与运维系统，包含两大核心功能模块：

### 1. RAG 智能问答系统
- 集成 Milvus 向量数据库实现语义检索
- 支持多轮对话和上下文记忆
- 自动调用工具（时间查询、文档检索、告警查询、日志查询）
- 支持流式输出（SSE）和非流式两种模式

### 2. AIOps 智能运维系统
- 采用 Planner-Executor-Replanner 多 Agent 协作架构
- 自动分析 Prometheus 告警
- 智能查询日志和监控指标
- 自动生成结构化运维报告

---

## 技术架构

### 技术栈

| 技术 | 版本 | 用途 |
|------|------|------|
| Java | 17 | 开发语言 |
| Spring Boot | 3.2.0 | Web 应用框架 |
| Spring AI Alibaba | 1.1.0.0-RC2 | AI Agent 框架 |
| DashScope SDK | 2.17.0 | 阿里云大模型服务 |
| Milvus SDK | 2.6.10 | 向量数据库客户端 |
| Docker Compose | - | 容器编排 |
| Reactor | - | 响应式编程 |

### 架构图

```
┌─────────────────────────────────────────────────┐
│                  前端界面 (Web UI)                 │
│         http://localhost:9900                     │
└──────────────────┬──────────────────────────────┘
                   │ HTTP/SSE
                   ▼
┌─────────────────────────────────────────────────┐
│              ChatController (REST API)            │
│  /api/chat, /api/chat_stream, /api/ai_ops       │
└──────────────────┬──────────────────────────────┘
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
┌──────────────────┐  ┌──────────────────┐
│   ChatService    │  │  AiOpsService    │
│  (ReactAgent)    │  │ (SupervisorAgent)│
└────────┬─────────┘  └────────┬─────────┘
         │                     │
         ├──────────┬──────────┼──────────┐
         ▼          ▼          ▼          ▼
   ┌─────────┐ ┌────────┐ ┌────────┐ ┌────────┐
   │ DateTime│ │DocsTool│ │Metrics │ │LogsTool│
   │ Tools   │ │        │ │ Tools  │ │        │
   └─────────┘ └────────┘ └────────┘ └────────┘
         │          │          │          │
         └──────────┴──────────┴──────────┘
                    │
         ┌──────────▼──────────┐
         │  DashScope LLM API  │
         │  (qwen3-max)        │
         └─────────────────────┘

┌─────────────────────────────────────────────────┐
│              Milvus Vector Database               │
│  - etcd (元数据存储)                              │
│  - MinIO (对象存储)                               │
│  - Milvus Standalone (向量检索引擎)               │
│  - Attu (Web 管理界面)                            │
└─────────────────────────────────────────────────┘
```

---

## 项目结构详解

### 目录树

```
SuperBizAgent/
├── src/main/java/org/example/
│   ├── Main.java                          # Spring Boot 启动类
│   │
│   ├── controller/                        # 控制器层（REST API）
│   │   ├── ChatController.java           # 统一聊天接口 ⭐
│   │   ├── FileUploadController.java     # 文件上传接口
│   │   └── MilvusCheckController.java    # Milvus 健康检查
│   │
│   ├── service/                           # 服务层（业务逻辑）
│   │   ├── ChatService.java              # 对话服务 ⭐
│   │   ├── AiOpsService.java             # AIOps 服务 ⭐
│   │   ├── RagService.java               # RAG 检索增强生成服务
│   │   ├── VectorIndexService.java       # 向量索引服务（文件→向量）
│   │   ├── VectorSearchService.java      # 向量搜索服务
│   │   ├── VectorEmbeddingService.java   # 向量嵌入服务（文本→向量）
│   │   └── DocumentChunkService.java     # 文档分片服务
│   │
│   ├── agent/tool/                        # Agent 工具集
│   │   ├── DateTimeTools.java            # 时间查询工具
│   │   ├── InternalDocsTools.java        # 内部文档检索工具
│   │   ├── QueryMetricsTools.java        # Prometheus 告警查询工具
│   │   └── QueryLogsTools.java           # 腾讯云日志查询工具
│   │
│   ├── config/                            # 配置类
│   │   ├── MilvusConfig.java             # Milvus 客户端配置
│   │   ├── MilvusProperties.java         # Milvus 属性配置
│   │   ├── DashScopeConfig.java          # DashScope API 配置
│   │   ├── DocumentChunkConfig.java      # 文档分片配置
│   │   ├── FileUploadConfig.java         # 文件上传配置
│   │   └── WebConfig.java                # Web 配置
│   │
│   ├── client/                            # 客户端工厂
│   │   └── MilvusClientFactory.java      # Milvus 客户端工厂
│   │
│   ├── constant/                          # 常量定义
│   │   └── MilvusConstants.java          # Milvus 相关常量
│   │
│   ├── dto/                               # 数据传输对象
│   │   ├── AIOpsRequest.java             # AIOps 请求对象
│   │   ├── DocumentChunk.java            # 文档分片对象
│   │   └── FileUploadRes.java            # 文件上传响应对象
│   │
│   └── tool/                              # 其他工具
│       └── DropCollection.java           # 删除集合工具
│
├── src/main/resources/
│   ├── static/                            # 静态资源（前端页面）
│   │   ├── index.html                    # 主页 HTML
│   │   ├── app.js                        # 前端 JavaScript
│   │   └── styles.css                    # 样式表
│   │
│   └── application.yml                   # 应用配置文件 ⭐
│
├── aiops-docs/                            # AIOps 运维文档库
│   ├── cpu_high_usage.md                 # CPU 高占用处理文档
│   ├── memory_high_usage.md              # 内存高占用处理文档
│   ├── disk_high_usage.md                # 磁盘高占用处理文档
│   ├── service_unavailable.md            # 服务不可用处理文档
│   └── slow_response.md                  # 响应缓慢处理文档
│
├── uploads/                               # 文件上传目录（运行时创建）
│
├── vector-database.yml                    # Docker Compose 配置（Milvus）
├── Makefile                               # 自动化构建脚本 ⭐
├── pom.xml                                # Maven 依赖配置
└── README.md                              # 项目简介
```

---

## 核心模块说明

### 1. 控制器层 (controller/)

#### ChatController.java
**作用**: 统一 REST API 入口，处理所有聊天和运维请求

**核心接口**:
- `POST /api/chat` - 普通对话（非流式）
- `POST /api/chat_stream` - 流式对话（SSE）
- `POST /api/ai_ops` - AIOps 智能运维（SSE）
- `POST /api/chat/clear` - 清空会话历史
- `GET /api/chat/session/{sessionId}` - 获取会话信息

**关键特性**:
- 会话管理：使用 `ConcurrentHashMap` 存储会话，支持多用户并发
- 历史窗口：自动维护最近 6 对对话消息
- 线程安全：使用 `ReentrantLock` 保护会话数据
- SSE 流式输出：实时推送 AI 回复内容

#### FileUploadController.java
**作用**: 处理文件上传和向量化

**核心接口**:
- `POST /api/upload` - 上传文件并自动索引到 Milvus

**工作流程**:
1. 接收上传的文件（.txt/.md）
2. 保存到 `uploads/` 目录
3. 调用 `VectorIndexService` 进行分片、向量化、存储

#### MilvusCheckController.java
**作用**: Milvus 健康检查和状态查询

**核心接口**:
- `GET /milvus/health` - 检查 Milvus 连接状态

---

### 2. 服务层 (service/)

#### ChatService.java
**作用**: 封装 ReactAgent 对话的核心逻辑

**核心方法**:
- `createDashScopeApi()` - 创建 DashScope API 实例
- `createStandardChatModel()` - 创建标准对话模型
- `buildSystemPrompt(history)` - 构建包含历史的系统提示词
- `createReactAgent()` - 创建 ReactAgent（整合工具）
- `executeChat()` - 执行非流式对话

**工具集成**:
- DateTimeTools - 获取当前时间
- InternalDocsTools - 检索内部文档
- QueryMetricsTools - 查询 Prometheus 告警
- QueryLogsTools - 查询腾讯云日志（Mock/真实模式切换）

#### AiOpsService.java
**作用**: 实现多 Agent 协作的告警分析流程

**架构模式**: Supervisor-Planner-Executor

**工作流程**:
1. **Supervisor Agent**: 调度 Planner 和 Executor
2. **Planner Agent**: 拆解任务、制定计划、生成最终报告
3. **Executor Agent**: 执行具体步骤、收集证据

**核心方法**:
- `executeAiOpsAnalysis()` - 执行完整的告警分析流程
- `extractFinalReport()` - 从执行结果中提取报告
- `buildPlannerAgent()` - 构建规划器 Agent
- `buildExecutorAgent()` - 构建执行器 Agent

**输出格式**: 结构化 Markdown 报告（告警清单、根因分析、处理方案、结论）

#### VectorIndexService.java
**作用**: 文件读取、分片、向量化、存储到 Milvus

**核心方法**:
- `indexDirectory(path)` - 索引整个目录
- `indexSingleFile(filePath)` - 索引单个文件
- `deleteExistingData(filePath)` - 删除旧数据（避免重复）
- `insertToMilvus()` - 插入向量到 Milvus

**工作流程**:
1. 读取文件内容
2. 删除该文件的旧向量数据
3. 调用 `DocumentChunkService` 分片
4. 调用 `VectorEmbeddingService` 生成向量
5. 构建元数据（文件名、分片索引等）
6. 插入 Milvus

#### VectorSearchService.java
**作用**: 从 Milvus 检索相似文档

**核心方法**:
- `searchSimilarDocuments(query, topK)` - 语义搜索

**应用场景**: RAG 问答时检索相关文档片段

#### VectorEmbeddingService.java
**作用**: 调用阿里云 DashScope Embedding API 生成向量

**核心方法**:
- `generateEmbedding(text)` - 文本→向量（1536 维）

**模型**: text-embedding-v4

#### DocumentChunkService.java
**作用**: 文档分片（按字符数切分，带重叠）

**核心方法**:
- `chunkDocument(content, filePath)` - 将文档切分为多个分片

**配置参数**:
- `max-size`: 800 字符/分片
- `overlap`: 100 字符重叠

#### RagService.java
**作用**: 检索增强生成（RAG）流程编排

**工作流程**:
1. 用户提问
2. 向量搜索相关文档
3. 拼接提示词（问题 + 检索结果）
4. 调用 LLM 生成答案

---

### 3. Agent 工具集 (agent/tool/)

#### DateTimeTools.java
**功能**: 获取当前日期时间

**工具名**: `getCurrentDateTime`

**用途**: 回答时间相关问题

#### InternalDocsTools.java
**功能**: 检索内部文档知识库

**工具名**: `queryInternalDocs`

**用途**: 
- 查询公司内部文档
- 检索最佳实践
- 查找技术指南

**底层**: 调用 `VectorSearchService` 进行语义搜索

#### QueryMetricsTools.java
**功能**: 查询 Prometheus 告警和监控指标

**工具名**: `queryPrometheusAlerts`

**用途**:
- 获取活跃告警列表
- 查询监控指标数据
- 分析告警趋势

**配置**: 支持 Mock 模式（测试用）和真实 Prometheus 连接

#### QueryLogsTools.java
**功能**: 查询腾讯云 CLS 日志

**工具名**: `queryCloudLogs`

**用途**:
- 查询指定服务的日志
- 过滤错误日志
- 分析日志模式

**配置**: 
- Mock 模式：返回模拟数据
- 真实模式：通过 MCP 服务调用腾讯云 API

---

### 4. 配置层 (config/)

#### MilvusConfig.java
**作用**: 创建和管理 MilvusServiceClient Bean

**生命周期**:
- 启动时创建客户端
- 关闭时自动释放资源（`@PreDestroy`）

#### MilvusProperties.java
**作用**: 映射 `application.yml` 中的 Milvus 配置

**配置项**:
- host: localhost
- port: 19530
- username/password: 认证信息
- database: default
- timeout: 10000ms

#### DashScopeConfig.java
**作用**: DashScope API 配置

**配置项**:
- api-key: 从环境变量 `DASHSCOPE_API_KEY` 读取
- retry: 重试策略（最多 3 次，指数退避）

#### DocumentChunkConfig.java
**作用**: 文档分片配置

**配置项**:
- max-size: 800
- overlap: 100

#### FileUploadConfig.java
**作用**: 文件上传配置

**配置项**:
- path: ./uploads
- allowed-extensions: txt,md

---

### 5. 客户端层 (client/)

#### MilvusClientFactory.java
**作用**: 创建 Milvus 客户端连接

**核心方法**:
- `createClient()` - 根据配置创建 MilvusServiceClient

**连接参数**:
- 主机地址
- 端口
- 超时时间
- 数据库名称

---

### 6. 数据层 (dto/)

#### DocumentChunk.java
**字段**:
- content: 分片内容
- chunkIndex: 分片索引
- title: 标题（可选）

#### AIOpsRequest.java
**字段**:
- alertId: 告警 ID
- serviceName: 服务名称
- timeRange: 时间范围

#### FileUploadRes.java
**字段**:
- fileName: 文件名
- filePath: 文件路径
- chunkCount: 分片数量
- success: 是否成功

---

## 环境配置

### 前置要求

1. **JDK 17+**
   ```bash
   java -version
   ```

2. **Maven 3.6+**
   ```bash
   mvn -version
   ```

3. **Docker & Docker Compose**
   ```bash
   docker --version
   docker-compose --version
   ```

4. **阿里云 DashScope API Key**
   - 注册地址: https://dashscope.aliyun.com
   - 获取 API Key 后设置为环境变量

### 环境变量配置

#### Windows (PowerShell)
```powershell
$env:DASHSCOPE_API_KEY="your-api-key-here"
```

#### Linux/Mac
```bash
export DASHSCOPE_API_KEY="your-api-key-here"
```

#### 永久配置（推荐）
在 `~/.bashrc` 或 `~/.zshrc` 中添加：
```bash
export DASHSCOPE_API_KEY="your-api-key-here"
```

---

## 快速开始

### 方式一：一键初始化（推荐）

```bash
make init
```

**执行流程**:
1. 启动 Docker Compose（Milvus + etcd + MinIO + Attu）
2. 启动 Spring Boot 服务（后台运行）
3. 等待服务就绪（最多 60 秒）
4. 上传 `aiops-docs/` 目录下的所有文档到向量数据库

**完成后访问**:
- Web 界面: http://localhost:9900
- Attu (Milvus 管理): http://localhost:8000
- MinIO Console: http://localhost:9001 (admin/minioadmin)

---

### 方式二：手动启动

#### 步骤 1: 启动 Milvus 向量数据库

```bash
docker-compose -f vector-database.yml up -d
```

**验证容器状态**:
```bash
docker ps | grep milvus
```

预期输出:
```
milvus-standalone   Up   19530/tcp, 9091/tcp
milvus-etcd         Up   2379/tcp, 2380/tcp
milvus-minio        Up   9000/tcp, 9001/tcp
milvus-attu         Up   3000/tcp
```

#### 步骤 2: 编译项目

```bash
mvn clean install
```

#### 步骤 3: 启动 Spring Boot 服务

```bash
mvn spring-boot:run
```

或使用 Makefile:
```bash
make start
```

#### 步骤 4: 上传文档（可选）

```bash
make upload
```

这会上传 `aiops-docs/` 目录下的所有 `.md` 文件到向量数据库。

---

### 停止服务

#### 停止 Spring Boot
```bash
make stop
```

#### 停止 Docker 容器
```bash
make down
```

#### 完全清理
```bash
make clean
```

---

## API 接口文档

### 基础信息

- **Base URL**: `http://localhost:9900/api`
- **Content-Type**: `application/json`
- **响应格式**: 统一 JSON 结构

### 1. 智能问答接口

#### 1.1 普通对话（非流式）

**接口**: `POST /api/chat`

**请求体**:
```json
{
  "Id": "session-123",
  "Question": "什么是向量数据库？"
}
```

**响应**:
```json
{
  "code": 200,
  "message": "success",
  "data": {
    "success": true,
    "answer": "向量数据库是一种专门用于存储和检索向量数据的数据库...",
    "errorMessage": null
  }
}
```

**特性**:
- 支持工具自动调用
- 自动维护会话历史（最多 6 对）
- 一次性返回完整答案

---

#### 1.2 流式对话（SSE）

**接口**: `POST /api/chat_stream`

**请求体**:
```json
{
  "Id": "session-123",
  "Question": "如何优化 Java 应用性能？"
}
```

**响应** (SSE 流):
```
event: message
data: {"type":"content","data":"Java"}

event: message
data: {"type":"content","data":" 应用性能优化可以从以下几个方面入手..."}

event: message
data: {"type":"done","data":null}
```

**前端示例**:
```javascript
const eventSource = new EventSource('/api/chat_stream', {
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ Id: 'session-123', Question: '你好' })
});

eventSource.addEventListener('message', (event) => {
  const data = JSON.parse(event.data);
  if (data.type === 'content') {
    console.log(data.data); // 逐块显示
  } else if (data.type === 'done') {
    eventSource.close();
  }
});
```

---

#### 1.3 清空会话历史

**接口**: `POST /api/chat/clear`

**请求体**:
```json
{
  "Id": "session-123"
}
```

**响应**:
```json
{
  "code": 200,
  "message": "success",
  "data": "会话历史已清空"
}
```

---

#### 1.4 获取会话信息

**接口**: `GET /api/chat/session/{sessionId}`

**响应**:
```json
{
  "code": 200,
  "message": "success",
  "data": {
    "sessionId": "session-123",
    "messagePairCount": 5,
    "createTime": 1714320000000
  }
}
```

---

### 2. AIOps 智能运维接口

#### 2.1 执行告警分析

**接口**: `POST /api/ai_ops`

**请求体**: 无需（自动分析当前告警）

**响应** (SSE 流):
```
event: message
data: {"type":"content","data":"正在读取告警并拆解任务...\n"}

event: message
data: {"type":"content","data":"📋 **告警分析报告**\n\n"}

event: message
data: {"type":"content","data":"## 📋 活跃告警清单\n..."}

event: message
data: {"type":"done","data":null}
```

**报告结构**:
```markdown
# 告警分析报告

---

## 📋 活跃告警清单

| 告警名称 | 级别 | 目标服务 | 首次触发时间 | 最新触发时间 | 状态 |
|---------|------|----------|-------------|-------------|------|
| HighCPUUsage | Critical | payment-service | 2026-04-28 10:00 | 2026-04-28 10:30 | 活跃 |

---

## 🔍 告警根因分析1 - HighCPUUsage

### 告警详情
- **告警级别**: Critical
- **受影响服务**: payment-service
- **持续时间**: 30分钟

### 症状描述
CPU 使用率持续超过 90%...

### 日志证据
[ERROR] 2026-04-28 10:15:23 - OutOfMemoryError...

### 根因结论
内存泄漏导致 GC 频繁触发，进而引起 CPU 飙升

---

## 🛠️ 处理方案执行1 - HighCPUUsage

### 已执行的排查步骤
1. 查询近 1 小时错误日志
2. 分析 JVM 堆内存使用情况
3. 检查 GC 日志

### 处理建议
1. 重启服务临时缓解
2. 使用 MAT 分析堆转储文件
3. 修复内存泄漏代码

### 预期效果
CPU 使用率降至 50% 以下

---

## 📊 结论

### 整体评估
当前有 1 个 Critical 级别告警，需要立即处理

### 关键发现
- payment-service 存在内存泄漏
- GC 频率异常高

### 后续建议
1. 立即重启服务
2. 安排代码审查
3. 增加监控告警阈值

### 风险评估
高风险：可能导致服务不可用
```

---

### 3. 文件管理接口

#### 3.1 上传文件

**接口**: `POST /api/upload`

**请求**: multipart/form-data

```bash
curl -X POST http://localhost:9900/api/upload \
  -F "file=@document.txt"
```

**响应**:
```json
{
  "code": 200,
  "message": "success",
  "data": {
    "fileName": "document.txt",
    "filePath": "./uploads/document.txt",
    "chunkCount": 5,
    "success": true
  }
}
```

**支持格式**: .txt, .md

**自动处理**:
1. 保存到 `uploads/` 目录
2. 文档分片
3. 生成向量
4. 存储到 Milvus

---

### 4. 健康检查接口

#### 4.1 Milvus 健康检查

**接口**: `GET /milvus/health`

**响应**:
```json
{
  "status": "healthy",
  "collection": "knowledge_base",
  "entityCount": 1234
}
```

---

## 运维指南

### 1. 查看服务日志

#### Spring Boot 日志
```bash
tail -f server.log
```

#### Docker 容器日志
```bash
# Milvus
docker logs -f milvus-standalone

# etcd
docker logs -f milvus-etcd

# MinIO
docker logs -f milvus-minio
```

---

### 2. 监控容器状态

```bash
make status
```

输出示例:
```
NAMES               STATUS         PORTS
milvus-standalone   Up 2 hours     19530/tcp, 9091/tcp
milvus-etcd         Up 2 hours     2379/tcp
milvus-minio        Up 2 hours     9000/tcp, 9001/tcp
milvus-attu         Up 2 hours     3000/tcp

运行中: 4 / 4
```

---

### 3. 重启服务

#### 重启 Spring Boot
```bash
make restart
```

#### 重启 Docker 容器
```bash
make down
make up
```

---

### 4. 清理数据

#### 清理临时文件
```bash
make clean
```

#### 重置向量数据库（危险操作）
```bash
docker-compose -f vector-database.yml down -v
docker-compose -f vector-database.yml up -d
```

⚠️ **警告**: 这会删除所有向量数据！

---

### 5. 备份与恢复

#### 备份 Milvus 数据
```bash
# 停止服务
docker-compose -f vector-database.yml down

# 备份数据目录
tar -czf milvus-backup.tar.gz ./volumes/

# 重新启动
docker-compose -f vector-database.yml up -d
```

#### 恢复数据
```bash
# 停止服务
docker-compose -f vector-database.yml down

# 恢复数据
tar -xzf milvus-backup.tar.gz

# 重新启动
docker-compose -f vector-database.yml up -d
```

---

### 6. 性能调优

#### Milvus 配置优化

编辑 `vector-database.yml`:
```yaml
standalone:
  environment:
    ETCD_ENDPOINTS: etcd:2379
    MINIO_ADDRESS: minio:9000
    # 增加内存限制
    MEM_LIMIT: 8GB
```

#### Spring Boot JVM 参数

编辑 `Makefile` 中的启动命令:
```bash
nohup mvn spring-boot:run -Dspring-boot.run.jvmArguments="-Xms2g -Xmx4g" > server.log 2>&1 &
```

---

### 7. 故障排查

#### 问题 1: Milvus 连接失败

**症状**: 
```
Failed to connect to Milvus at localhost:19530
```

**解决**:
```bash
# 检查容器状态
docker ps | grep milvus

# 查看日志
docker logs milvus-standalone

# 重启容器
docker restart milvus-standalone
```

---

#### 问题 2: API Key 未配置

**症状**:
```
DashScope API key is required
```

**解决**:
```bash
# 设置环境变量
export DASHSCOPE_API_KEY="your-api-key"

# 验证
echo $DASHSCOPE_API_KEY
```

---

#### 问题 3: 文档上传失败

**症状**:
```
Failed to index document
```

**解决**:
```bash
# 检查文件格式（仅支持 .txt 和 .md）
ls -la aiops-docs/

# 检查 Milvus 连接
curl http://localhost:9900/milvus/health

# 查看详细日志
tail -f server.log | grep "VectorIndexService"
```

---

#### 问题 4: 向量搜索无结果

**症状**: RAG 问答返回"未找到相关文档"

**解决**:
```bash
# 确认文档已上传
curl http://localhost:9900/milvus/health

# 重新上传文档
make upload

# 检查分片配置
grep "document.chunk" src/main/resources/application.yml
```

---

## 常见问题

### Q1: 如何修改向量维度？

**A**: 当前使用阿里云 `text-embedding-v4` 模型，固定输出 1536 维。如需更换模型，修改 `application.yml`:

```yaml
dashscope:
  embedding:
    model: text-embedding-v2  # 改为其他模型
```

⚠️ 注意：更换模型后需要重新向量化所有文档。

---

### Q2: 如何增加会话历史长度？

**A**: 修改 `ChatController.java` 中的 `MAX_WINDOW_SIZE`:

```java
private static final int MAX_WINDOW_SIZE = 10;  // 改为 10 对
```

---

### Q3: 如何添加自定义工具？

**A**: 
1. 在 `agent/tool/` 创建新工具类
2. 使用 `@Tool` 注解标记方法
3. 在 `ChatService.buildMethodToolsArray()` 中添加

示例:
```java
@Component
public class WeatherTools {
    
    @Tool(description = "查询天气")
    public String getWeather(String city) {
        return "晴天";
    }
}
```

---

### Q4: 如何切换 Mock 模式和真实模式？

**A**: 修改 `application.yml`:

```yaml
# Mock 模式（测试用）
cls:
  mock-enabled: true

prometheus:
  mock-enabled: true

# 真实模式（生产用）
cls:
  mock-enabled: false

prometheus:
  mock-enabled: false
```

---

### Q5: 如何查看 Milvus 中的数据？

**A**: 使用 Attu Web 界面

1. 访问 http://localhost:8000
2. 连接到 `milvus-standalone:19530`
3. 浏览集合和向量数据

---

### Q6: 文档分片大小如何调整？

**A**: 修改 `application.yml`:

```yaml
document:
  chunk:
    max-size: 1000  # 增大分片
    overlap: 200    # 增加重叠
```

---

### Q7: 如何实现多用户隔离？

**A**: 当前使用 SessionId 隔离，每个用户的会话独立存储。如需更强隔离：

1. 为每个用户创建独立的 Milvus Collection
2. 在元数据中添加 `userId` 字段
3. 搜索时添加过滤条件

---

### Q8: 如何监控服务性能？

**A**: 
1. 启用 Spring Boot Actuator
2. 集成 Prometheus + Grafana
3. 监控关键指标：
   - JVM 内存使用
   - HTTP 请求延迟
   - Milvus 查询耗时
   - LLM API 调用次数

---

### Q9: 如何部署到生产环境？

**A**: 

1. **打包应用**:
   ```bash
   mvn clean package -DskipTests
   ```

2. **生成 JAR 文件**:
   ```
   target/super-biz-agent-1.0-SNAPSHOT.jar
   ```

3. **部署到服务器**:
   ```bash
   scp target/super-biz-agent-1.0-SNAPSHOT.jar user@server:/opt/app/
   ```

4. **启动服务**:
   ```bash
   nohup java -jar super-biz-agent-1.0-SNAPSHOT.jar > app.log 2>&1 &
   ```

5. **配置反向代理（Nginx）**:
   ```nginx
   server {
       listen 80;
       server_name your-domain.com;
       
       location / {
           proxy_pass http://localhost:9900;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
       }
   }
   ```

---

### Q10: 如何保证数据安全？

**A**: 

1. **API Key 管理**:
   - 不要硬编码在代码中
   - 使用环境变量或密钥管理服务

2. **Milvus 认证**:
   - 启用用户名密码认证
   - 配置 TLS 加密连接

3. **网络隔离**:
   - Milvus 不暴露公网
   - 使用内网访问

4. **定期备份**:
   - 自动备份向量数据
   - 异地灾备

---

## 附录

### A. Makefile 命令速查

| 命令 | 说明 |
|------|------|
| `make help` | 显示帮助信息 |
| `make init` | 一键初始化（启动所有服务 + 上传文档） |
| `make up` | 启动 Docker Compose |
| `make down` | 停止 Docker Compose |
| `make start` | 启动 Spring Boot（后台） |
| `make stop` | 停止 Spring Boot |
| `make restart` | 重启 Spring Boot |
| `make check` | 检查服务状态 |
| `make upload` | 上传所有文档 |
| `make clean` | 清理临时文件 |
| `make status` | 查看容器状态 |
| `make wait` | 等待服务就绪 |

---

### B. 关键配置文件

#### application.yml 核心配置

```yaml
server:
  port: 9900

milvus:
  host: localhost
  port: 19530

spring:
  ai:
    dashscope:
      api-key: ${DASHSCOPE_API_KEY}

document:
  chunk:
    max-size: 800
    overlap: 100

rag:
  top-k: 3
  model: "qwen3-max"
```

---

### C. 依赖版本对照

| 依赖 | 版本 | 说明 |
|------|------|------|
| spring-boot-starter-parent | 3.2.0 | Spring Boot 父 POM |
| spring-ai-bom | 1.1.0 | Spring AI 依赖管理 |
| spring-ai-alibaba-bom | 1.1.0.0-RC2 | 阿里云 AI 扩展 |
| milvus-sdk-java | 2.6.10 | Milvus Java SDK |
| dashscope-sdk-java | 2.17.0 | 阿里云 DashScope SDK |
| lombok | 1.18.30 | 代码简化 |
| gson | 2.10.1 | JSON 处理 |

---

### D. 端口占用情况

| 服务 | 端口 | 用途 |
|------|------|------|
| Spring Boot | 9900 | Web 应用 |
| Milvus | 19530 | gRPC 接口 |
| Milvus | 9091 | HTTP 接口 |
| etcd | 2379 | 元数据存储 |
| MinIO | 9000 | 对象存储 API |
| MinIO Console | 9001 | Web 管理界面 |
| Attu | 8000 | Milvus Web UI |

---

### E. 推荐阅读

- [Spring AI 官方文档](https://spring.io/projects/spring-ai)
- [Milvus 官方文档](https://milvus.io/docs)
- [阿里云 DashScope 文档](https://help.aliyun.com/zh/dashscope)
- [Docker Compose 教程](https://docs.docker.com/compose/)

---

## 许可证

MIT License

---

**文档结束**

如有问题，请查阅项目 Issues 或联系维护团队。
