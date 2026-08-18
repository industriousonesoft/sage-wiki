# sage-wiki vs sage-memory 全方位对比分析

两个项目同属 xoai 的 sage- 家族、出自同一作者之手，共享「让 AI 拥有长期记忆」的愿景，却走出了两条截然不同的技术路线——一个是「把文档编译成知识图谱」，一个是「Agent 显式写入的记忆库」。本文从产品定位、记忆模型、架构、检索、图谱、LLM 成本、扩展性、生态、工程质量九个维度进行全面对比，并以两者共性收束，最后给出选型建议。

> **数据来源**：sage-wiki (MIT, v0.2.8, 2026-08-05) 和 sage-memory (MIT, v0.13.2, 2026-07-29)，均基于截至 2026-08-18 的代码仓库。

---

## 1. 产品定位与用户画像

| 维度 | sage-wiki | sage-memory |
|------|-----------|-------------|
| **一句话定位** | "graph memory and knowledge base that AI agents and humans build and query together"——文档进、LLM 编译器出互链 wiki + 知识图谱 | "Local MCP memory for AI agents — graph-native, learns from mistakes, no LLM required" |
| **官方 tagline** | — | "Persistent wisdom. Structured thought" / "Memory that learns. Not just remembers." |
| **目标用户** | 个人 vault → 团队 hub → 公司级知识图谱，三段式全谱系 | 个人开发者 / AI coding agent 优先，单机本地 |
| **核心交互** | 丢入文档 → 编译 → 检索问答；Agent 走 MCP（19 工具），人类走 Markdown/TUI/Web UI | Agent 在会话中显式调用 `sage_memory_store` / `sage_memory_search` |
| **心智模型** | 知识工程管线（编译前置） | 外脑 / 工作记忆（写入即检索） |
| **三种记忆映射** | 编译产出：summaries（摘要）/ concepts（概念）/ triples（三元组） | Knowledge（prose）→ sage-memory skill、Structure（graph）→ sage-ontology、Experience（防错规则）→ sage-self-learning |
| **核心理念句** | evidenced graph：typed entities、provenance、per-fact citations | "Zero infrastructure. One SQLite file." / "Intelligence lives in skills, not in the server. The server is fast and dumb." |
| **成熟度** | v0.2.8（2026-08-05），pre-1.0，882 commits，首提交 2026-04-04 | v0.13.2（2026-07-29），98 commits，首提交 2026-03-16 |
| **灵感来源** | Andrej Karpathy 的「LLM 编译个人知识库」设想，基于 Sage Framework 构建 | 同家族产品线，独立起步 |

**关键定位差异**：sage-wiki 是「知识库编译器 + 记忆层基础设施」，服务对象是**文档与组织**——把原始资料变成可引用、可追溯的知识；sage-memory 是「Agent 的会话记忆 server」，服务对象是**Agent 本身**——把对话中产生的经验变成可召回的规则。前者回答「这段知识从哪里来」，后者回答「上次这个坑是怎么踩的」。

两者的关系在 sage-memory 的 README 中有一段自我定义（题眼）：

> **Conversation-turn auto-extraction** — [sage-wiki](https://github.com/xoai/sage-wiki) extracts facts & concepts from raw data via an LLM. sage requires the agent to call `sage_memory_store` explicitly.

即：sage-wiki 用 LLM **自动**从原始数据中提取事实与概念；sage-memory 要求 Agent **主动**写入记忆。这一句点出了全文最重要的分野，下一节展开。

---

## 2. 记忆模型与数据流：编译 vs 显式写入（核心差异）

### 2.1 sage-wiki：文档 → 编译 → wiki + 图谱

```
原始文档 (raw/：论文、笔记、代码、邮件、图片)
   │
   ▼
LLM 编译器（分层 Tier 0–3 + Pass 1–3）
   ├─ Pass 1 摘要      ──→ wiki/summaries/
   ├─ Pass 2 概念提取  ──→ wiki/concepts/（图谱节点）
   └─ Pass 3 三元组抽取（opt-in）──→ 类型化边 + evidence
   │
   ▼
产出：Obsidian 兼容 Markdown wiki + .sage/wiki.db（FTS5 + 向量 + 本体）
   │
   ├─ Agent：19 个 MCP 工具（wiki_search / wiki_graph_query / wiki_capture …）
   └─ 人类：TUI 仪表盘 / Web UI / 纯 Markdown 浏览
```

### 2.2 sage-memory：Agent 显式写入 → 检索召回

```
Agent 会话中（写代码、对话、犯错被纠正）
   │  显式调用 sage_memory_store()（可附 entities/relations）
   ▼
SHA-256 去重 → memories / chunks / entities / edges 表
   │
   ▼
.sage-memory/memory.db（项目级）+ ~/.sage-memory/memory.db（全局级，双库）
   │
   ▼
六阶段检索管线（BM25 + vec0 + 图谱三通道加权 RRF）
   │
   ▼
召回结果写回 Agent 上下文 → 下次会话「还记得」
```

### 2.3 核心差异表

| 维度 | sage-wiki | sage-memory |
|------|-----------|-------------|
| **数据入口** | 文档文件（人/Agent 放进 `raw/`） | Agent 显式 API 调用（`sage_memory_store`） |
| **提取主体** | LLM 编译器（Tier 3 时）或确定性解析器（Tier 0–2） | Agent 自己（LLM 只做可选的 `extract_entities`） |
| **写入时机** | 编译批处理（`sage-wiki compile` / MCP `wiki_compile`） | 即时、单条、随会话发生 |
| **LLM 角色** | 编译器主体（摘要/概念/三元组） | 可选增强（实体提取/重排），全程可缺省 |
| **产出形态** | 人类可读 wiki 文章 + 图谱 + evidence 引用 | 私有记忆行（prose/chunks/edges），无人类视图 |
| **知识生命周期** | 源文档驱动：改源 → 增量重编译（`wiki_compile_diff`） | 记忆驱动：update/delete/promote（防错规则 context→personal→team） |
| **溯源能力** | ✅ per-fact citation（span/confidence/source_doc） | ❌ 无 evidence 字段（信任写入方） |

**类比**：sage-wiki 像出版社/档案馆——文献进、百科出，每条知识都有出处；sage-memory 像私人笔记本/外脑——主动记、随时查，内容由使用者负责。两者都在做「把隐性知识结构化」：一个靠编译管线自动完成，一个靠三个 skill（sage-memory / sage-ontology / sage-self-learning）教 Agent 自己完成。

---

## 3. 架构与技术栈

### 3.1 sage-wiki 架构（Go 单二进制）

```
┌─────────────────────────────────────────────────┐
│  CLI (cmd/sage-wiki)                            │
│  ├─ init / compile / search / query / tui       │
│  ├─ serve (MCP stdio/SSE + HTTP /v1 API)        │
│  └─ ontology / hub / pack / skill / cost ...    │  ← 25 个命令
├─────────────────────────────────────────────────┤
│  Core Engine (Go 1.26, 零 CGO, internal/ 46 包) │
│  ├─ compiler: 分层编译 Tier 0-3 + Pass 1-3      │  ← 最大包（104 文件）
│  ├─ search: BM25 + Vector + Graph 加权 RRF      │
│  ├─ ontology: 类型化实体-关系 + bi-temporal     │
│  ├─ storage: SQLite FTS5 + BLOB 向量            │
│  │   或 PostgreSQL + pgvector（企业部署）        │
│  ├─ trust: 输出隔离 → grounding → consensus     │
│  └─ mcp: 19 工具（internal/mcp/server.go）      │
├─────────────────────────────────────────────────┤
│  SDK & API                                      │
│  ├─ Python SDK (pip install sagewiki)           │
│  ├─ TypeScript SDK (npm install sagewiki)       │
│  ├─ Go 嵌入式库 (pkg/sagewiki, 进程内 MCP)      │
│  └─ OpenAPI 3.1 规范 (api/openapi.yaml)         │
└─────────────────────────────────────────────────┘
```

### 3.2 sage-memory 架构（Python MCP server）

```
┌─────────────────────────────────────────────────┐
│  CLI (console script: sage-memory)              │
│  ├─ run / serve / hub / auth / status           │
│  ├─ reindex / dedup / queue / worker            │
│  ├─ scan-codebase / code / embedder             │
│  └─ install-skills                              │  ← 13 个子命令
├─────────────────────────────────────────────────┤
│  MCP Server (FastMCP ≥3.0, server_fastmcp.py)   │
│  ├─ 12 个 sage_memory_* 工具（server.py）       │
│  ├─ SecurityMiddleware（bearer/Host/loopback）  │
│  └─ 传输：stdio / SSE / streamable HTTP         │
├─────────────────────────────────────────────────┤
│  Core (src/sage_memory/, 60+ 模块, 纯 Python)   │
│  ├─ search/rerank/expand: 六阶段检索管线        │
│  ├─ embedder: T0-T3 嵌入级联                    │
│  ├─ graph: 类型化有向边 + cycle-safe BFS        │
│  ├─ codebase: tree-sitter 扫描（最大子包 6.1K 行）│
│  └─ hub: 跨项目联邦（fan_out_search）           │
├─────────────────────────────────────────────────┤
│  Storage                                        │
│  ├─ SQLite + FTS5 + sqlite-vec（13 个迁移）     │
│  └─ 项目级 .sage-memory/ + 全局级 ~/.sage-memory/│
└─────────────────────────────────────────────────┘
```

### 3.3 技术栈对比

| 维度 | sage-wiki | sage-memory |
|------|-----------|-------------|
| **语言/运行时** | Go 1.26，零 CGO | Python ≥3.11，纯 Python |
| **分发方式** | 单二进制（~33MB）+ Docker（Alpine） | PyPI + `uvx sage-memory` |
| **存储** | modernc.org/sqlite（FTS5 + BLOB 向量）或 PostgreSQL/pgvector | SQLite + FTS5 + sqlite-vec（vec0），13 个 SQL 迁移 |
| **MCP 框架** | mark3labs/mcp-go | FastMCP ≥3.0 + mcp SDK |
| **UI** | Charmbracelet TUI；Preact + Tailwind Web UI（go:embed，~1.2MB） | 无（纯 MCP server） |
| **HTTP 服务** | 内置 serve（OpenAPI 3.1 契约） | uvicorn（SSE / streamable HTTP） |
| **运行时依赖** | 零系统依赖 | 4 个运行时依赖；extras: `neural`（fastembed bge-small）/ `codebase`（tree-sitter-language-pack） |

**关键架构差异**：
- **进程模型**：sage-wiki 是常驻 server 或一次性 CLI 命令；sage-memory 是随 Agent 会话启动/退出的 stdio 子进程（也可 serve 常驻）。
- **状态可见性**：sage-wiki 编译产物写入 wiki.db，运行中的 MCP server 需重启可见新向量；sage-memory 单文件直写、即写即查。
- **状态管理**：sage-wiki 用 `.manifest.json` 跟踪源↔文章映射与编译状态；sage-memory 用 SHA-256 去重 + 后台 worker 队列管理异步增强任务。

---

## 4. 检索与排序机制

### 4.1 sage-wiki 管线（`internal/search/pipeline.go`）

```
Query
  ├─ 强信号探测（BM25 top-1 ≥0.4 且 2× 差距 → 跳过扩展）
  ├─ LLM 查询扩展：lex×2 + vec×1 + HyDE×1（4 变体）
  ├─ 三通道并行
  │    ├─ BM25（chunk ~800 tokens + document 双索引）
  │    ├─ Vector（余弦相似度）
  │    └─ Graph（≤2 hops，4-Signal 相关性）
  ├─ 加权 RRF：bm25 0.7 / vector 0.3 / graph 0.2（k=60）
  ├─ LLM 重排 top-15（400 tokens/候选，8000 预算）
  │    ├─ 位置感知混合：1-3 名 75/25，4-10 名 60/40，11+ 名 40/60
  │    └─ 50% 覆盖门控（不足则弃用重排）
  └─ Tag / Trust / 时效 / 截断
```

### 4.2 sage-memory 六阶段管线（`src/sage_memory/search.py`）

```
Query
  ├─ LLM 扩展（可选，无 key 跳过）
  ├─ 三通道并行
  │    ├─ BM25（FTS5，OR 语义，常开）
  │    ├─ Vector（需 embedder quality ≥0.6，否则关闭）
  │    └─ Graph（需 entities 表非空；权重 0.7，ADR-004）
  ├─ 加权 RRF（graph 通道权重最高）
  ├─ chunk-rollup 去重合并
  ├─ LLM 重排（可选，position-blend 曲线 0.75/0.6/0.4）
  └─ 最终分：relevance + tag boost + recency + db_boost
```

### 4.3 机制对比

| 维度 | sage-wiki | sage-memory |
|------|-----------|-------------|
| **索引粒度** | chunk（~800 tokens）+ document 双索引 | memories + chunks 双表（FTS5 触发器同步） |
| **查询扩展** | LLM 4 变体 + 强信号短路（仅 Q&A 默认启用） | LLM lex/vec/hyde 变体（可选） |
| **图谱通道权重** | 0.2（三通道中最低） | 0.7（三通道中最高，图谱是一等通道） |
| **重排** | LLM top-15，位置感知混合 + 50% 覆盖门控 | LLM position-blend（0.75/0.6/0.4，可选） |
| **无 API key 行为** | Tier 0 FTS5 始终可用（~5ms/文档） | 全部 LLM 阶段静默降级，免费路径 97.2% recall@5 |
| **延迟** | FTS5 top-10 0.035ms；混合 RRF 4.9ms；LLM 阶段 1–5s（既有文档数据） | sub-3ms 全链路（无 LLM 时） |

**设计哲学要点**：sage-wiki 是「工业级 LLM 增强管线」——每次额外 LLM 调用都在换质量，用覆盖门控和强信号短路控制成本；sage-memory 是「fast and dumb」——server 只管 FTS5/RRF，智能放在 skills 与可选 LLM 阶段，默认零成本、零 key 可用。

---

## 5. 图谱实现

| 维度 | sage-wiki | sage-memory |
|------|-----------|-------------|
| **构建方式** | 编译产出（Pass 2 概念 + Pass 3 三元组，或基础模式关键词共现） | Agent 显式写入 edges（`sage_memory_link`） |
| **边语义** | 类型化关系 + evidence（span/confidence/source_doc） | typed directed edges（`src, dst, relation` 三元组） |
| **时态** | ✅ bi-temporal，`as_of` 查询，矛盾使旧边 invalidate | ❌ 无时态 |
| **实体解析** | ✅ 0.85 阈值自动应用，`ontology resolve --unlink` 可逆 | ❌ 无（依赖写入方规范实体名） |
| **遍历** | 4-Signal 相关性 + 2-hop BFS，按关系类型加权（contradicts 1.1 / cites 0.7） | cycle-safe multi-hop BFS，entity-mediated proximity |
| **检索权重** | 0.2 | 0.7 |
| **社区/洞察** | ✅ Louvain 社区发现 | ❌ 无（有 suggested_links 启发式链接建议） |

同一条 `cites` 边的两种表达：

```
sage-wiki:    (paper_A) ──cites──▶ (paper_B)
              evidence: span=§2.3, confidence=0.9, source_doc=notes.md
              （可追溯至具体句子，矛盾会时态性 invalidate 旧边）

sage-memory:  (chunker.py) ──cites──▶ (db.py)
              （快速、轻量、无出处）
```

**设计哲学**：在 sage-wiki 中，图谱是编译的**产物**——质量由编译器和 evidence 机制保证，适合审计与多跳问答；在 sage-memory 中，图谱是显式维护的**关系**——质量由 Agent 的写入习惯保证，轻量边足以支撑依赖类查询（"what depends on X"）。这也是两者图谱通道检索权重 0.2 vs 0.7 的根源：sage-wiki 的图谱稀疏而金贵，只作补充信号；sage-memory 的图谱是用户亲手构建的一等信号。

---

## 6. LLM 依赖与成本模型

| 维度 | sage-wiki | sage-memory |
|------|-----------|-------------|
| **LLM 必选？** | ❌ Tier 0–2 完全不用 LLM | ❌ 所有 LLM 阶段 opt-in |
| **LLM 用途** | 编译器（摘要/概念/三元组）+ 查询扩展 + 重排 | 仅 `extract_entities()` + 可选扩展/重排 |
| **提供商** | Anthropic/OpenAI/Gemini 原生结构化输出 + Ollama 本地 | Anthropic 主（claude-haiku-4-5）+ OpenAI 备（gpt-4o-mini），raw httpx，3 次重试 |
| **降级/优化** | Batch API、prompt caching（50–90% 输入节省）、`compile --estimate`、price registry | 无 key → 全部静默降级为免费路径 |
| **订阅复用** | OAuth 订阅认证（被 sage-memory 移植的源头） | ✅ OAuth 导入 `.claude/auth.json`、`.codex/auth.json`（ADR-010，移植自 sage-wiki `internal/auth/providers.go`） |

**sage-wiki 分层编译**（`internal/compiler/`，成本前置在「写」）：

| Tier | 内容 | 成本 | 速度 |
|------|------|------|------|
| Tier 0 | FTS5 索引 | 免费 | ~5 ms/文档 |
| Tier 1 | 向量嵌入 | ~$0.00002/文档 | ~200 ms |
| Tier 2 | 正则代码解析（10 语言，无 LLM） | 免费 | 快 |
| Tier 3 | 全量 LLM 编译（摘要/概念/三元组） | ~$0.05–0.15/文档 | 5–8 分钟 |

**sage-memory 嵌入级联**（`src/sage_memory/embedder.py`，成本按需发生在「读」）：

| Tier | 实现 | quality | 说明 |
|------|------|---------|------|
| T0 | 显式指定 embedder | 按所选 | 最高优先 |
| T1 | hosted API（OpenAI/Voyage/Cohere） | ~0.85+ | 需 key |
| T2 | FastEmbed 本地（bge-small-en-v1.5，ONNX） | 中 | extras `neural` |
| T3 | 零依赖 LocalEmbedder（本地 TF-IDF 式） | 0.45 | **低于 0.6 门控，不启用向量通道** |

**结论段**：成本花在**写**还是**读**。sage-wiki 把成本前置在编译（100K 文档 Tier 1 约 5.5 小时，既有文档数据），换来高质量、可引用的知识资产；sage-memory 写入永远零成本，成本只在你主动开启检索增强时发生——免费路径已有 97.2% recall@5，带 hosted embedder 达 98.6%（约 $0.50/500 查询，既有文档数据）。

---

## 7. 扩展性与规模

| 维度 | sage-wiki | sage-memory |
|------|-----------|-------------|
| **目标规模** | 100K+ 文档（无文件数硬限制） | 单个开发者的长期记忆 + 多项目 |
| **规模策略** | 分层编译 + HNSW ANN + int8 量化 + mmap 向量后端 | per-project 双库 + Hub 跨项目 FTS5 fan-out（ADR-008） |
| **存储升级路径** | SQLite → PostgreSQL + pgvector（公司级） | 无（单 SQLite 是卖点） |
| **性能指标** | FTS5 top-10 0.035ms；RRF 4.9ms；100K Tier 1 索引 ~5.5h（既有文档数据） | sub-3ms 搜索；~1000 写/秒；sub-0.3ms 图遍历；340K 行代码库扫描后 91% recall |
| **已知瓶颈** | 首编译耗时；内存向量缓存超 RAM 退化 | 单机无同步（Git/syncthing 自理）；无 trust 体系 |

**定性**：sage-wiki 的扩展轴是**知识规模**（文档数），靠分层编译控制成本曲线；sage-memory 的扩展轴是**会话跨度 × 项目数**，靠双库布局 + hub 联邦控制边界。前者为「组织知识」设计，后者为「个人经验」设计。

---

## 8. 生态与工具面

| 维度 | sage-wiki | sage-memory |
|------|-----------|-------------|
| **CLI** | cobra 25 个命令 | console script + 13 个子命令（run/serve/hub/auth/status/reindex/dedup/scan-codebase/code/embedder/install-skills…） |
| **TUI** | ✅ Charmbracelet 四标签仪表盘 | ❌ |
| **Web UI** | ✅ Preact+Tailwind（go:embed） | ❌ |
| **SDK** | ✅ PyPI/npm `sagewiki` + Go `pkg/sagewiki` 进程内嵌入 | ❌（MCP 即接口） |
| **MCP 工具数** | 19 | 12 |
| **技能文件** | ✅ 生成 agent skill 文件（claude-code/cursor/windsurf/agents-md/codex/gemini） | ✅ 3 个捆绑 skills（版本 1.2.0）+ `install-skills`（5 种 agent 适配器） |
| **扩展机制** | 外部解析器（沙箱子进程）+ 8 个 Contribution Packs | extras：`neural` / `codebase` |
| **代码库理解** | Tier 2 正则解析（10 语言） | tree-sitter（10 语言）+ 增量 hash 重扫 + path/affected/hubs 查询 |
| **传输协议** | MCP stdio/SSE + REST `/v1`（OpenAPI 3.1） | stdio / SSE / streamable HTTP |

**MCP 工具清单**（均已核对 `internal/mcp/server.go` 与 `server.py`）：

| sage-wiki（19） | sage-memory（12） |
|-----------------|-------------------|
| `wiki_search` `wiki_read` `wiki_status` `wiki_ontology_query` `wiki_graph_query` `wiki_list` `wiki_add_source` `wiki_write_summary` `wiki_write_article` `wiki_add_ontology` `wiki_learn` `wiki_commit` `wiki_compile_diff` `wiki_compile` `wiki_lint` `wiki_query` `wiki_capture` `wiki_compile_topic` `wiki_provenance` | `sage_memory_set_project` `sage_memory_store` `sage_memory_search` `sage_memory_update` `sage_memory_delete` `sage_memory_list` `sage_memory_link` `sage_memory_graph` `sage_memory_scan_codebase` `sage_memory_code_path` `sage_memory_code_affected` `sage_memory_code_hubs` |

（注：sage-memory 的 hub 不暴露独立 MCP 工具，`sage_memory_search` 在 hub 激活时内部走 `fan_out_search` 分支聚合跨项目结果，`server_fastmcp.py:265`。）

**安全机制对比**：
- sage-wiki：output quarantine（输出隔离）→ grounding → consensus 三级信任体系、per-relation provenance、HMAC-SHA256 webhooks、S3 mirror 备份（AES-256-GCM 加密）、Prometheus `/metrics`、SPEC-08 资源限额（max_doc_bytes 10 MiB、compile_doc_timeout 15m 等）。
- sage-memory：SecurityMiddleware（bearer token 用 `hmac.compare_digest` 比对、Host allowlist、Origin 校验、非 loopback 无 token 拒绝启动、`set_project` path scoping + denied dirs `~/.ssh`/`~/.gnupg`/`~/.aws`/`/etc`）。

---

## 9. 测试与工程质量

| 维度 | sage-wiki | sage-memory |
|------|-----------|-------------|
| **提交/起步** | 882 commits，首提交 2026-04-04 | 98 commits，首提交 2026-03-16 |
| **代码规模** | ~300K 总行；162,695 行 Go / 752 文件；compiler 最大包（104 文件） | ~54,700 总行；~18,350 行 src Python / 271 文件 |
| **测试规模** | 426 个 `*_test.go` | ~120 测试文件 / 27.5K 测试行 |
| **质量机制** | SPEC-09 parity goldens（LLM fixture 录制回放）、fuzz、determinism 检查、SPEC-08 限额 | 13 个 SQL 迁移、ADR 决策记录（ADR-004/008/010） |
| **记忆评测** | LOCOMO 92.0%、LongMemEval-S 93.3%、BEAM 100K 0.691（既有文档数据） | LongMemEval-S 97.2% recall@5（$0）/ 98.6%（带 embedder）、4 个真实代码库 91% recall |

**口径说明（重要）**：两者都跑 LongMemEval-S，但写入方式与评测配置不同——sage-wiki 是文档编译入库存量评测，sage-memory 是显式写入召回评测，分数**不可直接横向对比**，本节数字仅作工程投入与质量意识的佐证。

工程文化同源：sage-wiki 用 SPEC-xx 规范驱动开发，sage-memory 用 ADR 架构决策记录，两者共享「先立规范、后写实现」的流程。

---

## 10. 共性：同一作者的两条产品线

| 维度 | 说明 |
|------|------|
| **作者/组织** | 同一作者（xoai），同为 MIT 协议，sage- 命名家族 |
| **双库布局** | 项目级 + 全局级 SQLite（`.sage/wiki.db` vs `.sage-memory/` + `~/.sage-memory/`） |
| **Hub 联邦** | 两项目均有 hub 概念，跨项目聚合检索（sage-wiki hub 命令族 vs sage-memory `fan_out_search`） |
| **订阅认证** | OAuth 凭证复用；sage-memory ADR-010 明确「移植自 sage-wiki」 |
| **核心理念** | 都信「知识图谱是一等检索通道」；都强调本地优先与零基础设施起步 |
| **技术迁移** | sage-memory 0.13.0 从 sage-wiki 移植了 Pattern B（共享 MCP server）与 Pattern C（hub 联邦） |

代码库中的互引证据（正文逐一给出出处）：
1. sage-memory README「Where you might prefer another tool」段直接推荐 sage-wiki 做会话轮次自动提取，并以 "Both tools are valid; pick based on your infrastructure." 收束。
2. `src/sage_memory/hub/config.py` 注释 "Defaults on `add_project`: `searchable=True` (matches sage-wiki `hub.go:122-126`)"。
3. `src/sage_memory/auth/oauth.py` 注释 "Per ADR-010: ports sage-wiki's `internal/auth/oauth.go` pattern."
4. sage-memory `docs/guides/team-setup.md` 明确 "sage-memory has **no built-in trust system** like sage-wiki's"，并说明团队场景叙事向 sage-wiki 收敛。
5. sage-memory `docs/guides/retrieval-engine.md` §10 专门设「Embedder Comparison with sage-wiki」一节。
6. sage-memory 的 `tests/test_serve_security.py` 引用 "sage-wiki [LRN:gotcha] — same hardening task"，安全加固经验双向共享。

**时间线**：sage-memory 先起步三周（2026-03-16 vs 2026-04-04），但 sage-wiki 迭代密度约 9 倍（882 vs 98 commits）——一个走「轻、快、纯 Python」路线，一个走「重、全、工业级 Go」路线，同一作者用两个语言栈分别验证「记忆产品」的两端。

---

## 11. 结论与选型建议

**设计哲学差异**：

- **sage-wiki = 知识工程管线**。信任编译器与 evidence，把文档变成可审计、可引用的组织记忆；成本前置（编译）、规模后置（100K+）；输出带隔离与审核，适合「知识需要出处」的场景。
- **sage-memory = Agent 的工作记忆**。信任 Agent 的显式写入，零基础设施、LLM 全可选；智能在 skills 而非 server（"fast and dumb"），适合「经验需要沉淀」的场景。

| 场景 | 推荐 | 理由 |
|------|------|------|
| 从大量文档构建可引用知识库 | sage-wiki | 编译管线 + per-fact citation，知识有出处 |
| Agent 会话间记住偏好、教训、防错规则 | sage-memory | self-learning 循环把每次纠正变成 prevention rule |
| 多跳关系问答 + 证据溯源 | sage-wiki | evidenced graph + `wiki_graph_query` + provenance |
| 完全本地、零 API key | sage-memory | 免费路径 97.2% recall@5，零基础设施 |
| 团队共享 + 审核信任机制 | sage-wiki | Output Trust 三级体系 + Git/服务器/Hub 部署 |
| 代码库结构记忆（跨文件依赖问答） | sage-memory（或 sage-wiki Tier 2） | tree-sitter 扫描 + code_affected/code_hubs 即装即用 |
| 100K+ 文档规模 | sage-wiki | 分层编译 + pgvector 企业级路径 |
| 五分钟上手、单 SQLite 文件 | sage-memory | `uvx sage-memory` 一条命令 |

**互补使用**：两者不是二选一。同一 Agent 可同时挂两个 MCP server——sage-wiki 作「文档知识层」（长半衰期、组织知识），sage-memory 作「会话经验层」（短反馈、个人教训）；经验层中沉淀的洞察可以反向指导 sage-wiki 的编译选题（哪些源值得 Tier 3 全量编译）。两者共享 hub 概念与 auth 机制、出自同一作者，边界清晰、天然互补。

> Both tools are valid; pick based on your infrastructure.（两者都有其价值；请根据你的基础设施选择。）—— sage-memory README
