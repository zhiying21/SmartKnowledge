# 智知库（SmartKnowledge）开发手册 — 详细技术设计

> **文档版本**：v1.0  
> **评审对象**：PRD v2.0、FSD v1.0、数据库设计文档 v1.0  
> **生成日期**：2026-07-05  
> **文档状态**：评审完成，待决策确认  
> **目标读者**：架构师、后端/前端开发、测试、DevOps、项目经理

---

## 一、评审结论与总体建议

### 1.1 总体评价

三份文档整体质量较高，业务框架完整、技术方向合理、数据模型覆盖全面。但从**工程可落地性、18周里程碑可行性、成本可控性**三个维度看，当前版本存在**范围偏大、部分指标过于乐观、跨文档细节不一致**的问题。建议在进入开发前做一次**范围收敛与技术决策冻结（Architecture Decision Record, ADR）**。

### 1.2 关键评审结论

| 维度 | 结论 | 风险等级 |
|------|------|----------|
| **PRD v2.0** | 功能覆盖全面，但P0范围过大；多项量化指标缺乏 baseline；里程碑偏乐观 | 中高 |
| **FSD v1.0** | 流程与API规格详细，但部分安全与降级策略欠具体；Agent 30s超时偏激进 | 中 |
| **数据库设计 v1.0** | Schema 覆盖全，索引与分区策略合理，但存在字段冗余、缺失表、向量隔离方案摇摆 | 中 |
| **跨文档一致性** | 向量隔离级别、Agent依赖关系、缓存失效策略在三份文档中存在不一致 | 中 |
| **整体可落地性** | 按当前范围18周完成全部P0/P1风险高，建议MVP收缩后分阶段推进 | 高 |

### 1.3 核心建议（必须在开发前决策）

1. **P0 范围收缩**：将 Agent（US-201/202/204）、记忆管理、工具调用、Self-Reflection 从 P0 降至 P1；MVP 聚焦「多文档上传 + 混合检索 RAG + 多轮对话 + 用户系统 + 权限」。
2. **冻结向量隔离方案**：统一采用 **Milvus 单 Collection + 按 `kb_id` Partition 隔离**（非每知识库独立 Collection），兼顾安全与运维成本。
3. **调整不可量化指标**："首字<1.5s"、"检索命中率>90%"、"单次问答<¥0.5" 需补充测试 baseline 与模型假设，否则改为"目标值"而非验收红线。
4. **明确本地模型责任**：意图识别、NER、PII 检测若使用本地模型，需列出训练/标注数据计划与人力投入。
5. **统一任务队列**：当前 FSD/DB 同时描述 Celery + 自建 `task_queue` 表，建议以 Celery 为唯一执行引擎，`task_queue` 仅作为运营视角的任务查询视图。

---

## 二、系统总体架构

### 2.1 架构原则

| 原则 | 说明 |
|------|------|
| **无状态服务** | API/WebSocket/Worker 均为无状态，支持 K8s HPA 横向扩展 |
| **存储按特征选型** | 关系数据→PostgreSQL；向量→Milvus；缓存/队列→Redis；文件→对象存储 |
| **防御性设计** | 多 Provider 熔断、模型降级、检索降级、成本熔断四层降级 |
| **安全左移** | 文档解析阶段即脱敏；向量索引中只存脱敏后文本 |
| **可观测优先** | TraceID 全链路、RAG 指标自动采集、Bad Case 自动入库 |

### 2.2 逻辑架构

```
┌─────────────────────────────────────────────────────────────────────┐
│  客户端层                                                            │
│  React 18 + TypeScript + Tailwind + shadcn/ui                      │
│  聊天页 / 文档管理 / 管理后台 / 数据看板                            │
├─────────────────────────────────────────────────────────────────────┤
│  网关层 (Nginx/Traefik)                                             │
│  SSL 终结 / 路由 / 限流 / WAF / 静态资源缓存                        │
├─────────────────────────────────────────────────────────────────────┤
│  应用层 (FastAPI + Uvicorn)                                         │
│  API 路由 │ 业务服务 │ RAG Pipeline │ Agent 引擎 │ 模型网关        │
├─────────────────────────────────────────────────────────────────────┤
│  数据与中间件层                                                      │
│  PostgreSQL │ Milvus │ Redis │ MinIO/OSS │ Celery Worker         │
├─────────────────────────────────────────────────────────────────────┤
│  AI 能力层                                                           │
│  Embedding(BGE-M3) │ LLM(GPT-4o/GLM-4 等) │ Reranker │ OCR       │
├─────────────────────────────────────────────────────────────────────┤
│  观测与评估层                                                        │
│  Prometheus + Grafana + Loki + 自研 RAG 评估 Pipeline               │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.3 技术栈确认

| 组件 | 选型 | 版本/规格 | 决策理由 | 变更需重新评估 |
|------|------|-----------|----------|----------------|
| 前端框架 | React + TypeScript | 18.x | 生态成熟，团队友好 | 否 |
| UI 库 | shadcn/ui + Tailwind | 最新稳定版 | 可定制、支持暗色/无障碍 | 否 |
| 状态管理 | Zustand + React Query | 最新稳定版 | 轻量、TypeScript 友好 | 否 |
| 后端框架 | FastAPI + Uvicorn | 0.111+ | 异步、自动 OpenAPI | 否 |
| 向量数据库 | Milvus | 2.3+ | HNSW、Partition 隔离、生产级 | 是（核心决策） |
| 关系数据库 | PostgreSQL | 15+ | JSONB、FTS、稳定 | 否 |
| 缓存/队列 | Redis Cluster | 7.x | 缓存、计数器、Celery Broker | 否 |
| 任务队列 | Celery + Redis | 5.3+ | 成熟、监控完善 | 否 |
| 对象存储 | MinIO（私有）/ 阿里云 OSS | 最新稳定版 | S3 兼容、低成本 | 否 |
| Embedding | BGE-M3 | 1024 维 | 多语言、效果与成本平衡 | 是 |
| Reranker | BGE-Reranker-v2 / Cohere | Cross-Encoder | 精排效果好 | 是 |
| LLM | GPT-4o / GLM-4 / Kimi | 按 Provider 配置 | 多 Provider 冗余 | 否 |
| 容器编排 | Docker Compose（开发）/ K8s（生产） | - | 开发便捷、生产弹性 | 否 |
| 观测 | Prometheus + Grafana + Loki | - | K8s 原生集成 | 否 |

---

## 三、核心子系统详细设计

### 3.1 文档服务（Document Service）

#### 3.1.1 职责边界
- 文档上传、元数据管理、版本控制
- 文档解析（PDF/DOCX/TXT/MD/XLSX/CSV/PPT）
- 智能分块、PII 脱敏、Embedding、索引构建
- 向量/缓存清理、状态流转

#### 3.1.2 解析引擎策略

| 格式 | 主解析引擎 | 降级引擎 | 备注 |
|------|------------|----------|------|
| PDF | PyMuPDF | pdfplumber | 优先提取文本层与结构 |
| DOCX | python-docx | - | 提取段落、表格、标题层级 |
| MD/TXT | 自定义 Markdown 解析器 | - | 保留标题、代码块、表格 |
| XLSX/CSV | pandas + openpyxl | - | 转为结构化文本 |
| PPT | python-pptx | - | P1 支持 |
| 扫描件 PDF | PaddleOCR / Tesseract | - | 无文本层时触发 |

**关键设计点**：
- 解析前必须进行 **MIME 类型 + 文件魔数** 双重校验，防止扩展名伪造。
- 扫描件 OCR 准确率若 < 85%，文档状态标记为 `INDEXED_WITH_WARNING`，并在 UI 提示"识别结果可能不准确"。
- 病毒扫描使用 ClamAV，异步执行；阳性文件立即隔离并告警。

#### 3.1.3 分块策略

| 知识域 | 分块策略 | chunk_size | overlap | 说明 |
|--------|----------|------------|---------|------|
| 通用文档 | 递归字符分块 | 512 tokens | 50 tokens | 默认策略 |
| 技术文档 | 代码感知分块 | 1024 tokens | 100 tokens | 保留代码块完整性 |
| 制度/行政 | 段落分块 | 384 tokens | 30 tokens | 以标题边界优先 |
| 表格 | 行组分块 | 按行 | - | 表头随数据重复保留 |

**设计约束**：
- chunk token 数不得超过 Embedding 模型上限（BGE-M3 为 8192）。
- 每个 chunk 必须保留结构化元数据：`doc_id`, `chunk_index`, `page_num`, `section_title`, `section_level`, `chunk_type`。
- 分块后文本需进行 **PII 脱敏**，脱敏后文本写入向量数据库与 FTS；原始文本加密存储，仅白名单用户审批后查看。

#### 3.1.4 索引构建流程

```
上传 → 校验 → 存对象存储 → 写 documents(UPLOADED) → Celery 异步解析
  → 解析/结构识别/PII 脱敏 → 分块 → 写 document_chunks(CHUNKED)
  → Embedding(batch=32, 缓存 24h) → 写 Milvus + PostgreSQL FTS(INDEXING)
  → 状态更新为 INDEXED → WebSocket/邮件通知
```

**失败处理**：
- 解析/E embedding/向量写入失败均自动重试 3 次，退避间隔 1s/5s/25s。
- 重试耗尽后状态为 `FAILED`，用户可手动重试。
- 部分 chunk 失败标记为 `PARTIAL_INDEXED`，运营后台可查看失败 chunk 列表。

### 3.2 RAG Pipeline（检索增强生成）

#### 3.2.1 Pipeline 阶段

```
Query 理解 → Query 扩展/分解 → 多路检索 → RRF 融合 → Reranker 精排
  → 权限过滤/去重 → 上下文压缩 → Prompt 组装 → 模型路由 → 流式生成
  → 引用校验 → 安全过滤 → 置信度评分 → 输出
```

#### 3.2.2 Query 理解层

| 子模块 | 实现方式 | 模型/规则 | 输出 |
|--------|----------|-----------|------|
| 意图识别 | 轻量分类模型或 LLM 提示 | 本地 BERT / 云端轻量 LLM | factual_qa / summary / comparison / calculation / multi_step |
| Query 扩展 | 同义词词典 + 缩写还原 + 共现扩展 | PostgreSQL synonyms/abbreviations 表 | expanded_queries[] |
| Query 分解 | LLM 提示 | GPT-3.5 级别 | sub_queries[]（仅多步/对比问题） |
| 敏感信息检测 | 正则 + NER | 本地/云端 | 标记 PII_QUERY，写入审计 |

**设计约束**：
- 意图识别模型必须具备 **可解释性兜底规则**：当模型置信度 < 0.7 时，默认走 `factual_qa`。
- Query 扩展词数不宜过多，建议每个 query 扩展不超过 3 个变体，防止检索噪声。

#### 3.2.3 多路检索层

| 检索方式 | 引擎 | Top-K | 阈值 | 权重（hybrid） |
|----------|------|-------|------|----------------|
| 稠密检索 | Milvus HNSW | 15 | cosine ≥ 0.65 | 0.5 |
| 稀疏检索 | PostgreSQL BM25/FTS | 15 | - | 0.3 |
| 全文检索 | PostgreSQL GIN | 10 | - | 0.2 |

**RRF 融合公式**：`score = Σ 1/(k + rank_i)`，k=60。

**权限过滤**：
- 检索时 Milvus `expr` 必须包含 `kb_id IN [...] AND doc_id IN [...]`。
- `doc_id` 列表由 PostgreSQL 根据用户权限预计算（公开 + 自己上传 + 所在团队）。
- 精排后二次校验 chunk 所属 doc_id 权限，防御向量层绕过。

#### 3.2.4 重排序与上下文压缩

- **Reranker**：候选 ≤ 15 时全量精排；>15 时先 RRF Top-15 再精排。
- **Reranker 熔断**：服务异常时降级为稠密检索 score 排序，并记录降级事件。
- **去重**：chunk 间 Jaccard > 0.8 时合并；同一文档 chunk 按页码排序保留上下文。
- **Token 预算**：总上下文 ≤ 模型最大输入 × 70%（如 GPT-4o 128k 上下文则 ≤ 89k）。
- **关键句提取**（P1）：使用轻量 NLI 模型保留与 query 相关的句子，低相关句子压缩。

#### 3.2.5 生成控制层

**模型路由策略**：

| 任务特征 | 路由模型 | 说明 |
|----------|----------|------|
| 简单事实问答（<100 tokens 输入） | 轻量模型（GLM-4-Flash / GPT-3.5） | 低成本 |
| 复杂分析/多文档对比/Agent | 强模型（GPT-4o / GLM-4） | 高质量 |
| 高峰期/预算紧张 | 轻量模型 | 自动降级 |
| 所有 Provider 不可用 | Ollama/vLLM 本地模型 | 离线兜底 |

**系统 Prompt 模板（必须包含）**：
```text
你是企业知识库助手，基于以下提供的上下文回答问题。
规则：
1. 必须基于提供的上下文回答，禁止编造（strict_mode=true 时强制执行）。
2. 使用 Markdown 格式，代码块标注语言类型。
3. 在关键事实后标注引用[^1][^2]。
4. 若上下文不足，明确说明"未在知识库中找到相关信息"。
5. 对不确定信息标注"置信度：低"。
```

**引用校验**：
- 流式输出实时解析 `[^n]`，映射到 chunk_id。
- 若模型未生成引用，系统在答案末尾强制追加"信息来源"列表。
- strict_mode=true 且无引用时，拒绝生成事实性内容，返回"未找到足够依据"。

**输出安全过滤**：
- 内容安全 API 扫描（涉政/涉黄/暴恐/歧视），置信度 > 0.7 时拦截。
- PII 二次扫描，命中则 `***` 替换。
- Faithfulness 评分 < 0.7 时追加免责声明。

#### 3.2.6 缓存策略

| 缓存类型 | Key | TTL | 一致性策略 |
|----------|-----|-----|------------|
| Query 结果 | `sk:query:{hash(query+kb_ids+search_mode+user_id)}` | 5min | 文档更新时按 kb_id 批量清除 |
| Embedding | `sk:emb:{hash(text+model_version)}` | 24h | 模型版本变更时全量失效 |
| 模型响应 | `sk:llm:{hash(prompt+model+temp)}` | 1h | 仅简单事实问答高置信缓存 |
| 会话上下文 | `sk:conv:{conversation_id}` | 24h | 消息写入时更新 |

**注意**：Query 缓存必须按用户维度隔离，防止跨用户数据泄露。

### 3.3 Agent 引擎（P1/P2，非 P0）

#### 3.3.1 架构调整建议

原 PRD 将 Agent 多能力列为 P0，建议调整为：

| 能力 | 建议优先级 | 说明 |
|------|------------|------|
| 复杂问题拆解（Plan-and-Execute） | P1 | 基于 LLM 提示实现，无需复杂框架 |
| 工具调用（document_search/calculator） | P1 | 内部工具优先，外部 web_search 默认关闭 |
| 记忆管理 | P1 | 短期记忆 + 长期偏好即可 |
| 多 Agent 协作 | P2 | 初期单 Agent 足够 |
| Self-Reflection | P1 | 作为可选增强，而非强制阻塞 |
| 对话分支 | P2 | 需要前端树形展示，成本较高 |

#### 3.3.2 Agent 状态机

```
[接收 Query] → 任务识别
  ├─ 简单问题 → 直接 RAG（降级）
  └─ 复杂问题 → PLANNING → EXECUTING → TOOL_CALL → REFLECTING
      ├─ 通过 → FINAL_ANSWER
      ├─ 不通过（retry<2）→ 修正计划重新执行
      └─ 不通过（retry≥2）→ 降级 RAG + 低置信提示
```

#### 3.3.3 关键约束

- **超时**：默认 60s（原 30s 过短，复杂任务常需多次 LLM 调用）。
- **最大步骤**：默认 10，可在管理后台配置。
- **工具沙箱**：calculator 使用受限 Python `eval`（禁止 `import`、`__import__`、`subprocess`、`os`）；code_interpreter 使用 Docker 隔离（P2）。
- **工具失败**：自动重试 1 次，仍失败则跳过并在最终答案中说明。

### 3.4 对话服务

#### 3.4.1 会话模型

- 会话列表按 `updated_at DESC` 分页，每页 20 条。
- 消息历史使用 cursor-based 分页，避免深分页性能问题。
- 单会话消息数上限 500，超出时自动归档早期消息或生成摘要。
- 30 天未活跃的会话自动归档（pinned 除外）。

#### 3.4.2 上下文管理

- 默认保留最近 10 轮对话（用户/助手交替）。
- 超出上下文窗口时，对早期对话调用 LLM 生成摘要，保留关键决策点。
- 分支对话：复制原会话 messages[:M] 与 Agent 记忆，新会话独立演进。

#### 3.4.3 消息加密

- 消息内容 `content` 使用 AES-256-GCM 加密，密钥由 KMS 按用户派生。
- 管理员可查看元数据（时间、Token、模型），不可查看明文。
- `content_plain_hash` 用于去重检测，但需注意该字段会泄露"内容是否相同"的信息，敏感场景可改为加密哈希。

### 3.5 用户与权限服务

#### 3.5.1 RBAC 模型

- 角色：user / manager / admin（系统内置，不可删除）。
- 权限三元组：`resource:action:scope`，例如 `document:read:team`。
- 用户-角色-团队多对多；权限变更时刷新 Redis 权限缓存并广播失效。

#### 3.5.2 认证流程

- 邮箱注册：密码强度 8 位以上，含字母+数字+特殊字符。
- JWT：Access Token TTL 2h，Refresh Token TTL 30d，Redis 白名单存储。
- SSO：企业微信/钉钉/飞书/LDAP；首次登录自动创建用户，初始密码为空（不可邮箱登录）。
- 账户锁定：连续 5 次密码错误锁定 15 分钟。

#### 3.5.3 配额管理

| 维度 | 默认配额 | 检查点 | 超限行为 |
|------|----------|--------|----------|
| 每日提问次数 | 100 | Redis 计数器 | 429 + 紧急申请入口 |
| 每月 Token 消耗 | 1,000,000 | Redis + 异步落库 | 警告@80%，限制@100% |
| 文档上传数量 | 200/月 | 上传成功时 | 429 |
| Agent 调用次数 | 50/日 | Agent 请求时 | 降级为基础 RAG |

### 3.6 模型网关（Model Gateway）

#### 3.6.1 职责

- 统一封装多 Provider 差异（OpenAI / Azure / 智谱 / 通义 / Kimi / Ollama）。
- 路由策略：task_based / cost_based / load_based / ab_test。
- 熔断、限流、缓存、计费、降级。

#### 3.6.2 熔断器配置

```python
CIRCUIT_BREAKER = {
    "failure_threshold": 5,       # 1 分钟内 5 次失败
    "recovery_timeout": 60,       # 60s 后半开
    "half_open_max_calls": 3,
    "expected_exception": [TimeoutError, ConnectionError, RateLimitError]
}
```

#### 3.6.3 降级链

1. Provider A 故障 → Provider B。
2. 所有云端 Provider 故障 → 本地 Ollama/vLLM。
3. 生成服务不可用 → 返回缓存结果或"系统繁忙"。

### 3.7 任务队列

#### 3.7.1 队列划分

| Queue | 用途 | Worker | 优先级 |
|-------|------|--------|--------|
| `document.parse.high` | 手动上传 | 解析 Worker | 高 |
| `document.parse.normal` | 批量导入 | 解析 Worker | 中 |
| `document.parse.low` | 自动同步 | 解析 Worker | 低 |
| `document.index` | Embedding + 索引 | 索引 Worker | 高 |
| `document.ocr` | 扫描件 OCR | OCR Worker | 中 |
| `notification` | 通知发送 | 通用 Worker | 低 |

#### 3.7.2 任务持久化

- Celery 使用 Redis 作为 Broker 和 Result Backend。
- 运营后台需要查询任务状态时，通过 Celery 结果后端或事件流同步到 `task_queue` 表。
- `task_queue` 表作为**只读运营视图**，不直接参与任务调度。

---

## 四、数据层详细设计

### 4.1 PostgreSQL 核心表调整建议

#### 4.1.1 用户与权限模块

**`users` 表**：保留当前设计，建议补充：
- `auth_type` 字段：`password` / `sso_only` / `mixed`，用于区分密码重置逻辑。
- `email_verified_at` 字段，邮箱注册需验证。

**`user_roles` 表**：建议增加 `expires_at` 索引，支持临时权限过期清理。

#### 4.1.2 文档模块

**`documents` 表**：
- `status` 枚举值过多，建议拆分为 `processing_status`（处理状态）和 `lifecycle_status`（生命周期状态）：
  - `processing_status`: PENDING / PARSING / CHUNKED / INDEXING / INDEXED / PARTIAL / FAILED
  - `lifecycle_status`: ACTIVE / ARCHIVED / DELETING / DELETED
- `custom_permissions` JSONB 建议结构化为：
  ```json
  {"user_ids": [], "team_ids": [], "department_ids": [], "version": 1}
  ```
- 增加 `last_cited_at` 字段，用于冷文档检测。

**`document_chunks` 表**：
- 明确 `content` 为原始文本，`content_masked` 为脱敏后文本；FTS 使用 `content_masked` 的 `to_tsvector`。
- `vector_id` 字段用于关联 Milvus `id`（INT64），必须建立唯一索引。
- 建议增加 `embedding_model` + `embedding_version` 组合索引，支持模型版本升级后的渐进式重索引。

**新增表**：
- `prompt_templates`（US-108）：
  ```sql
  CREATE TABLE prompt_templates (
      id UUID PRIMARY KEY,
      kb_id UUID,
      scope VARCHAR(20), -- personal/team/global
      name VARCHAR(100),
      content TEXT,
      variables JSONB,
      created_by UUID,
      status VARCHAR(20), -- active/pending/rejected
      created_at TIMESTAMP
  );
  ```
- `standard_answers`（修正答案库）：
  ```sql
  CREATE TABLE standard_answers (
      id UUID PRIMARY KEY,
      feedback_id UUID,
      query_pattern VARCHAR(500),
      answer TEXT,
      citations JSONB,
      similarity_threshold FLOAT DEFAULT 0.9,
      usage_count INT DEFAULT 0,
      created_at TIMESTAMP
  );
  ```

#### 4.1.3 对话模块

**`conversations` 表**：
- `total_cost` 保留，建议同时冗余 `total_tokens` 便于快速查询。
- `kb_ids` GIN 索引用于限定检索范围。

**`conversation_messages` 表**：
- `content` 加密存储，建议加密在应用层完成，避免数据库密钥泄露导致明文暴露。
- `citations` JSONB 建议固定结构：
  ```json
  [{"citation_index": 1, "doc_id": "uuid", "chunk_id": "uuid", "relevance": 0.95}]
  ```

#### 4.1.4 Agent 模块

**`execution_plans` / `execution_steps` / `tool_calls`**：保留当前设计。
建议 `execution_steps` 增加 `step_id` 唯一约束 `(plan_id, step_id)`。

#### 4.1.5 审计与日志模块

**`audit_logs` 表**：
- 主键建议改为 `(event_at, id)` 复合主键，配合分区表提高查询效率。
- `trace_id` 单独建立普通索引而非唯一索引（跨服务调用可能重复记录）。

**`token_usage_logs` 表**：
- 增加 `provider` + `model_name` 组合索引，便于成本分摊报表。
- `cost_cny` 计算依赖外部价格表，建议在应用层计算后写入，避免数据库端维护价格。

### 4.2 Milvus 向量库设计（最终方案）

**统一采用：单 Collection + 按 `kb_id` Partition 隔离。**

```yaml
Collection: document_chunks
Fields:
  - id: INT64, PK, auto_id=True
  - doc_id: VARCHAR(36)
  - kb_id: VARCHAR(36)
  - chunk_index: INT32
  - content: VARCHAR(4096)        # 脱敏后文本
  - embedding: FLOAT_VECTOR(1024) # BGE-M3
  - sparse_vector: SPARSE_FLOAT_VECTOR
  - metadata: JSON                # {page_num, section_title, token_count}

Index:
  embedding:
    index_type: HNSW
    metric_type: COSINE
    params: {M: 16, efConstruction: 256}
  sparse_vector:
    index_type: SPARSE_INVERTED_INDEX
    metric_type: IP

Partitions:
  - _default
  - kb_{kb_id}    # 动态创建
```

**检索参数**：
- `ef=128`
- `limit=15`
- `expr="kb_id in [...] and doc_id in [...]"`

**删除策略**：
- 文档删除时，使用 `expr="doc_id == '{doc_id}'"` 批量删除该文档所有 chunk。
- 知识库删除时，直接删除对应 Partition。

### 4.3 Redis 数据结构设计

| 用途 | Key | 数据结构 | TTL |
|------|-----|----------|-----|
| Query 结果缓存 | `sk:query:{user_id}:{hash}` | String | 300s |
| Embedding 缓存 | `sk:emb:{hash(text+model_version)}` | String | 86400s |
| 会话上下文 | `sk:conv:{conversation_id}` | List | 86400s |
| 用户配额 | `sk:quota:{user_id}:{dimension}` | Hash | 按周期 |
| Refresh Token 白名单 | `sk:refresh:{user_id}:{jti}` | String | 2592000s |
| 限流计数 | `sk:rate:{user_id}:{endpoint}` | String | 60s |
| 分布式锁 | `sk:lock:{resource}` | String | 30s |
| 登录失败计数 | `sk:fail:{user_id}:login` | String | 900s |
| 模型熔断状态 | `sk:circuit:{provider}` | String | 60s |
| 用户权限缓存 | `sk:perms:{user_id}` | String | 3600s |

### 4.4 对象存储结构

```
bucket-smartknowledge/
├── raw/{kb_id}/{doc_id}/{filename}           # 原始文件
├── parsed/{doc_id}/text.json                  # 解析产物
├── parsed/{doc_id}/chunks.json                # 分块产物
├── export/{user_id}/{date}/{filename}         # 导出文件
├── backup/{date}/postgres/                    # PG 备份
├── backup/{date}/milvus/                      # Milvus 备份
└── audit/{date}/audit_logs.parquet            # 审计归档
```

---

## 五、安全架构

### 5.1 分层安全策略

| 层级 | 措施 |
|------|------|
| **输入层** | WAF / Nginx 限流；XSS/SQL 过滤；Prompt Injection 检测（关键词+模型） |
| **处理层** | RBAC + ACL；PII 脱敏；模型调用审计；TraceID 全链路 |
| **输出层** | 内容安全审核；引用强制校验；Faithfulness 检测；PII 二次脱敏 |
| **数据层** | TLS 1.3；AES-256 列级加密；KMS 密钥托管；审计日志哈希链 |

### 5.2 Prompt Injection 防护

- **规则拦截**：检测"忽略前文"、"system:"、"DAN"、"角色扮演"等模式。
- **模型检测**：使用轻量分类模型打分，>0.8 拦截。
- **输出审计**：对低置信、无引用的敏感回答进行安全复核。
- **红队测试**：M4、M6 各进行一次。

### 5.3 权限穿透防护

- 所有回答必须基于检索到的上下文生成。
- 回答前进行二次权限校验：确保引用 chunk 的 doc_id 在用户可访问列表中。
- 用户尝试诱导模型泄露无权限文档时，返回"未找到相关信息"并记录审计日志。

### 5.4 数据加密

| 数据 | 加密方式 | 密钥 |
|------|----------|------|
| 用户密码 | bcrypt cost=12 | - |
| 消息内容 | AES-256-GCM | KMS 按用户派生 |
| PII 原始文本 | AES-256-GCM | KMS 按文档派生 |
| API 密钥 | AES-256-GCM | KMS 主密钥 |
| 备份文件 | AES-256 客户端加密 | 独立备份密钥 |

---

## 六、性能与扩展性设计

### 6.1 性能目标（调整后）

| 指标 | 目标值 | 说明 |
|------|--------|------|
| 简单问题 TTFB | ≤2s | 不含网络延迟；复杂问题 ≤5s |
| 完整响应时间 | 简单≤5s；复杂≤15s | Agent 任务允许更长 |
| 流式速率 | ≥10 tokens/s | 中文约 8-12 字/s |
| 向量检索延迟 | ≤100ms @ 10万文档 | P99 |
| 文档解析速度 | ≤3分钟/10MB | 纯文本 PDF |
| 并发在线用户 | ≥100 | 单 K8s 集群 |
| 缓存命中率 | ≥25% | 初期目标，随数据上升 |

### 6.2 扩展路径

| 阶段 | 文档规模 | 用户规模 | 架构动作 |
|------|----------|----------|----------|
| MVP (0-6月) | <10万 | <1万 | 单 PG + 单 Milvus + Redis 主从 |
| 成长期 (6-18月) | <100万 | <10万 | PG 读写分离 + Milvus 集群 + K8s HPA |
| 规模期 (18月+) | <1000万 | <50万 | 租户级分库 + 冷热分离 + 多区域 |

### 6.3 数据库优化

- **连接池**：PgBouncer 事务级连接池，max=1000。
- **分区表**：`audit_logs`、`token_usage_logs` 按月 RANGE 分区，自动化脚本预创建。
- **归档**：90 天前日志转 Parquet 存对象存储，本地保留元数据索引。
- **覆盖索引**：高频列表查询使用覆盖索引。

### 6.4 向量检索优化

- HNSW 参数：`M=16, efConstruction=256, ef=128`。
- 按 `kb_id` Partition 隔离，减少单次扫描数据量。
- Embedding 批量请求，减少网络 RTT。
- 未来可引入 FP16 量化降低内存。

---

## 七、可观测性设计

### 7.1 指标采集

| 类别 | 指标 | 采集方式 |
|------|------|----------|
| 系统 | CPU/内存/磁盘/网络 | Node Exporter |
| API | 延迟 P50/P95/P99、错误率、QPS | Prometheus FastAPI 中间件 |
| 向量 | 检索延迟、召回率 | 自定义埋点 |
| 队列 | 堆积数、处理速率、失败率 | Celery Exporter |
| 模型 | 调用成功率、Token 消耗、费用 | Model Gateway 埋点 |
| RAG | 检索命中率、Faithfulness、幻觉率 | 评估 Pipeline |
| 成本 | 按用户/团队/模型费用 | token_usage_logs 聚合 |

### 7.2 日志与追踪

- **TraceID**：网关生成，贯穿所有服务与数据库调用。
- **日志聚合**：Loki 收集，按服务/TraceID 检索。
- **审计日志**：关键操作独立存储，分区 + 归档 + 哈希链。

### 7.3 告警策略

| 告警级别 | 条件 | 通知渠道 | 升级策略 |
|----------|------|----------|----------|
| P3 | API 错误率 > 5% 持续 5min | 邮件 | - |
| P2 | 队列堆积 > 100 持续 10min | 邮件 + Webhook | 5min 未 Ack 升级 |
| P1 | 向量 DB P99 > 500ms | 电话/SMS | 自动创建 Incident |
| P0 | 所有 Provider 不可用 | 电话 + 自动降级 | SRE 负责人介入 |

---

## 八、部署与运维架构

### 8.1 生产 K8s 部署

```
Ingress Controller (Nginx/Traefik)
    │
    ├─ Web Frontend Pods (React 静态资源)  HPA 2-10
    ├─ API Service Pods (FastAPI)          HPA 3-20
    ├─ WebSocket Pods (SSE 长连接)         HPA 2-10
    ├─ Celery Worker Pods                  HPA 2-10
    ├─ Model Gateway Pods                  2-5
    └─ Cache Pods (Redis Cluster)          3+

StatefulSet:
    ├─ PostgreSQL HA (主从 + Patroni)
    ├─ Milvus Cluster (Coordinator + Node)
    └─ MinIO Cluster

观测层：
    ├─ Prometheus + AlertManager
    ├─ Grafana Dashboards
    └─ Loki

GitOps：
    └─ ArgoCD 蓝绿发布，自动回滚
```

### 8.2 资源估算（100 并发在线用户）

| 组件 | 副本数 | CPU/副本 | 内存/副本 | 存储 |
|------|--------|----------|-----------|------|
| API Service | 3 | 2核 | 4GB | - |
| Celery Worker | 3 | 2核 | 4GB | - |
| PostgreSQL | 1主1从 | 4核 | 16GB | 500GB SSD |
| Milvus | 1 节点 | 8核 | 32GB | 1TB SSD |
| Redis | 3 主3从 | 2核 | 8GB | - |
| MinIO | 4 节点 | 4核 | 8GB | 2TB |

### 8.3 CI/CD 流程

```
代码提交 → 单元测试 → 构建镜像 → 安全扫描（Trivy/Dependabot）
  → 推送镜像仓库 → ArgoCD 同步 → 蓝绿发布
  → 自动化冒烟测试 → 错误率 > 5% 自动回滚
```

---

## 九、开发实施路线图（调整后）

### 9.1 推荐迭代计划

| 里程碑 | 周期 | 目标 | 交付范围 |
|--------|------|------|----------|
| **M0 技术预研** | 第 1-2 周 | 验证核心链路 | POC：PDF 解析 + BGE-M3 + Milvus + GPT-4o 问答 |
| **M1 MVP** | 第 3-6 周 | 端到端跑通 | 单/多文档上传、基础 RAG、流式输出、注册登录、JWT、多会话 |
| **M2 RAG 核心** | 第 7-10 周 | 检索质量提升 | 混合检索、RRF、Reranker、Query 扩展、引用溯源、权限隔离 |
| **M3 Agent 与工具** | 第 11-14 周 | 复杂问题处理 | Plan-and-Execute、document_search、calculator、记忆管理 |
| **M4 企业级** | 第 15-18 周 | 安全/成本/观测 | SSO、配额、审计、模型网关、监控告警、RAG 评估 |
| **M5 运营优化** | 第 19-22 周 | 数据飞轮 | 统计看板、Bad Case、知识缺口、分块可视化、提示词模板 |
| **M6 上线准备** | 第 23-24 周 | 生产就绪 | K8s 部署、压测、渗透测试、文档、培训 |

### 9.2 与原始 18 周计划差异

- 将原始计划扩展为 **24 周**（M0-M6），预留缓冲。
- Agent 从 P0 后移至 M3（P1）。
- 多 Agent 协作、对话分支、GraphRAG 明确为 P2，不在 24 周内承诺。

---

## 十、开发规范

### 10.1 API 设计规范

- Base URL：`/api/v1`
- 认证：`Authorization: Bearer {access_token}`
- TraceID：请求头 `X-Request-ID`
- 分页：优先 cursor-based；列表接口兼容 `page` + `page_size`
- 流式：`Accept: text/event-stream`
- 错误格式：
  ```json
  {
    "error_code": "E-XXXX-XXX",
    "message": "人类可读错误信息",
    "detail": "技术详情（Debug 模式）",
    "trace_id": "uuid"
  }
  ```

### 10.2 代码规范

- Python：Black + isort + flake8 + mypy
- TypeScript：ESLint + Prettier + strict mode
- 提交规范：Conventional Commits
- 单元测试覆盖率：核心业务 ≥ 70%，关键路径 ≥ 80%

### 10.3 数据库规范

- 所有表必须含 `created_at`、`updated_at`。
- 业务表必须软删除 `deleted_at`。
- 枚举值使用 `CHECK` 约束。
- JSONB 字段必须有 GIN 索引。
- 分区表主键必须包含分区键。

### 10.4 安全规范

- 禁止在日志中打印 Token、API Key、密码。
- 所有外部调用必须设置超时与重试。
- 所有用户输入必须校验与转义。
- 敏感操作必须记录审计日志。

---

## 十一、测试策略

### 11.1 测试分层

| 测试类型 | 范围 | 工具 | 目标 |
|----------|------|------|------|
| 单元测试 | 函数/类 | pytest / vitest | 覆盖率 ≥ 70% |
| 集成测试 | 服务间接口 | pytest + TestContainers | 核心链路 |
| E2E 测试 | 用户流程 | Playwright | 关键场景 |
| 性能测试 | API/向量检索 | Locust/k6 | 100 并发达标 |
| 安全测试 | 渗透/红队 | Burp Suite / 自研用例 | 无高危漏洞 |

### 11.2 RAG 效果评估

- 建立评估数据集：100-500 条标注问答对。
- 指标：Context Precision、Context Recall、Faithfulness、Answer Relevancy、Hallucination Rate。
- 每次模型/策略变更需跑评估集对比，下降则阻塞发布。

### 11.3 核心验收用例

1. **基础 RAG**：上传白皮书，提问获取带引用的准确答案，TTFB < 2s。
2. **权限隔离**：无权限用户无法通过问答/文档列表/Prompt 诱导获取文档内容。
3. **模型熔断**：主 Provider 故障后自动切换，用户无感知。
4. **敏感数据**：含薪资文档问答自动脱敏，审计日志记录。
5. **Agent 对比**：复杂对比任务生成表格，引用准确。

---

## 十二、风险与缓解措施

| 风险 | 影响 | 概率 | 缓解措施 |
|------|------|------|----------|
| 18周无法完成全部P0/P1 | 高 | 高 | MVP 收缩；Agent 后移；分24周实施 |
| 文档解析质量差 | 高 | 中 | 多引擎 fallback；人工校对流程；解析质量评分 |
| 向量检索召回率低 | 高 | 中 | 混合检索+重排；Query扩展；持续调优；评估闭环 |
| 大模型 API 成本超支 | 高 | 中 | 配额；缓存；模型路由；Token监控；高峰期降级 |
| Agent 幻觉/错误推理 | 高 | 高 | 强制引用；Self-Reflection；置信度评分；人工反馈 |
| Prompt Injection | 高 | 中 | 输入检测；输出审计；权限穿透测试；红队测试 |
| 敏感信息泄露 | 高 | 低 | PII脱敏；权限隔离；审计日志；水印；下载审批 |
| 第三方 API 中断 | 高 | 低 | 多 Provider；本地模型 fallback；熔断降级 |
| 并发性能瓶颈 | 中 | 中 | 异步处理；HPA；Redis缓存；HNSW优化；连接池 |

---

## 十三、待决策事项

| 序号 | 事项 | 建议决策 | 决策时点 |
|------|------|----------|----------|
| 1 | Agent 是否进入 MVP | **否**，M3 实现 | 立项评审 |
| 2 | 向量隔离级别 | **Partition 级** | 架构评审 |
| 3 | 本地模型部署方式 | Ollama 作为 fallback，vLLM 作为高性能选项 | M3 前 |
| 4 | SSO Provider 优先级 | 企业微信 > 钉钉 > 飞书 > LDAP | M4 前 |
| 5 | 是否引入 GraphRAG | **否**，P2 评估 | M5 后 |
| 6 | 多语言支持范围 | 默认中文，英文 P2 | M2 后 |
| 7 | 对话分支功能 | **P2**，M5 实现 | M3 后 |
| 8 | RLHF 微调 | **P3**，需积累 1 万+ Bad Case | M6 后 |

---

## 十四、附录

### 14.1 文档问题清单（需三份文档协同修订）

1. **PRD**：将 Agent 相关 US 从 P0 调整为 P1；量化指标补充 baseline 与模型假设。
2. **FSD**：统一向量隔离描述为 Partition 级；Agent 超时改为 60s；明确 calculator 沙箱实现。
3. **数据库设计**：补充 `prompt_templates`、`standard_answers` 表；调整 `documents` 状态字段拆分；明确 `task_queue` 仅作为运营视图。

### 14.2 关键接口清单

- `POST /documents/upload`
- `GET /documents`
- `DELETE /documents/{doc_id}`
- `POST /chat/completions`（SSE）
- `POST /feedback`
- `POST /agent/execute`（SSE）
- `POST /auth/login`
- `GET /auth/sso/{provider}/authorize`
- `GET /admin/analytics/dashboard`
- `GET /admin/audit-logs`
- `PUT /admin/model-gateway/config`

### 14.3 参考资源

- FastAPI: https://fastapi.tiangolo.com/
- Milvus: https://milvus.io/
- BGE Embedding: https://github.com/FlagOpen/FlagEmbedding
- RAGAS: https://docs.ragas.io/
- OWASP LLM Top 10: https://owasp.org/www-project-top-10-for-large-language-model-applications/

---

> **文档结束**
> 本手册基于 PRD v2.0、FSD v1.0、数据库设计 v1.0 评审生成，用于指导后续开发实施。关键决策（Agent 优先级、向量隔离、里程碑调整）需在项目启动会上确认。
