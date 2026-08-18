# sage-wiki vs llm_wiki 全方位对比分析

两个项目都源于 Andrej Karpathy 提出的「LLM 编译个人知识库」理念，但走向了截然不同的产品路径。本文从产品定位、架构技术栈、特色功能、Agent 能力四个维度进行全面对比。

> **数据来源**：sage-wiki (MIT, v0.2.6, 2026-07-31) 和 llm_wiki (GPL v3.0, v0.6.8)，均基于截至 2026-08-12 的代码仓库。

---

## 1. 产品定位差异

| 维度 | sage-wiki | llm_wiki |
|------|-----------|----------|
| **产品形态** | Go 二进制 CLI + MCP 服务器 + 可嵌入 Web UI | 跨平台桌面应用 (Tauri v2) |
| **目标用户** | 从个人到企业团队的全谱系：个人用本地模型零成本运行，团队用 Git / 自托管服务器共享，企业用 PostgreSQL + pgvector 规模化部署 | 个人用户为主，提供五种场景模板（研究、阅读、个人成长、商业、通用） |
| **核心理念** | 「Graph Memory」—— 知识图谱是检索通道而非副视图，强调 evidence（证据）、provenance（溯源）、trust（信任）、bi-temporal（双向时态） | 「Personal knowledge base that builds itself」—— 强调自动化、桌面端体验、开箱即用 |
| **交互方式** | CLI 优先（20+ 命令），TUI 终端仪表盘，Web UI 可选嵌入，MCP 协议供 Agent 调用 | 桌面 GUI 优先（三栏布局：知识树 + 聊天 + 预览），内置 Chat Agent，图标侧边栏八模块切换 |
| **数据主权** | 自托管，SQLite 单文件（零配置）或 PostgreSQL，无外部依赖 | 本地桌面应用，数据存于本地文件系统的 Markdown 文件中 |
| **开源协议** | MIT | GPL v3.0 |
| **成熟度** | 较为完备：Python + TypeScript SDK，OpenAPI 3.1 规范，CI 驱动测试，memory benchmark 评测，7 种语言 README | 快速迭代中：v0.6.8，丰富的 UI 交互功能，4 种语言界面 (en/zh/it/ru) |

**关键定位差异**：sage-wiki 定位为 **Agent 的记忆层基础设施**（"graph memory and knowledge base that AI agents and humans build and query together"），而 llm_wiki 定位为 **个人桌面知识管理工具**（"A personal knowledge base that builds itself"）。前者是后端引擎 + API 服务，后者是完整的桌面应用体验。

---

## 2. 产品架构与技术栈差异

### 2.1 sage-wiki 架构

```
┌─────────────────────────────────────────────────┐
│  CLI (cmd/sage-wiki)                            │
│  ├─ init / compile / search / query / tui       │
│  ├─ serve (MCP stdio/SSE + HTTP /v1 API)        │
│  └─ ontology / hub / pack / skill / scribe ...  │
├─────────────────────────────────────────────────┤
│  Core Engine (Go 1.26, 零 CGO, ~30 个内部包)     │
│  ├─ Compiler: 四层分级编译 (Tier 0-3) + Pass 1-3 │
│  ├─ Search: BM25 + Vector + Graph 三通道 RRF 融合 │
│  ├─ Ontology: 类型化实体-关系图谱 + BFS + 双向时态  │
│  ├─ Storage: SQLite FTS5 + BLOB Vector           │
│  │   或 PostgreSQL + pgvector (企业部署)          │
│  ├─ Trust: 输出隔离 → grounding → consensus 审核   │
│  └─ MCP: 19 工具 (7读 9写 3复合) + graph_query    │
├─────────────────────────────────────────────────┤
│  SDK & API                                      │
│  ├─ Python SDK (pip install sagewiki)           │
│  ├─ TypeScript SDK (npm install sagewiki)       │
│  ├─ Go 嵌入式库 (pkg/sagewiki, 进程内 MCP)       │
│  └─ OpenAPI 3.1 规范 (api/openapi.yaml)          │
└─────────────────────────────────────────────────┘
```

**核心技术选型**：
- Go 纯后端（单一语言，零 CGO，跨平台一致行为）
- SQLite FTS5 (BM25 全文搜索) + BLOB 向量 (余弦相似度)
- 纯 Go PDF 提取 (`ledongthuc/pdf`)
- bubbletea + glamour TUI 终端仪表盘
- Preact + Tailwind CSS + force-graph Web UI，通过 `go:embed` 嵌入 (~1.2 MB)
- MCP 基于 `mark3labs/mcp-go`
- 约 33 MB 单二进制文件

### 2.2 llm_wiki 架构

```
┌─────────────────────────────────────────────────┐
│  Desktop App (Tauri v2)                         │
│  ├─ Frontend: React 19 + TypeScript + Vite 8    │
│  │   ├─ shadcn/ui + Tailwind CSS v4             │
│  │   ├─ sigma.js + graphology (图谱可视化)       │
│  │   ├─ Milkdown (WYSIWYG Markdown 编辑器)       │
│  │   ├─ Zustand (状态管理)                       │
│  │   └─ react-i18next (en/zh/it/ru)             │
│  ├─ Backend: Rust (src-tauri/)                  │
│  │   ├─ Agent Runtime (agent/) — 13 个模块       │
│  │   │   ├─ runtime.rs: 工具调用循环 (max 8轮)    │
│  │   │   ├─ tools.rs: 12+ 内置工具               │
│  │   │   ├─ skills.rs: SKILL.md 扫描与加载        │
│  │   │   ├─ provider.rs: 9 种 LLM 提供商          │
│  │   │   ├─ router.rs: 查询意图路由               │
│  │   │   ├─ permissions.rs: 能力权限模型          │
│  │   │   └─ workspace.rs: Agent 文件产出管理      │
│  │   ├─ Search: Tokenized + Vector (LanceDB)     │
│  │   ├─ API Server: localhost:19828              │
│  │   └─ Clip Server: localhost:19827             │
│  └─ Chrome Extension (Web Clipper, Manifest V3)  │
├─────────────────────────────────────────────────┤
│  External Integration                           │
│  ├─ MCP Server (Node.js, mcp-server/)            │
│  ├─ Agent Skill (llm_wiki_skill repo)            │
│  └─ Claude Code CLI / Codex CLI (子进程传输)     │
└─────────────────────────────────────────────────┘
```

**核心技术选型**：
- Tauri v2 (Rust 后端 + Web 前端混合)
- React 19 + TypeScript 5.7 + Vite 8
- shadcn/ui + Tailwind CSS v4 组件库
- sigma.js + graphology + ForceAtlas2 图谱可视化
- Milkdown (ProseMirror-based) WYSIWYG 编辑器
- LanceDB 嵌入式向量数据库（可选启用）
- 多格式解析：pdf-extract/pdfium-render, docx-rs, calamine, office_oxide/anydoc, epub, mobi
- tiny_http 内嵌 HTTP 服务器
- KaTeX + Mermaid 内容渲染

### 2.3 关键架构差异

| 维度 | sage-wiki | llm_wiki |
|------|-----------|----------|
| **核心语言** | Go（单一语言，零 CGO） | Rust (后端) + TypeScript/React (前端) + Node.js (MCP Server) |
| **存储引擎** | SQLite (FTS5 + BLOB Vector)，可选 PostgreSQL/pgvector | 文件系统 (Markdown + JSON) + LanceDB (可选向量) |
| **编译策略** | Pass 1 (摘要) → Pass 2 (概念提取+文章生成) → Pass 3 (三元组抽取，可选)；支持 Batch API、prompt caching、成本估算 | 两步 CoT Ingest（分析 → 生成），SHA256 增量缓存，持久化队列，崩溃恢复 |
| **搜索架构** | BM25 + Vector + Graph 三通道加权 RRF 融合 + LLM 查询扩展 + 重排序 + 覆盖度门控 | Tokenized (含 CJK 二元分词) + Vector (LanceDB) + Graph Expansion (4-Signal, 2-hop 衰减) + 上下文预算控制 (4K-1M tokens) |
| **知识图谱** | 类型化实体-关系图谱，triple extraction + entity resolution，evidenced edges (span/confidence/source_doc)，双向时态，矛盾无效化旧边 | 4-Signal Relevance Model (Direct Link ×3.0, Source Overlap ×4.0, Adamic-Adar ×1.5, Type Affinity ×1.0) + Louvain 社区发现 + Graph Insights |
| **部署方式** | 单 Go 二进制 + Docker 镜像 + 自托管，无外部运行时依赖 | 桌面安装包 (.dmg / .msi / .deb / .AppImage)，需要 Tauri 系统依赖 |
| **API 设计** | OpenAPI 3.1 规范，20 路由，Bearer 认证，结构化错误码，异步 Job API (202 + job_id)，Go/Python/TypeScript 三语言 SDK | localhost:19828 HTTP JSON API，Token 认证，SSE 流式 Agent Chat，MCP Server (Node.js 独立包) |
| **扩展性** | 外部解析器协议 (stdin/stdout)，Contribution Packs 注册表 | Chrome Extension Web Clipper，Skills 文件夹扫描，Claude Code CLI / Codex CLI 子进程传输 |

---

## 3. 项目与多项目管理

### 3.1 sage-wiki：独立目录 + Hub 联邦

**项目即目录**。`sage-wiki init my-wiki` 在任意路径创建一个自包含的项目：

```
my-wiki/
├── config.yaml           # providers, models, compiler, search, ontology
├── raw/                  # 放入源文件（文章、论文、代码、图片）
├── wiki/                 # 编译产出 —— Obsidian 兼容的 Markdown
│   ├── summaries/        # 每个源文件的 LLM 摘要
│   ├── concepts/         # 概念文章（知识图谱节点）
│   ├── images/           # 视觉描述的图片
│   ├── outputs/          # 已审核的查询答案归档
│   ├── under_review/     # 待审核的查询答案（默认隔离区）
│   └── archive/          # 已裁剪的文章
├── .sage/wiki.db         # 单一 SQLite 文件：FTS 索引、向量、本体、队列
└── .manifest.json        # 源文件↔文章映射 + 编译状态
```

项目之间**平等独立**，各自拥有完整的 `config.yaml` + `wiki.db`。可以分散在不同路径下，没有中心注册表。

**Vault Overlay 模式**：支持直接在已有 Obsidian vault 内初始化，不改变原有目录结构——`.sage/wiki.db` 和 `.manifest.json` 静默存在于 vault 根目录下：

```bash
cd ~/Documents/MyObsidianVault
sage-wiki init --vault    # .sage/wiki.db 出现在现有 vault 中
```

**Hub 联邦**：多个独立 wiki 可以通过 Hub 聚合，实现跨项目搜索和编译：

```bash
sage-wiki hub init my-hub          # 创建 hub
sage-wiki hub add /path/to/wiki1   # 将已有 wiki 加入 hub
sage-wiki hub add /path/to/wiki2
sage-wiki hub search "attention"   # 跨所有 wiki 聚合搜索
sage-wiki hub compile              # 跨所有 wiki 触发编译
sage-wiki hub status               # 查看各 wiki 编译状态
```

Hub 适合「不同团队维护各自 wiki，但偶尔需要跨库搜索」的场景（如多个产品线各自有知识库，架构师需要跨库查询）。

### 3.2 llm_wiki：类 Obsidian 多项目 + 场景模板

**项目完全隔离**，每个项目是独立的目录树，在 GUI 中管理：

```
my-wiki/
├── purpose.md              # 目标、关键问题、研究范围
├── schema.md               # Wiki 结构规则、页面类型
├── raw/
│   ├── sources/            # 上传的文档（不可变）
│   └── assets/             # 本地图片
├── wiki/
│   ├── index.md            # 内容目录
│   ├── log.md              # 操作历史
│   ├── overview.md         # 全局摘要（自动更新）
│   ├── entities/           # 人物、组织、产品
│   ├── concepts/           # 理论、方法、技术
│   ├── sources/            # 源文件摘要
│   ├── queries/            # 已保存的聊天回答 + 研究
│   ├── synthesis/          # 跨源综合分析
│   └── comparisons/        # 并排对比
├── .obsidian/              # Obsidian vault 配置（自动生成）
└── .llm-wiki/              # 应用配置、聊天历史、审核项
```

**场景模板**：创建项目时从 5 种预设中选择，每种预配置 `purpose.md` 和 `schema.md`：

| 模板 | 适用场景 |
|------|----------|
| Research | 长期深度研究某个主题 |
| Reading | 读书笔记、章节追踪 |
| Personal Growth | 个人目标、健康、心理 |
| Business | 竞品分析、客户调研 |
| General | 通用知识管理 |

**项目间完全隔离**：无跨项目搜索、无联邦机制。每个项目的设置（LLM 提供商、Web 搜索、嵌入配置）独立持久化。

**项目迁移**：支持 ZIP 导出/导入，用于跨设备迁移或备份。

**Chrome Web Clipper 项目选择器**：裁剪网页时弹出下拉菜单选择目标项目，支持多项目并行使用。

### 3.3 对比总结

| 维度 | sage-wiki | llm_wiki |
|------|-----------|----------|
| **项目创建** | CLI: `sage-wiki init <dir> [--vault] [--pack]` | GUI: 创建项目 → 选场景模板 |
| **项目存储** | 目录 + 单一 SQLite 文件 (`.sage/wiki.db`) | 目录 + Markdown 文件 + JSON 配置 (`.llm-wiki/`) |
| **项目间关系** | 平等独立，可选 Hub 联邦聚合 | 完全隔离，无跨项目功能 |
| **跨项目搜索** | ✅ Hub 联邦聚合搜索 + 编译 | ❌ 各自独立 |
| **Vault Overlay** | ✅ `--vault` 覆盖已有 Obsidian vault | ❌ 每个项目是独立新目录 |
| **Obsidian 兼容** | 产出目录可直接作为 Obsidian vault 打开 | 自动生成 `.obsidian/` 配置，同为 Obsidian 兼容 |
| **项目迁移** | 目录直接复制（含 `.sage/wiki.db`） | ZIP 导出/导入 |
| **多项目管理界面** | CLI `hub` 命令族 | GUI 项目列表 + 设置面板 |
| **场景模板** | Contribution Packs（8 个领域包，init 时 `--pack` 选择） | 5 种场景模板（创建时选择） |

**设计哲学差异**：
- sage-wiki 把项目看作**可联邦的数据后端**——每个 wiki 是自治的，Hub 在它们之上提供聚合层，适合需要跨团队/跨项目知识检索的组织场景
- llm_wiki 把项目看作**互不相关的个人空间**——每个项目是独立的 Obsidian 式知识库，适合个人管理多个领域的知识（工作 vs 学习 vs 生活）

---

## 4. 各自特色

### 4.1 sage-wiki 独有特色

1.  **Graph Memory 与 Evidence 体系**
    - 每个关系携带 `evidence`（支持性文本片段）、`confidence`（0-1）、`source_doc`
    - 结论可追溯到具体句子，而非整篇文档
    - 双向时态图：`as_of` 查询可回答 "我们一月份认为什么？"
    - 矛盾处理：新事实使旧边无效（invalidate），而非冲突覆盖

2.  **Output Trust 机制**
    - 查询输出默认隔离（quarantine）
    - 三级审核流程：grounding verification → consensus → manual promotion/demotion
    - `under_review/` 和 `outputs/` 双目录管理

3.  **Entity Resolution（实体解析）**
    - "K8s" 和 "Kubernetes" 自动合并为同一规范节点
    - 阈值 0.85 以上自动应用，1.0 表示仅人工审核
    - 精确可逆：`ontology resolve --unlink`

4.  **Tiered Compilation（分层编译）**
    - Tier 0: 免费 FTS5 索引 (~5ms/文档)
    - Tier 1: 廉价向量嵌入 (~$0.00002, ~200ms/文档)
    - Tier 2: 代码解析器，无 LLM 成本 (~10ms/文档，10 种语言)
    - Tier 3: 全量 LLM 编译 (~$0.05-0.15, ~5-8 分钟/文档)
    - 100K 文档 Tier 1 索引约 5.5 小时，Tier 3 按需自动升级

5.  **Contribution Packs（贡献包）**
    - 8 个预设领域包：academic-research, software-engineering, product-management, personal-knowledge, study-group, meeting-organizer, content-creation, legal-compliance
    - 每个包包含：本体类型、提示词模板、技能触发器
    - Git 注册表，事务性应用 (apply/rollback)，配置白名单安全

6.  **Client SDK 三件套**
    - Python: `pip install sagewiki` (httpx, ≥3.9)
    - TypeScript: `npm install sagewiki` (零依赖, Node/Deno/Bun/Edge)
    - Go: `pkg/sagewiki` 进程内嵌入（无子进程、无端口管理）
    - CI 验证示例：LangGraph 记忆节点 + Vercel AI SDK 工具

7.  **多团队协作模式**
    - Git 同步：wiki 文件入 git，多成员拉取合并
    - 共享服务器：自托管 + 认证 + 反向代理
    - Hub 联邦：多项目聚合搜索和编译
    - 团队信任审核：output trust 在团队上下文中的扩展

8.  **Scribe 会话转录**
    - 从 Claude Code JSONL 转录中提取知识实体
    - 可扩展的转录接口（支持新格式）

9.  **Benchmark 评测体系**
    - 记忆评测：LOCOMO 92.0%, LongMemEval-S 93.3%, BEAM 100K 0.691 (gpt-5 judge)
    - 质量+性能评测：中位数 87.4%，搜索 recall@10 100%，FTS5 top-10 0.035ms

10. **纯 Go 零 CGO**：单二进制部署，无系统依赖，Docker 镜像 ~20MB (Alpine)

### 4.2 llm_wiki 独有特色

1.  **桌面 GUI 体验**
    - 三栏可拖拽布局：知识树/文件树 (左) + 聊天 (中) + 预览 (右)
    - 图标侧边栏八模块：Wiki, Sources, Search, Graph, Lint, Review, Deep Research, Settings
    - Activity Panel 实时显示文件级 ingest 进度
    - 所有状态持久化：对话、设置、Review 项、项目配置

2.  **两步 Chain-of-Thought Ingest**
    - Step 1 (分析): LLM 读取源文件 → 提取关键实体、概念、论证、与已有知识的连接和矛盾
    - Step 2 (生成): 基于分析结果生成 wiki 页面、更新 index.md、log.md、overview.md
    - SHA256 增量缓存：未修改文件自动跳过
    - 持久化摄入队列：崩溃恢复、取消、重试（最多 3 次）

3.  **purpose.md —— Wiki 的灵魂**
    - 定义目标、关键问题、研究范围、演进中的论点
    - LLM 在每次 ingest 和 query 时读取此文件作为上下文
    - LLM 可根据使用模式建议更新
    - 与 schema.md（结构规则）互补：purpose 是方向意图，schema 是结构约束

4.  **多模态图片摄入**
    - 从 PDF/Office 文档提取嵌入图片
    - 视觉 LLM 生成事实性描述
    - 图片感知搜索结果，带 lightbox 预览和跳转源
    - 图片存储于 `wiki/media/`

5.  **4-Signal Knowledge Graph + Louvain 社区发现**
    - 四种信号加权：Direct Link (×3.0), Source Overlap (×4.0), Adamic-Adar (×1.5), Type Affinity (×1.0)
    - Louvain 算法自动发现知识集群，凝聚力评分 (cohesion < 0.15 警告)
    - 12 色调色板可视化，悬停高亮邻居节点
    - 位置缓存防止布局抖动

6.  **Graph Insights —— 惊喜连接与知识缺口**
    - **惊喜连接**：跨社区边、跨类型链接、peripheral↔hub 耦合，复合惊喜评分
    - **知识缺口**：孤立页面 (degree ≤ 1)、稀疏社区 (cohesion < 0.15)、桥接节点 (连接 3+ 集群)
    - 交互式：点击高亮图中对应节点和边，一键触发 Deep Research

7.  **Deep Research**
    - LLM 优化研究主题（读取 overview.md + purpose.md 获取上下文）
    - 多查询网络搜索：Tavily, SerpApi, SearXNG, Firecrawl, Brave, Bocha
    - 全内容提取（无截断），自动合成 wiki 研究页面
    - 3 并发任务队列，实时流式进度
    - 可编辑确认对话框（修改主题和搜索查询后开始）

8.  **Chrome Web Clipper 扩展**
    - Manifest V3，专用扩展
    - Readability.js 精确文章提取 + Turndown.js HTML→Markdown
    - 项目选择器（多项目支持）
    - 本地 HTTP API (port 19827) 通信，自动触发两步入库
    - Alt+Shift+L 快捷键裁剪

9.  **Async Review System（异步审核）**
    - LLM 在 ingest 时标记需人工判断的项目
    - 预定义动作类型（Create Page, Deep Research, Skip）—— 防止 LLM 幻觉任意动作
    - 预生成搜索查询（ingest 时即生成）
    - 不阻塞 ingest 流程，用户随时处理

10. **Rust Backend Chat Agent + Skills**
    - 工具调用循环运行在 Rust 后端（非浏览器 TypeScript 循环）
    - 最多 8 轮迭代，去重守卫，无进展检测
    - Skills 系统：扫描项目/用户技能文件夹，`/skill` 切换
    - User Input Forms：技能可请求结构化用户输入（单选/多选/自由文本）

11. **Claude Code CLI / Codex CLI 作为 LLM 提供商**
    - 通过子进程 (`claude -p --output-format stream-json --input-format stream-json`) 复用本地订阅
    - 同等的 Codex CLI 传输 (`codex exec --json`)
    - OAuth 凭证复用，无需额外 API Key

12. **丰富的内容渲染**
    - KaTeX 数学公式：inline `$...$` 和 block `$$...$$`，自动包装 LaTeX 环境
    - Mermaid 图表：流程图、序列图等直接在聊天和预览中渲染
    - Thinking/Reasoning 显示：`<think>` 块折叠，流式 5 行渐隐滚动

13. **Cascade 清理机制**
    - 删除源文件 → 自动删除 wiki 摘要页面
    - 三种匹配方法：frontmatter `sources[]`、页面名称、章节引用
    - 共享实体保留：只从 `sources[]` 移除被删源，不删除整个实体页面
    - 死 wikilink 清理

14. **文件夹 Auto-Watch**
    - 检测 `raw/sources/` 的外部变更（新增、编辑、删除）
    - 自动复用相同的 ingest/delete 生命周期

15. **场景模板**
    - Research、Reading、Personal Growth、Business、General
    - 每种预配置 purpose.md 和 schema.md

### 4.3 共性

| 维度 | 说明 |
|------|------|
| **核心理念来源** | 都源于 Andrej Karpathy 的 LLM Wiki 模式 (`llm-wiki.md`) |
| **三层架构** | Raw Sources (不可变) → Wiki (LLM 生成) → Schema (规则配置) |
| **Obsidian 兼容** | Markdown + `[[wikilinks]]` + YAML frontmatter，wiki 目录可直接作为 Obsidian vault |
| **MCP 集成** | 都提供 MCP Server，可被 Claude Code 等外部 Agent 调用 |
| **Agent Skill 文件** | 都提供生成的技能文件，教 Agent 何时/如何查询知识库 |
| **多 LLM 提供商** | 都支持 OpenAI、Anthropic、Google Gemini、Ollama 等 |
| **多格式源文档** | 都支持 PDF、DOCX、PPTX、XLSX、EPUB、Markdown、纯文本等 |
| **知识图谱** | 都有知识图谱构建和查询能力（实现方式不同） |
| **Web Search** | 都集成外部网络搜索 |
| **操作日志** | 都有操作历史记录（log.md） |
| **Index 文件** | 都有 index.md 作为内容目录和 LLM 导航入口 |

---

## 5. llm_wiki 的 Agent 能力深度分析

### 5.1 内嵌 Agent 架构

**是的，llm_wiki 内嵌了一个完整的 Agent 运行时**，位于 `src-tauri/src/agent/`（纯 Rust 实现），包含 13 个模块：

```
agent/
├── mod.rs          # 子系统入口：声明共享于 UI、API、MCP 之间
├── types.rs        # AgentChatRequest/Response, AgentMode, AgentToolEvent 等类型
├── runtime.rs      # AgentRuntime: 核心工具调用循环（~5700 行）
├── tools.rs        # 12+ 内置工具注册与执行
├── skills.rs       # SKILL.md 扫描、加载、激活（多层次技能根目录）
├── context.rs      # 上下文组装（purpose.md, schema.md, index.md, 对话历史…）
├── router.rs       # 查询意图路由
├── provider.rs     # 9 种 LLM 提供商抽象层
├── events.rs       # 流式事件：tool_call, tool_result, message, thinking, user_input…
├── permissions.rs  # 能力权限模型 (Capability-based)
├── session.rs      # 对话会话持久化
├── workspace.rs    # Agent 工作空间文件管理
├── cancel.rs       # 取消令牌注册表
```

### 5.2 Agent 工作流程

```
用户消息
  → AgentRuntime 组装上下文
    （purpose.md + schema.md + index.md + 对话历史 + 技能指令）
  → LLM 调用（配置的提供商）
  → 解析响应（可能包含 tool_calls JSON）
  → Router 匹配工具 → 执行工具
    （wiki.search / source.search / graph.search / web.search /
     workspace.write_file / shell.exec …）
  → 工具结果反馈给 LLM
  → 多轮迭代（最多 8 轮，含去重守卫和无进展检测）
  → 流式事件推送到前端
    （thinking 显示 → tool_call/tool_result → agent 消息增量）
```

### 5.3 Agent 工具集

| 工具 | 分类 | 说明 |
|------|------|------|
| `wiki.search` | Read | 关键词 + 向量搜索 wiki 页面 |
| `wiki.read_page` | Read | 按路径读取 wiki 页面（含知识上下文/反向链接） |
| `wiki.write_page` | Write | 在 `wiki/` 下创建 Markdown 页面 |
| `source.search` | Read | 对原始源文件的 grep 式搜索 |
| `graph.search` | Read | 图谱遍历，查找实体关系 |
| `web.search` | Network | 外部网络搜索（Tavily/SerpApi/SearXNG/Firecrawl/Brave/Bocha/Ollama） |
| `anytxt.search` | Read | AnyTXT 桌面搜索引擎集成 |
| `deep_research.run` | Compound | 编排更广泛的证据收集 |
| `workspace.write_file` | Write | 在 `agent-workspace/` 创建文件 |
| `workspace.append_file` | Write | 追加到 workspace 文件 |
| `shell.exec` | Process | 受控 shell 命令执行（需审批，30s 超时） |
| `skill.read_file` | Read | 从激活的技能目录延迟读取参考文件 |
| `llm.generate` | Compound | 最终结构化答案生成 |

**安全边界**（在 Rust 后端强制执行，非 UI 层）：
- 文件读取限制：2 MB/页
- 文件写入限制：2 MB/页
- Shell 输出限制：20,000 字符
- Shell 命令长度限制：4,000 字符
- Web 搜索超时：30 秒
- 最多 8 轮工具迭代

### 5.4 Agent 模式

| 模式 | 说明 |
|------|------|
| **Fast** | 快速响应，最少工具调用 |
| **Standard** | 标准模式，常规工具使用 |
| **Deep** | 深度模式，更广泛的证据收集 |
| **LocalFirst** | 本地优先，优先使用 wiki 和源文件，减少网络调用 |

**检索模式**：
- **Standard**：混合检索
- **Smart**：迭代式智能检索
- **Faithful** (Read Sources Only)：仅从原始导入材料中回答

### 5.5 Skills 系统

llm_wiki 的 Agent 支持从多个根目录扫描和加载 SKILL.md 文件：

```
技能根目录（优先级从高到低）：
├── .llm-wiki/skills/          # 项目级技能
├── ~/.claude/skills/          # Claude Code 技能
├── ~/.codex/skills/           # Codex 技能
└── ~/.agents/skills/          # 通用 Agent 技能
```

- 技能通过 `/skill` 命令激活
- 技能可声明所需工具权限：`shell.exec`, `workspace.write_file`, `skill.read_file`
- User Input Forms：技能可请求结构化用户输入（最多 12 字段，8 选项，400 字符文本）

### 5.6 与 Claude Code / Pi 等 Agent 的区别

| 维度 | llm_wiki Agent | Claude Code | Pi (OpenCode) |
|------|---------------|-------------|---------------|
| **定位** | 垂直领域知识工作 Agent：专为 wiki 检索、写入、深度研究设计 | 通用编程 Agent：代码编写、调试、重构、文件操作 | 通用编程 Agent：Karpathy 原文档提到的 CLI Agent 之一 |
| **运行环境** | 嵌入在桌面应用内，随应用启动，Rust 原生执行 | 独立 CLI 进程 (`claude`)，在终端中运行 | 独立 CLI 进程 |
| **工具范围** | 领域限定：wiki 搜索/读写、源文件、图谱、web 搜索、workspace、受控 shell | 通用：文件系统读写、shell 执行、git 操作、MCP 工具、代码编辑、浏览器 | 通用：文件系统、shell、代码编辑 |
| **文件系统访问** | 限制在项目目录内 | 可访问整个文件系统（受权限策略控制） | 可访问整个文件系统 |
| **迭代深度** | 最多 8 轮工具调用，含去重守卫 | 多轮对话，无硬性迭代限制（有 token 预算限制） | 多轮对话 |
| **权限模型** | 能力模型：项目 workspace 自动放行，外部 shell 需审批 | 分层权限 (allow/deny/ask)，可持久化到 settings | 取决于具体实现 |
| **LLM 提供商** | 9 种：OpenAI, Anthropic, Google, Azure, Ollama, MiniMax, Custom, **Claude Code CLI**, **Codex CLI** | 自身就是 LLM 客户端，使用 Anthropic API | 取决于配置 |
| **与其他 Agent 的关系** | **同时是工具和调用者**：提供 MCP Server + Agent Skill 供外部 Agent 调用；也可通过子进程调用 Claude Code CLI 作为 LLM 后端 | 可通过 MCP 连接 llm_wiki 的 MCP Server，查询其知识库 | 可通过 MCP 连接 llm_wiki |
| **技能系统** | 扫描多层 SKILL.md，`/skill` 激活，支持 User Input Forms | SKILL.md 技能系统，`/skill` 或自动触发 | 取决于具体实现 |
| **流式事件** | SSE 流式推送 tool_call/tool_result/thinking/message 等事件到前端 UI | 终端流式输出 | 终端流式输出 |

**核心区别总结**：
- llm_wiki Agent 是**垂直领域的知识工作 Agent**（"帮我搜索 wiki、写页面、做深度研究"），运行在桌面应用的 Rust 后端中
- Claude Code 是**通用编程 Agent**（"帮我写代码、修 bug、重构项目"），是独立的 CLI 工具
- Pi 是**通用编程 Agent**（Karpathy 原文档中作为示例提到的 CLI Agent）
- llm_wiki **同时是被外部 Agent 调用的工具**（通过 MCP Server + Agent Skill），形成嵌套调用链：「Claude Code → MCP → llm_wiki API → 查询知识库」
- llm_wiki **也可以反向调用 Claude Code**（作为 LLM 提供商，通过子进程传输，复用本地订阅），形成双向关系

**llm_wiki Agent 的独特价值**：
- 内置在知识管理桌面应用中，用户无需离开 GUI
- 专为知识工作优化的工具集（图谱搜索、源文件追溯、wiki 写入、Deep Research）
- Skills 系统使 Agent 行为可定制（不同项目加载不同技能）
- 与 Deep Research、Graph Insights、Review System 深度集成
- User Input Forms 机制让技能可以收集结构化用户反馈
- 双模式：既是被外部 Agent 调用的知识库，也能通过子进程复用 Claude Code CLI 的能力

**局限性**：
- 最大 8 轮工具迭代限制了复杂多步任务
- 无代码编辑能力（不能修改项目源代码）
- Shell 执行受限（需审批，30s 超时，输出截断）
- 不能自主安装依赖或修改系统配置
- 文件系统访问限定在项目目录内

---

## 6. 总结：选型建议

| 场景 | 推荐 |
|------|------|
| 需要为 AI Agent（Claude Code 等）构建长期记忆层 | **sage-wiki** — 19 MCP 工具，Graph Memory，SDK 支持 |
| 个人桌面端知识管理，希望开箱即用 | **llm_wiki** — 桌面 GUI，Chrome 裁剪，Deep Research |
| 团队共享知识库，需要审核和信任机制 | **sage-wiki** — Git 同步/共享服务器/Hub 联邦，Output Trust |
| 需要 SDK 集成到现有应用 | **sage-wiki** — Python/TS/Go 三语言 SDK + OpenAPI |
| 需要可视化图谱探索和社区发现 | **llm_wiki** — sigma.js 交互式图谱，Louvain 社区检测，Graph Insights |
| 需要大规模 (100K+ 文档) 成本控制 | **sage-wiki** — Tiered Compilation，Tier 0-2 几乎零成本 |
| 需要多模态图片理解和搜索 | **llm_wiki** — PDF 图片提取 + 视觉 LLM 描述 + 图片搜索 |
| 需要企业级部署 (PostgreSQL, 认证, 指标) | **sage-wiki** — PostgreSQL/pgvector, /metrics, Bearer 认证 |
| 需要 Web 裁剪和一键知识摄入 | **llm_wiki** — Chrome Extension Web Clipper + Auto-Ingest |

两个项目可以互补使用：用 llm_wiki 作为日常桌面知识管理前端，用 sage-wiki 作为团队/Agent 的后端记忆引擎。
