# 大文件处理机制

sage-wiki 可以摄取任意大小的源文件 —— 从单段笔记到 500 页的 PDF ——
并生成结构化摘要、概念图谱和文章。由于 LLM 的上下文窗口是有限的（通常
64K–1M tokens），sage-wiki **绝不会将整个超大文件一次性传给 LLM**。
相反，它采用分层分治策略：Token 估算 → 分块 → 分组 → 层级合成。
本文档逐一解释每一层的机制和相关的配置项。

## 整体流程

```
源文件（任意大小）
  → Token 估算（拉丁语系 ~4 字符/token，中日韩 ~1.5 tokens/字符）
  → 单块？  ───→ 一次 LLM 调用 → 摘要
  → 多块？  ───→ 逐块摘要（必要时分组）
                 → 层级树状合成（log₈ 收敛）
                 → 最终摘要
  → 文章写入：分段感知的源上下文（SplitThreshold + 术语匹配）
  → 搜索索引：块级 FTS5 + 向量索引（chunk_size、chunk_overlap）
```

## 第一层：Token 估算

在任何 LLM 调用之前，sage-wiki 会估算提取文本将消耗的 token 数量。
估算器（`internal/extract/extract.go:99`）采用混合文字系统的启发式算法：

| 文字系统 | 比例 |
|----------|------|
| 拉丁字母 / ASCII | ~4 字符/token |
| 中日韩文字（汉字、谚文、假名） | ~1.5 tokens/字符 |

一个 40 KB 的英文 Markdown 文件（~10K tokens）会超过默认的
`summary_max_tokens * 2 = 8000` 输入预算，从而触发分块。
同样大小的中文文档（~15K tokens）会更早触发分块，因为 CJK 字符
的 token 密度更高。

## 第二层：分块策略

当估算的 token 数超过 `summary_max_tokens * 2` 时，
`ChunkIfNeeded()`（`internal/extract/extract.go:164`）会对文本进行切分。
根据文件类型，使用两种算法：

### Markdown —— 按标题分块

文件在 `#`、`##`、`###` 标题边界处切分。每个块逐行累积，直到其累计
token 数接近上限，此时当前块被 flush，新块开始。每个块都携带其所属
章节标题作为元数据，LLM 将标题作为上下文一并处理。

### 纯文本 —— 按段落分块

当文档中没有 Markdown 标题时，在双换行边界（`\n\n`）处切分。遵循
相同的 token 预算规则：段落累积直至达到每块上限，然后开始新块。

## 第三层：块分组

当文件产生很多块时，每块的输出预算可能过低，无法生成高质量的摘要。
例如，`summary_max_tokens: 4000` 配合 100 个块，每块只能分配到
40 个输出 token —— 连一句连贯的话都不够。

`groupChunks()`（`internal/compiler/summarize.go:461`）强制执行一个下限：
**每组至少获得 1000 个输出 token**（`minChunkTokenBudget`）。
当 `maxTokens / len(chunks) < 1000` 时，相邻的块被合并成组。
对于 100 个块和 4000 token 预算，最多形成 `4000 / 1000 = 4` 个组，
每组覆盖约 25 个块。每个组在一次 LLM 调用中完成摘要。

## 第四层：层级合成

块组摘要完成后，可能仍有几十条中间摘要。相比将其全部塞入一个巨大的
prompt，sage-wiki 采用 **树状合成**（`synthesizeHierarchical`，
`internal/compiler/summarize.go:496`）：

```
第 0 层：64 条块摘要
第 1 层：每 ≤8 条合成为 1 条 → 8 条摘要
第 2 层：每 ≤8 条合成为 1 条 → 1 条最终摘要
```

每一步合成最多将 `synthesisGroupSize = 8` 条摘要合并为一条，使用一次
LLM 调用。收敛速度是对数级别的：1000 条块摘要仅需约 4 层。**任何一次
LLM 调用都只需处理不超过 8 条摘要加简短的指令 prompt**——远在任何模型
的上下文窗口限制之内。

### 批处理模式的限制

在批处理 API 模式（`compiler.mode: batch`）下，多块源文件会被**跳过**。
批处理 API 异步返回结果，无法支持层级合成所需的顺序依赖链。这些源文件
会被记录日志，应在 `standard` 模式下编译。

## 第五层：分段感知的文章写入

在 Pass 3（文章写入）阶段，`buildSourceContext()`（`internal/compiler/write.go:961`）
重新读取源文件，为 LLM 组装上下文。使用与写入目标匹配的不同策略——
只选择每个源文件中最相关的部分：

1. **小文档（< `split_threshold` 字符，默认 15000）：** 包含全部内容，
   上限 4000 字符，超出部分标记 `[...truncated...]`。

2. **含 Markdown 标题的大文档：** 通过 `SplitByHeadings()` 按标题边界
   拆分文档。只有标题或正文包含概念名称（或其别名）的章节才会被包含。
   每个匹配章节上限 4000 字符。相比包含整个文件，通常能将上下文大小
   减少 3–4 倍。

3. **不含标题的大文档：** 包含全部内容，但截断至 4000 字符。

如果没有章节匹配目标概念，则用第一节作为兜底，确保 LLM 始终有部分
基础材料可用。

## 第六层：搜索索引分块

对于搜索，源文件在索引时以可配置的 token 粒度进行分块（`search.chunk_size`，
默认 800 tokens）。每个块获得独立的 FTS5 条目和向量嵌入。可选的重叠
（`search.chunk_overlap_tokens`，默认 0）会将前一个块的尾部文本作为
下一个块的前缀，使跨边界的事实能同时出现在相邻块中。

这意味着一个 500 KB 的源文件（在 800 tokens/块下约 25 个索引块）可以
在段落级别匹配查询，而不仅仅是文档级别——在不扩大每条结果负载的情况下
提升召回率。

### 全文档嵌入：均值池化

当需要全文档嵌入向量时（例如文档级向量索引通道），超过 5000 个 rune
的文本被切分为多个块，每个块分别嵌入，最终将得到的向量**均值池化**
为单个固定维度的向量（`internal/embed/embed.go:232`）。

## 第七层：查询上下文的 Token 预算

在问答过程中，组装的上下文受 `search.context_max_tokens`（默认 8000）
限制。文章按相关性分数贪婪添加，直到预算耗尽。每篇文章额外有
16000 字符的上限。通过 MCP `wiki_search` 工具返回的搜索结果每条上限
为 `search.result_max_chars`（默认 2000 runes），附带指向 `wiki_read`
获取全文的提示。

## 第八层：重排序截断

当 `search.rerank` 启用时（默认：开启），发送给 LLM 进行相关性评分的
每条候选块被截断至 400 tokens。总 prompt 预算为 8000 tokens，每次查询
最多考虑 15 个候选。这确保即使初始检索返回大量结果，重排序也能快速
完成且不超出上下文限制。

## 第九层：Zip 压缩包安全防护

Office 文档（`.docx`、`.xlsx`、`.pptx`、`.epub`）本质上是 zip 压缩包。
sage-wiki 强制执行解压上限以防止 zip 炸弹攻击：

| 限制 | 值 |
|------|-----|
| 单条目解压后 | 50 MB |
| 单个压缩包总计 | 200 MB |

通过两层机制执行：快速的声明大小预检，以及流式有界读取器——溢出时报错，
绝不静默截断。

## 配置参考

```yaml
compiler:
  summary_max_tokens: 4000      # 每次摘要调用的输出 token 数；输入预算 = 2x
  article_max_tokens: 4000      # 每次文章写入调用的输出 token 数
  extract_max_tokens: 8192      # 每次概念提取调用的输出 token 数
  split_threshold: 15000        # 字符数；超过此值在写入阶段启用分段感知
  split_strategy: headings      # "headings"（默认）

search:
  chunk_size: 800               # 每个搜索索引块的 token 数（100–5000）
  chunk_overlap_tokens: 0       # 相邻块之间的重叠 token 数（≤ chunk_size/2）
  context_max_tokens: 8000      # 问答上下文组装的 token 预算
  result_max_chars: 2000        # 每条 wiki_search 结果的最大 rune 数
```

### 内部常量

| 常量 | 位置 | 值 | 用途 |
|------|------|-----|------|
| `minChunkTokenBudget` | `summarize.go:379` | 1000 | 每组块的最低输出 token 数 |
| `synthesisGroupSize` | `summarize.go:383` | 8 | 每次合成调用最多处理的摘要数 |
| `maxEmbedChars` | `embed.go:200` | 5000 | 每个嵌入块的最大 rune 数 |
| `DefaultHeadRunes` | `readhead.go:16` | 500 | 类型检测读取的 rune 数 |
| `maxChunkTokens` | `rerank.go:35` | 400 | 每个重排序候选项的 token 数 |
| `maxZipEntryBytes` | `ziplimits.go:27` | 50 MB | 单条目解压上限 |
| `maxZipTotalBytes` | `ziplimits.go:28` | 200 MB | 压缩包总解压上限 |

## 追踪示例：一个 500 KB Markdown 文件的完整处理流程

1. **类型检测** —— 读取前 500 个 rune 进行内容类型识别。
2. **提取** —— 整个文件读入内存；解析 YAML frontmatter。
3. **Token 估算** —— ~125K 拉丁字符 → ~31K tokens，超过
   `4000*2=8000` 输入预算 → 触发分块。
4. **分块** —— 在 `##` 标题处切分，产生约 25 个块。
5. **分组** —— `4000/25 = 160` tokens/块，低于 1000 下限 →
   块合并为 `4000/1000 = 4` 组，每组约 6 个块。
6. **按组摘要** —— 4 次 LLM 调用，每次 1000 token 输出预算。
7. **层级合成** —— 4 条摘要 → 1 条最终摘要，仅需一次合成调用
   （4 ≤ synthesisGroupSize=8）。
8. **文章写入** —— 重新读取源文件；500 KB > 15000 字符分块阈值
   → 按标题拆分；仅包含匹配概念名称/别名的章节，每节上限 4000 字符。
9. **索引** —— 摘要文本 < 5000 runes → 单向量嵌入（无需均值池化）。
10. **搜索** —— 25 个块以 800 tokens/块索引；查询在段落粒度匹配。

**摘要阶段总 LLM 调用次数：5 次**（4 次分组 + 1 次合成）。
如果不分块，该文件仅输入就需要约 31K tokens 的上下文窗口——输出
长度问题仍未解决。
