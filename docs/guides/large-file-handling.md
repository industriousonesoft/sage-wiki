# Handling Large Source Files

sage-wiki ingests source files of any size — from one-paragraph notes to
500-page PDFs — and produces structured summaries, concept graphs, and
articles. LLM context windows are finite (typically 64K–1M tokens), so
sage-wiki **never passes an entire oversized file to the LLM in one call**.
Instead it applies a layered divide-and-conquer strategy: token estimation
→ chunking → grouping → hierarchical synthesis. This guide explains each
layer and every configuration knob that controls it.

## How it works

```
Source File (any size)
  → Token estimation (Latin ~4 chars/tok, CJK ~1.5 chars/tok)
  → Single chunk?  ───→ One LLM call → summary
  → Multi-chunk?    ───→ Per-chunk summaries (grouped if needed)
                         → Hierarchical tree synthesis (log₈ convergence)
                         → Final summary
  → Write pass: section-aware source context (SplitThreshold + term matching)
  → Search index: chunk-level FTS5 + vectors (chunk_size, chunk_overlap)
```

## Layer 1: Token Estimation

Before any LLM call, sage-wiki estimates how many tokens the extracted text
will consume. The estimator (`internal/extract/extract.go:99`) uses a
mixed-script heuristic:

| Script | Ratio |
|--------|-------|
| Latin / ASCII | ~4 characters per token |
| CJK (Han, Hangul, Kana) | ~1.5 tokens per character |

A 40 KB English markdown file (~10K tokens) exceeds the default
`summary_max_tokens * 2 = 8000` input budget and triggers chunking. A 40 KB
Chinese document (~15K tokens) would trigger it earlier because CJK
characters are token-heavier.

## Layer 2: Chunking Strategies

When the estimated tokens exceed `summary_max_tokens * 2`, `ChunkIfNeeded()`
(`internal/extract/extract.go:164`) splits the text. Two algorithms are
used depending on file type:

### Markdown — split by headings

The file is split at `#`, `##`, `###` heading boundaries. Each chunk
accumulates lines until its running token count nears the limit, at which
point it is flushed and a new chunk starts. Every chunk carries its section
heading as metadata, so the LLM sees the heading as context.

### Plain text — split by paragraphs

When no markdown headings are present, the text is split on double-newline
boundaries (`\n\n`). The same token-budget discipline applies: paragraphs
accumulate until the per-chunk limit is reached, then a new chunk starts.

## Layer 3: Chunk Grouping

When a file produces many chunks, the per-chunk output budget can fall too
low for a quality summary. For example, with `summary_max_tokens: 4000` and
100 chunks, each chunk would get only 40 output tokens — not enough for a
coherent sentence.

`groupChunks()` (`internal/compiler/summarize.go:461`) enforces a floor:
**each group must receive at least 1000 output tokens** (`minChunkTokenBudget`).
When `maxTokens / len(chunks) < 1000`, adjacent chunks are merged into
groups. With 100 chunks and a 4000-token budget, at most
`4000 / 1000 = 4` groups are formed, each covering ~25 chunks. Every group
is summarized in one LLM call with a 1000-token output budget.

## Layer 4: Hierarchical Synthesis

After chunk-group summarization, there may still be dozens of intermediate
summaries. Rather than stuffing all of them into one enormous prompt,
sage-wiki uses **tree-structured synthesis** (`synthesizeHierarchical`,
`internal/compiler/summarize.go:496`):

```
Tier 0:  64 chunk summaries
Tier 1:  synthesize groups of ≤8  →  8 summaries
Tier 2:  synthesize groups of ≤8  →  1 final summary
```

Each synthesis step merges at most `synthesisGroupSize = 8` summaries into
one, using one LLM call. The convergence is logarithmic: 1000 chunk
summaries require only ~4 tiers. **No single LLM call ever sees more than
8 summaries plus a short instruction prompt** — well within any model's
context window.

### Batch mode caveat

In batch API mode (`compiler.mode: batch`), multi-chunk sources are skipped
entirely. The batch API delivers results asynchronously and cannot support
the sequential dependency chain that hierarchical synthesis requires. These
sources are logged and should be compiled in `standard` mode.

## Layer 5: Section-Aware Article Writing

During Pass 3 (article writing), `buildSourceContext()`
(`internal/compiler/write.go:961`) reads source files again to assemble
context for the LLM. It uses a different strategy suited to the write
pass's goal — selecting only the most relevant portions of each source:

1. **Small documents (< `split_threshold` chars, default 15000):** the
   entire content is included, capped at 4000 characters with a
   `[...truncated...]` marker.

2. **Large documents with markdown headings:** the document is split at
   heading boundaries via `SplitByHeadings()`. Only sections whose heading
   or body text contains the concept name (or its aliases) are included.
   Each matching section is capped at 4000 characters. This typically
   reduces context size by 3–4× compared to including the whole file.

3. **Large documents without headings:** the full content is included but
   truncated to 4000 characters.

If no sections match the concept, the first section is used as a fallback
so the LLM always has some grounding material.

## Layer 6: Search Index Chunking

For search, sources are chunked at indexing time with a configurable token
granularity (`search.chunk_size`, default 800 tokens). Each chunk receives
its own FTS5 entry and vector embedding. Optional overlap
(`search.chunk_overlap_tokens`, default 0) prefixes each chunk with the
tail of its predecessor, so boundary-straddling facts appear in both chunks.

This means a 500 KB source file (~25 indexed chunks at 800 tokens each) can
match a query at the passage level, not just the document level — improving
recall without inflating the per-result payload.

### Whole-document embedding: mean pooling

When a whole-document embedding is needed (e.g., for the document-level
vector index leg), text longer than 5000 runes is split into chunks, each
embedded separately, and the resulting vectors are **mean-pooled** into a
single fixed-dimension vector (`internal/embed/embed.go:232`).

## Layer 7: Query-Context Token Budgeting

During Q&A, assembled context is bounded by `search.context_max_tokens`
(default 8000). Articles are added greedily by relevance score until the
budget is exhausted. Each individual article is additionally capped at
16000 characters. Search results returned via the MCP `wiki_search` tool
are capped at `search.result_max_chars` (default 2000 runes) per result,
with a pointer to `wiki_read` for the full text.

## Layer 8: Re-ranking Truncation

When `search.rerank` is enabled (default: true), each candidate chunk sent
to the LLM for relevance scoring is truncated to 400 tokens. The total
prompt budget is 8000 tokens, and at most 15 candidates are considered per
query. This keeps re-ranking fast and within context limits even when the
initial retrieval returns many hits.

## Layer 9: Zip Archive Protection

Office documents (`.docx`, `.xlsx`, `.pptx`, `.epub`) are zip archives.
sage-wiki enforces hard decompression limits to prevent zip bombs:

| Limit | Value |
|-------|-------|
| Per-entry decompressed | 50 MB |
| Aggregate per archive | 200 MB |

These are enforced at two layers: a fast declared-size pre-check, and a
streaming bounded reader that errors on overflow — never silent truncation.

## Configuration Reference

```yaml
compiler:
  summary_max_tokens: 4000      # output tokens per summarize call; input budget = 2x
  article_max_tokens: 4000      # output tokens per article write call
  extract_max_tokens: 8192      # output tokens per concept extraction call
  split_threshold: 15000        # chars; enable section-aware source context above this
  split_strategy: headings      # "headings" (default)

search:
  chunk_size: 800               # tokens per search-index chunk (100–5000)
  chunk_overlap_tokens: 0       # tokens of overlap between adjacent chunks (≤ chunk_size/2)
  context_max_tokens: 8000      # token budget for Q&A context assembly
  result_max_chars: 2000        # max runes per wiki_search result
```

### Internal constants

| Constant | File | Value | Purpose |
|----------|------|-------|---------|
| `minChunkTokenBudget` | `summarize.go:379` | 1000 | Min output tokens per chunk group |
| `synthesisGroupSize` | `summarize.go:383` | 8 | Max summaries per synthesis call |
| `maxEmbedChars` | `embed.go:200` | 5000 | Max runes per embedding chunk |
| `DefaultHeadRunes` | `readhead.go:16` | 500 | Runes read for type detection |
| `maxChunkTokens` | `rerank.go:35` | 400 | Tokens per rerank candidate |
| `maxZipEntryBytes` | `ziplimits.go:27` | 50 MB | Per-entry decompression cap |
| `maxZipTotalBytes` | `ziplimits.go:28` | 200 MB | Aggregate decompression cap |

## Trace: How a 500 KB Markdown File Flows

1. **Type detection** — first 500 runes read for content-type signals.
2. **Extraction** — whole file read into memory; YAML frontmatter parsed.
3. **Token estimation** — ~125K chars Latin → ~31K tokens, exceeds
   `4000*2=8000` input budget → chunking triggered.
4. **Chunking** — split on `##` headings, ~25 chunks produced.
5. **Grouping** — `4000/25 = 160` tokens/chunk, below 1000 floor → chunks
   merged into `4000/1000 = 4` groups of ~6 chunks each.
6. **Per-group summarization** — 4 LLM calls, each with 1000-token output
   budget.
7. **Hierarchical synthesis** — 4 summaries → 1 final summary in one
   synthesis call (4 ≤ synthesisGroupSize of 8).
8. **Write pass** — source re-read; 500 KB > 15000 chars split threshold
   → split by headings; only sections matching the concept's name/aliases
   are included, each capped at 4000 chars.
9. **Indexing** — summary text < 5000 runes → single-vector embedding (no
   mean-pooling needed).
10. **Search** — 25 chunks indexed at 800 tokens each; query matches at
    passage granularity.

**Total LLM calls for summarization: 5** (4 group + 1 synthesis). Without
chunking, this file would require a context window of ~31K tokens just for
input — and would still need a separate strategy for output length.
