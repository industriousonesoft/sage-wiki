# sage-wiki vs llm_wiki 搜索框架对比分析

本文深入对比两个项目的搜索架构设计，覆盖索引粒度、查询扩展、重排序、图谱增强、跨语言、上下文预算、性能可扩展性等维度。

> 数据来源：sage-wiki (v0.2.6) 和 llm_wiki (v0.6.8)，基于截至 2026-08-12 的代码仓库。

---

## 一、架构总览

### sage-wiki：多阶段、多通道、LLM 增强的工业级检索管线

```
Query
  → 强信号探测（BM25 top-1 ≥0.4 且 2× 差距则跳过扩展）
  → LLM 查询扩展（lex×2 + vec×1 + hyde×1，共 4 变体）
  → 并行三通道（每变体）：
     ├─ BM25 块级 FTS5 + BM25 文档级 FTS5
     ├─ Vector 块级 + Vector 文档级（余弦相似度）
     └─ Graph 邻近度（本体图谱，≤2 hops，按关系类型加权）
  → 加权 RRF 文档级融合（w_channel / (60 + rank)）
  → 水化补全（仅 ID 的腿回填文本）
  → LLM 重排序（top-15 候选，400 tokens/候选，8000 tokens 预算）
  → 位置感知混合（检索权重 vs 重排权重按排名分档）
  → 覆盖门控（重排器需 ≥50% 覆盖才生效）
  → Tag 过滤/加权 → 时效打破平局 → Trust 规则 → 截断
  [Q&A 专属]
  → Graph 上下文扩展（4-Signal 相关性，2-hop）
  → Token 预算贪婪填充（主结果 → 图谱扩展 → depth-1 遍历）
  → LLM 合成答案
```

**核心模块**：`internal/search/`（Go），`internal/memory/`（FTS5 + 向量），`internal/ontology/`（图谱遍历），`internal/llm/`（扩展+重排）

### llm_wiki：简洁、零 LLM 依赖的经典检索管线

```
Query
  → 分词（CJK 二元 + 中英停用词 + Unicode 回退）
  → 关键词打分（标题 +10 / 文件名精确 +200 / 标题短语 +50 / 内容出现 +20×次数）
  ──→ RRF 融合 (k=60)
  → Vector（LanceDB，可选，余弦相似度）
  ──→
  → Graph 扩展（1-hop wikilink 邻接，1/(rank+1) 加权，限额 15%-30%）
  → 截断（limit）
  [Chat 专属]
  → 上下文预算（字符级：5% 索引 + 50% 页面 + 30% 历史+系统 + 15% 响应保留）
  → 填充 → LLM 生成
```

**核心模块**：`src-tauri/src/commands/search.rs`（Rust 后端搜索），`src-tauri/src/commands/vectorstore.rs`（LanceDB），`src/lib/search.ts`（TypeScript 前端分词），`src/lib/graph-relevance.ts`（图谱相关性），`src/lib/context-budget.ts`（预算分配）

---

## 二、分维度利弊分析

### 2.1 索引粒度

| | sage-wiki | llm_wiki |
|---|---|---|
| **方式** | 块级 (~800 tokens/chunk) + 文档级双索引 | 整页文档级索引 |
| **全文搜索** | FTS5 (BM25)，每块独立条目 | 文件系统遍历 (walkdir)，无 FTS |
| **向量存储** | SQLite BLOB 向量 + 内存缓存 + 可选 HNSW ANN | LanceDB（嵌入式向量数据库） |

**sage-wiki 优势**：
- 块级索引使长文档内的精确段落可被检索（如 3000 词的 "Transformer Architecture" 文章中找到 "flash attention" 段落，而非整篇返回）
- FTS5 提供 BM25 相关性排序（TF-IDF 的改进），远比简单词频匹配精确
- 文档级 + 块级双粒度设计：同一文档多处命中可累积得分，文档级匹配和块级匹配互为验证

**sage-wiki 劣势**：
- 需要编译步骤建立索引，新添加的文档不能即时搜索
- 块级索引占用更多存储（每个 ~800 token 块一个 FTS5 + 向量条目）
- 重索引操作需要显式触发

**llm_wiki 优势**：
- 零索引建立时间：文件系统即数据源，新增文件立即可搜索
- 架构极简：walkdir 遍历，无额外存储开销
- 无需维护索引与源文件的一致性

**llm_wiki 劣势**：
- 每次搜索需遍历全部 Markdown 文件（上限 10,000），大 wiki 下 I/O 线性增长（万文件约 50-200ms）
- 无 BM25，纯词频/位置打分，长文档中词频高的噪声内容得分偏高
- 长文档内部无法精准定位：搜索命中整篇，但不知道在哪个段落

---

### 2.2 查询扩展 (Query Expansion)

| | sage-wiki | llm_wiki |
|---|---|---|
| **方式** | LLM 生成 4 种变体（lex×2 + vec×1 + hyde×1） | 无 |
| **触发条件** | 仅 `sage-wiki query` 默认启用；搜索表面（MCP/CLI/API）默认关闭，按调用 opt-in | N/A |
| **前置检查** | 强信号探测：top BM25 归一化得分 ≥0.4 且与第二名差距 ≥2× 则跳过扩展 | N/A |

**sage-wiki 优势**：
- **HyDE**（Hypothetical Document Embedding，假设答案嵌入）：将查询转为假设答案段落，捕捉语义高度相关但零词汇重叠的内容，这是纯关键词搜索做不到的
- Lex 变体提升 BM25 召回（用不同的关键词组合命中不同文档），Vec 变体提升向量召回
- 强信号短路机制避免了简单查询浪费 LLM 调用（如搜索 "kubernetes" 没必要扩展）

**sage-wiki 劣势**：
- 每次触发多一次 LLM 调用（~1-3s 延迟，~$0.001-0.005 成本）
- 只在 Q&A 路径生效（`sage-wiki query`），MCP `wiki_search` 等工具默认不启用
- 对实时性要求高的场景（如 TUI 即时搜索）不适用

**llm_wiki**：无需 LLM 调用，查询延迟更低（毫秒级），零成本。但无法处理「用不同表述找同一概念」的场景（如搜索 "GPU memory optimization" 找不到 "SRAM tiling" 相关内容）。

---

### 2.3 重排序 (Re-ranking)

| | sage-wiki | llm_wiki |
|---|---|---|
| **方式** | LLM 对 top-15 候选逐一打分，与原始检索分数按位置感知权重混合 | 无 |
| **混合权重** | 排名 1-3: 检索权重 75% / 重排权重 25%；排名 4-10: 60%/40%；排名 11+: 40%/60% | N/A |
| **覆盖门控** | 重排器需覆盖 ≥50% 候选才应用混合；低于此阈值回退纯 RRF 排序 | N/A |
| **候选截断** | 每个候选截断到 400 tokens，总重排预算 8000 tokens | N/A |

**sage-wiki 优势**：
- **位置感知混合**：排名越靠前，检索信号权重越大——防止 LLM 把高置信度的 #1 结果误排到末尾
- 覆盖门控防止 LLM 输出不完整（如只给 3 个打分而漏了 12 个）时破坏排序
- 重排序后得分归一化到 [0,1] 区间，便于后续阈值过滤

**sage-wiki 劣势**：
- 多一次 LLM 调用（~2-5s 延迟）
- Ollama 等本地模型默认自动禁用重排序（结构化 JSON 输出不够稳定）
- 每候选截断到 400 tokens 可能丢失关键上下文，影响重排准确性

**llm_wiki**：无重排序意味着最终排序完全依赖关键词打分 + RRF 融合，对语义相近但词频不突出的内容排序能力弱。例如一篇讲 "backpropagation" 的文章，如果全文使用 "error back-propagation through computational graphs" 而没有反复出现 "backpropagation"，关键词打分偏低。

---

### 2.4 图谱增强 (Graph Enhancement)

| | sage-wiki | llm_wiki |
|---|---|---|
| **图谱类型** | 类型化实体-关系图（triple extraction + entity resolution），边携带 evidence span/confidence/source_doc，双向时态 | wikilink `[[链接]]` 无向邻接图 |
| **构建方式** | LLM 编译时 Pass 2（概念提取）+ Pass 3（三元组抽取，opt-in）；基础模式使用关键词共现 | 解析 Markdown 中的 `[[wikilinks]]` |
| **扩展方式** | 4-Signal 相关性评分 + 2-hop BFS 遍历，按关系类型可配置权重 | 1-hop wikilink 邻接，1/(rank+1) 加权，限额 15%-30% |
| **作为检索通道** | 是，作为第三通道直接参与 RRF 融合（默认 weight: 0.2） | 否，仅在关键词/向量结果之后追加 |

**sage-wiki 优势**：
- **图谱是一等检索通道而非补充**：一个仅靠图谱遍历就能回答的多跳问题（"Kubernetes 用到的所有网络技术"），不需要任何关键词匹配
- **Source Overlap (×4.0) 是最强信号**：同一篇源文档产生的多篇文章天然语义相关，且此信号零计算成本（仅对已有 ontology `cites` 边做集合交集）
- Entity resolution 使别名命中后从整个集群的邻居扩展（"K8s" 命中 → 扩展 "Kubernetes" 的全部关联实体）
- 类型化边支持按关系类型加权遍历（`contradicts` 1.1, `cites` 0.7），全部可配置
- `wiki_graph_query` 专用于多跳关系问答，每个引用携带源文档出处和置信度

**sage-wiki 劣势**：
- 图谱质量依赖 LLM 编译质量。未启用 `ontology.triples` 时仅基于关键词共现构建图谱，准确性下降
- 三元组抽取每文档增加一次 LLM 调用（opt-in，默认不消耗 API key）
- 图谱通道权重需手动调优，错误的权重配置可能引入噪声

**llm_wiki 优势**：
- **图谱构建零成本**：直接解析 Markdown 中的 `[[wikilinks]]`，无需 LLM 调用
- **即时可用**：新增页面中的 wikilink 立即可在图谱中生效
- **4-Signal Relevance Model** 在 TypeScript 前端实现（`graph-relevance.ts`），算法迭代灵活
- Louvain 社区发现和 Graph Insights 提供更高层次的知识结构洞察（惊喜连接、知识缺口）

**llm_wiki 劣势**：
- wikilink 图谱质量完全依赖页面编写质量——如果 LLM 在生成 wiki 页面时未添加足够的 `[[交叉引用]]`，图谱会大面积空洞
- 1-hop 邻接扩展过于简单：只能发现「A 链接到 B」的直接关系，无法发现「A 和 C 都链接到 B」的隐含关系
- 图谱结果仅追加（占最终结果 15%-30%），不参与核心排序，信号强度弱
- 无 evidence / 溯源信息，无法区分「强关联」和「偶然提及」

---

### 2.5 跨语言搜索

| | sage-wiki | llm_wiki |
|---|---|---|
| **核心机制** | 全量暴力余弦搜索（独立于 BM25），多语言查询可命中零词汇重叠的跨语言内容 | LanceDB 余弦相似度（依赖嵌入模型的多语言能力） |
| **分词** | FTS5 默认分词器 + corpus-adaptive stopwording（基于语料统计自动调整停用词） | CJK 二元分词（相邻字符组对）+ 中英停用词 + Unicode 字符级回退 |
| **显式支持** | 文档中明确声明跨语言搜索能力（Polish query → English content） | 取决于嵌入模型选择 |

**sage-wiki 优势**：
- 显式设计跨语言场景：即使查询语言和内容语言完全不同（如波兰语查询英文文章），矢量余弦搜索仍能找到语义匹配
- 暴力搜索保证召回，不依赖分词器或模型的特定语言能力
- FTS5 的 corpus-adaptive stopwording 自动适应不同语言的词频分布

**sage-wiki 劣势**：
- 暴力余弦搜索在 100K+ 向量规模时计算开销大（HNSW ANN 可缓解，需手动启用）
- 跨语言向量搜索的质量取决于嵌入模型的多语言对齐能力

**llm_wiki 优势**：
- CJK 二元分词 + 停用词设计对中英混合查询特别友好
- 文件名精确匹配（+200 分）和标题短语匹配（+50 分）能快速定位中文标题文章
- Unicode 字符级回退确保极端情况（如特殊符号查询）不崩溃

**llm_wiki 劣势**：
- 嵌入模型的多语言能力决定跨语言搜索质量上限
- CJK 二元分词虽覆盖中文，对韩语、泰语等语言可能效果不佳
- 无显式的跨语言设计——纯靠嵌入模型隐式对齐

---

### 2.6 上下文预算与 Q&A 整合

| | sage-wiki | llm_wiki |
|---|---|---|
| **预算单位** | Token（精确） | Character（近似） |
| **默认预算** | 8000 tokens | 204,800 chars (~50K tokens，按英文估算) |
| **分配策略** | 按组合分数贪婪填充（主结果 → 图谱扩展 → depth-1 遍历），每篇截断 4000 tokens | 固定比例：5% 索引 + 50% 页面 + 30% 历史+系统 + 15% 响应保留 |
| **页面截断** | 4000 tokens / 篇 | 随 pageBudget 缩放（原硬编码 30,000 chars） |

**sage-wiki 优势**：
- **Token 级精确控制**：不会因字符估算偏差导致上下文超限被截断
- 三级优先级（主结果 ×3 > 图谱扩展 ×2 > depth-1 遍历 ×1）确保最相关内容优先填充
- 每篇 4000 tokens 上限防止单篇超长文章「吃光」全部预算
- 700+ chars 的内容标注长度便于前端显示

**sage-wiki 劣势**：
- Token 估算采用 chars/4 近似，对 CJK 内容偏差较大（中文字符 ≈ 0.5-1 token）
- 配置参数较多（`context_max_tokens`, `chunk_size`, `chunk_overlap_tokens`），学习成本高

**llm_wiki 优势**：
- **配置极简**：用户通过滑块 4K-1M 调节，自动计算分配
- 响应保留（15%）确保 LLM 有足够空间生成答案而非被上下文占满
- `context-budget.ts` 作为独立模块，纯函数，有单元测试覆盖

**llm_wiki 劣势**：
- **字符级估算在 CJK 内容中偏差大**：中文字符 ≈ 0.5-1 token，英文 ≈ 0.25 token，中英混合时实际 token 消耗不可预测
- 硬编码 5%/50%/30%/15% 分割缺乏场景灵活性——索引占比在 100 页时偏少，在 10,000 页时浪费
- 每页最大字符数随 pageBudget 线性缩放，极端大窗口下可能让单篇噪声文章占满预算
- 默认 204,800 chars 在仅 4K 上下文的模型上不适用，需用户手动调低

---

### 2.7 性能与可扩展性

| | sage-wiki | llm_wiki |
|---|---|---|
| **小规模 (< 1K doc)** | FTS5 top-10: 0.035ms，混合 RRF: 4.9ms | walkdir 遍历: ~5-20ms |
| **中等规模 (~10K doc)** | 混合 RRF: 4.9ms，Graph BFS: 0.001ms | walkdir 遍历 + 内容读取: ~50-200ms |
| **大规模 (100K doc)** | Tier 0/1 索引 + HNSW ANN，持续可用 | **MAX_SEARCH_FILES = 10,000 硬限制**，超出截断 |
| **向量加速** | 进程内内存矩阵（归一化预加载），比 SQLite 扫描快 ~11× | LanceDB 嵌入式 ANN 检索 |
| **编译时间** | 100K 文档 Tier 1 索引 + 嵌入约 5.5 小时 | 无编译，零等待 |

**sage-wiki 优势**：
- FTS5 倒排索引 + 内存向量缓存 + HNSW ANN 确保万级文档毫秒响应
- **无文件数硬限制** —— 100K 文档通过分层编译（Tier 0-3）依然可用
- Graph BFS 遍历 0.001ms 极快
- 向量内存缓存使频繁搜索零磁盘 I/O

**sage-wiki 劣势**：
- 首次编译 100K 文档需要数小时
- 内存向量缓存在向量规模超过可用 RAM 时会退化
- 跨进程向量写入不一致：CLI 编译后的向量，运行中的 MCP 服务器需重启才能看到

**llm_wiki 优势**：
- **零编译，零等待**：安装即用，新文档拖入即可搜索
- LanceDB 作为嵌入式向量数据库，无需额外进程
- 小规模 (< 1000 文档) 体验极佳

**llm_wiki 劣势**：
- **MAX_SEARCH_FILES = 10,000 硬限制**：超过后搜索结果不完整且仅在 stderr 输出日志，无 UI 警告
- walkdir 遍历 + 逐文件读取在大规模下 I/O 成为瓶颈
- 无 FTS 倒排索引，关键词搜索的延迟随文件数线性增长

---

### 2.8 外部搜索集成

| | sage-wiki | llm_wiki |
|---|---|---|
| **Web 搜索** | 无内置集成（Agent 自行通过 MCP 调用外部工具） | 7 种：Tavily、SerpApi、SearXNG、Firecrawl、Brave、Bocha、Ollama Web Search |
| **桌面搜索** | 无 | AnyTXT 集成（`anytxt.search` 工具） |
| **搜索工具可被 Agent 调用** | 是（19 MCP 工具中的 `wiki_search`） | 是（12+ 内置工具中的 `wiki.search`、`source.search`、`web.search`） |

**llm_wiki 将外部搜索作为一等公民**，嵌入在 Deep Research 工作流和 Chat Agent 工具集中。执行 Deep Research 时，LLM 生成多查询 → 多提供商并发搜索 → 全内容提取 → 自动合成 wiki 页面 → 自动入库。

**sage-wiki 将外部搜索留给调用方 Agent**：Agent 通过 MCP `wiki_search` 搜索本地知识库，如需 Web 搜索则自行使用 `web_search` 类 MCP 工具，搜索结果可随后通过 `wiki_capture` 写入 wiki。

---

### 2.9 可配置性

**sage-wiki** —— 几乎所有参数可调：

```yaml
search:
  hybrid_weight_bm25: 0.7
  hybrid_weight_vector: 0.3
  hybrid_weight_graph: 0.2
  graph_relation_weights:
    contradicts: 1.1
    cites: 0.7
  pipeline: unified          # "unified" | "legacy"
  default_limit: 10
  query_expansion: true      # Q&A 路径
  rerank: true               # Q&A 路径
  rerank_min_coverage: 0.5
  chunk_size: 800
  chunk_overlap_tokens: 0
  graph_expansion: true
  graph_max_expand: 10
  graph_depth: 2
  context_max_tokens: 8000
  weight_direct_link: 3.0
  weight_source_overlap: 4.0
  weight_common_neighbor: 1.5
  weight_type_affinity: 1.0
  ann:
    enabled: false           # 大规模 vault 时启用
```

**llm_wiki** —— 关键参数可调：

- 上下文窗口：4K-1M 滑块
- 向量搜索：独立开关 + 端点 + API Key + 模型 + 维度
- 检索模式：Standard / Smart（迭代）/ Faithful（仅源材料）
- Agent 模式：Fast / Standard / Deep / LocalFirst
- Web 搜索：每提供商独立配置（最多 7 种）
- AnyTXT：端点 + 限制数
- 分词权重硬编码：`FILENAME_EXACT_BONUS: 200.0`, `PHRASE_IN_TITLE_BONUS: 50.0`, `TITLE_TOKEN_WEIGHT: 5.0`, `CONTENT_TOKEN_WEIGHT: 1.0`

---

## 三、总结

| 维度 | sage-wiki | llm_wiki |
|------|-----------|----------|
| **架构复杂度** | 高 —— 多阶段 LLM 增强管线 | 低 —— 经典检索 + 可选向量 |
| **检索质量** | ★★★★★ 块级 + LLM 扩展 + LLM 重排 + 图谱通道 | ★★★☆☆ 词频 + 可选向量 + wikilink 扩展 |
| **查询延迟** | 100ms-5s（取决于 LLM 阶段是否启用） | 50-500ms（全本地执行） |
| **单次查询成本** | LLM 阶段 ~$0.001-0.01（可完全关闭） | 零（无 LLM 调用） |
| **长文档检索** | ★★★★★ 块级索引精准定位段落 | ★★☆☆☆ 整页匹配，长文档内部不可检索 |
| **跨语言能力** | ★★★★★ 暴力向量搜索 + 显式跨语言设计 | ★★★☆☆ 依赖嵌入模型的多语言质量 |
| **图谱深度** | ★★★★★ 类型化边 + evidence + 2-hop + 实体解析 | ★★★☆☆ wikilink 1-hop 邻接 |
| **大规模扩展性** | ★★★★★ 无文件数硬限制 + HNSW ANN | ★★★☆☆ 10K 文件硬上限 |
| **部署复杂度** | 需编译建立索引（首次耗时） | 开箱即用，零等待 |
| **可调性** | 几乎所有参数可配置（~20 个调节点） | 关键参数可调（~10 个调节点） |

### 设计哲学差异

- **sage-wiki**：搜索是**知识工程管线**。每多一个 LLM 调用都在提升质量——查询扩展提升召回、重排序提升精度、三元组抽取提升图谱质量。代价是延迟和成本。适合作为 **Agent 的记忆后端**，追求高精度答案。

- **llm_wiki**：搜索是**桌面工具的快速查找**。零 LLM 依赖意味着零成本、零等待。图谱来自免费的 wikilink 解析。适合**个人日常使用**，快速定位信息。

### 互补使用

两个项目在设计上互补：llm_wiki 提供了 sage-wiki 所没有的 Deep Research 多源 Web 搜索和 7 种搜索引擎集成；sage-wiki 提供了 llm_wiki 所缺乏的大规模扩展性和高精度 LLM 增强检索。在理想的多工具协作场景中，可以 llm_wiki 作为前端摄入和桌面交互层，sage-wiki 作为后端大规模知识图谱和 Agent 记忆引擎。
