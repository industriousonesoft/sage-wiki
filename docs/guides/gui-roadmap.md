# sage-wiki 团队 GUI 路线图

## 背景

sage-wiki 作为团队知识库，当前的 CLI + YAML 配置模式对非开发者成员门槛较高：

- 初始化依赖终端命令（`sage-wiki init` / `compile`）
- 配置需手工编辑 `config.yaml`（20+ 配置项）
- 编译过程是终端黑盒，团队成员无法感知状态
- 搜索 / Q&A / Trust 审核流程都需通过 CLI 或 MCP 完成
- 团队协作状态（谁编译了、谁审核了）不透明

目标：通过增强现有 Web UI，让非开发者成员通过浏览器完成全部日常操作，同时保持 MCP / CLI 用户不受影响。

## 设计原则

1. **Web UI 而非桌面应用** —— 团队部署模式下 wiki 运行在服务器上，成员通过浏览器访问；每人安装桌面客户端不符合团队共享模式
2. **复用现有架构** —— 全部基于现有 Preact + Tailwind + force-graph 技术栈和 `/v1` API + SSE 端点，不引入新语言或运行时
3. **保持单二进制部署** —— 通过 `go:embed` 打包，`sage-wiki serve --ui` 一条命令启动
4. **MCP / CLI 不受影响** —— GUI 只是现有 API 的另一个消费者，不改变后端契约

## 当前门槛 → GUI 解决方案映射

### 门槛 1：初始化全靠 CLI

**现状**：`sage-wiki init my-wiki` → 手工编辑 `config.yaml` 填 API Key → `sage-wiki compile`

**解决**：Setup Wizard（首次启动引导）

`sage-wiki serve --ui` 启动时检测到项目未初始化（无 `config.yaml` / `wiki.db`），浏览器打开后进入引导流程：

```
Step 1: 选择 LLM 提供商（Anthropic / OpenAI / Gemini / Ollama / Custom）
        → 填入 API Key → 「测试连接」按钮验证
Step 2: 选择存储后端（SQLite 默认 / PostgreSQL + pgvector）
        → 填入连接信息 → 测试连接
Step 3: 选择源文件目录（浏览选择或拖拽）→ 预览将处理的文件列表
Step 4: （可选）选择 Contribution Pack（8 个领域包）
Step 5: 确认摘要 → 一键开始首次编译 → 进度条实时反馈
```

后端实现：Wizard 的每一步通过 `/v1` API 生成/写入 `config.yaml`，或新增 `POST /v1/setup` 端点一次性完成初始化。

### 门槛 2：编译状态不可见

**现状**：`sage-wiki compile` 在终端运行，团队成员不知道进度、失败项、覆盖率。

**解决**：Dashboard 首页

利用已有异步 Job API（`POST /v1/jobs/compile` 返回 `202` + `job_id` 轮询）：

- **编译进度**：进度条 + 当前处理文件 + 预估剩余时间
- **编译历史**：最近 N 次编译（触发者、时间、源文件数、耗时、成本）
- **源文件覆盖率**：饼图展示 Tier 0/1/2/3 分布 + 未编译/编译中/失败文件列表
- **失败项**：编译失败的源文件 + 错误信息 + 重试按钮
- **成本统计**：本月 token 使用量 + 估算费用（按编译任务分组）

### 门槛 3：配置只能手写 YAML

**现状**：`config.yaml` 注释虽全，但非开发者不敢改。

**解决**：Settings 面板（分类 Tab）

| Tab | 内容 |
|-----|------|
| **LLM** | 提供商、模型、API Key（密码框）、temperature、超时、Batch API 开关 |
| **搜索** | 三通道权重滑块（BM25/Vector/Graph）、上下文预算、重排序/查询扩展开关、ANN 开关 |
| **本体** | 关系类型管理（增删改）、实体解析阈值、社区检测开关、三元组抽取开关 |
| **编译** | 默认 Tier、自动升级信号、chunk 大小、代码解析器开关 |
| **团队** | 共享模式（Git/Server/Hub）、认证 Token 管理、访问控制 |
| **信任** | Grounding 阈值、共识所需人数、自动 promote 规则 |

实现：每个字段旁显示从 `config.yaml` 注释提取的说明文字；表单提交 → 验证 → 写入 `config.yaml`（或通过 API 更新）。非法值即时红框提示，提交前 dry-run 校验。

### 门槛 4：搜索和 Q&A 缺乏交互

**现状**：`sage-wiki search` / `sage-wiki query` CLI 文本输出。

**解决**：Chat + Search 统一面板（llm_wiki 风格）

```
┌─────────────────────────────────────────────────────────┐
│ 搜索框（自动切换到 Search / Chat 模式）                    │
├──────────────┬──────────────────────────────────────────┤
│ 结果列表      │ 预览窗格                                   │
│ ├─ 标题+摘要  │  ├─ 全文 Markdown 渲染（高亮命中词）         │
│ ├─ 得分徽章   │  ├─ 来源标签（文件/概念/输出）              │
│ ├─ 通道图标   │  ├─ 关联图谱面板（相关节点缩略图）           │
│ └─ 页码      │  └─ 「在图中查看」按钮                      │
├──────────────┴──────────────────────────────────────────┤
│ Chat 模式：                                               │
│  Q: 问题                                                   │
│  A: 流式回答 + 引用标注 [1] [2] + 来源文章列表              │
│     [存入 wiki] [重新生成] 按钮                            │
└─────────────────────────────────────────────────────────┘
```

- Search 模式：调用 `POST /v1/search`，结果显示通道来源徽章（BM25/Vector/Graph）
- Chat 模式：调用 SSE 流式 Q&A 端点，回答带引用编号，点击引用跳转到来源文章
- 「存入 wiki」按钮：调用 `wiki_write_output` 对应的 API，将优质回答归档（触发 Trust 审核流程）

#### Chat 模式是否需要内嵌 Agent？——不需要

Chat 模式不是内嵌 Agent，而是复用 sage-wiki 已有的**单次 Q&A 管线**（`wiki_query`）。这是与 llm_wiki 的关键架构分叉点：

| | llm_wiki Chat | sage-wiki Chat（本方案） |
|---|---|---|
| **实现** | Rust 内嵌 Agent 运行时：LLM 决定调用哪个工具 → 执行 → 反馈 → 再决定（最多 8 轮迭代） | 确定性管线：`wiki_query` 一次调用完成全部检索与合成 |
| **为什么这样设计** | 其搜索只有词频 + 可选向量，检索质量弱，需要 LLM 反复调 `wiki.search` / `wiki.read_page` / `web.search` 来补偿 | 检索管线已闭环：查询扩展 → 三通道 RRF → 图谱扩展 → LLM 重排，单发即中，无需 LLM 自行探索 |
| **延迟** | 不可预测（1-8 轮 × 每轮 LLM 调用） | 可预测（检索 ~5ms + 单次 LLM 调用） |
| **成本** | 每轮工具调用都消耗 token | 单次合成，token 用量有界 |
| **权限模型** | 需要完整的能力权限系统（shell、文件写入、网络） | 无工具循环，无需权限模型 |

Chat 模式的本质是给现有管线加一个**对话式前端**：

| 组件 | 实现方式 |
|---|---|
| 流式输出 | 已有基础（现有 Web UI 的 QAPanel 即为流式 Q&A） |
| 多轮对话状态 | 前端维护会话历史（`wiki_query` 支持 history 参数） |
| 引用标注跳转 | 渲染时解析引用编号 → 链接到文章视图 |
| 「存入 wiki」按钮 | 调用 `wiki_write_output` 对应 API（触发 Trust 审核） |
| 会话持久化 | 浏览器 localStorage 或服务端轻量存储 |

**为什么不在 GUI 里内嵌 Agent**：

1. **检索管线已经闭环** —— 查询扩展 + 三通道 + 重排 + 图谱扩展已把「找资料」完成，不需要 LLM 自己摸索
2. **Agent 能力已经外置** —— Claude Code 等外部 Agent 通过 MCP 的 19 个工具就是 sage-wiki 的 Agent 层；GUI 里再造内嵌 Agent 属于重复建设
3. **成本和可预测性** —— 单次管线延迟固定、成本有界；Agent 循环两者均不可预测

**如果未来需要 Agent 行为**（例如「把所有关于 X 的文章合成一个对比页」），正确路径不是内嵌新 Agent，而是：

- 方案 A：GUI 加「Agent 模式」开关 → 后端复用已有 19 个 MCP 工具定义，做轻量工具循环（LLM 选择 search/read/write 组合，上限 N 轮）
- 方案 B（推荐）：引导用户用 Claude Code + sage-wiki MCP —— 这条链路今天已完整存在，无需新开发

### 门槛 5：图谱只能看图

**现状**：GraphView 是只读力导向图。

**解决**：交互式图谱

- **点击节点** → 侧滑面板：文章摘要、关联关系列表（带 relation type + evidence + confidence）、provenance 溯源、「阅读全文」跳转
- **悬停边** → Tooltip 显示 relation type、confidence、source_doc
- **实体解析提案** → 图上特殊标记（虚线框），侧滑面板中一键 approve / reject
- **社区着色切换** → 按类型 / 按社区 / 按 Tier 着色
- **节点搜索** → 输入关键词，高亮 + 定位匹配节点
- **半径展开** → 双击节点展开 2-hop 邻居

### 门槛 6：Trust / Review 流程只有 CLI

**现状**：`sage-wiki verify` / `sage-wiki outputs promote` 等命令。

**解决**：Review 面板

```
待审核列表（可筛选：输出 / 实体解析 / 冲突标记）
┌────────────────────────────────────────────┐
│ ⚠ 输出 #42 「XX 架构的优缺点分析」           │
│   来源查询: "XX 架构适合我们吗？"            │
│   Grounding 状态: ✅ 3/3 已验证              │
│   冲突标记: 与 #31 存在矛盾                  │
│   [Promote] [Demote] [Reject] [查看全文]    │
├────────────────────────────────────────────┤
│ ⚠ 实体解析提案: "K8s" → "Kubernetes"       │
│   置信度: 0.92                              │
│   [Approve] [Reject] [稍后处理]             │
└────────────────────────────────────────────┘
```

- 按类型筛选（quarantined outputs / entity proposals / contradictions）
- 批量操作（全选 approve / reject）
- 操作历史（谁在何时做了什么决定）

### 门槛 7：团队管理不透明

**现状**：Hub 命令管理，成员无法看到协作状态。

**解决**：Team 面板

- **成员列表**：贡献统计（编译数、审核数、提交文章数）
- **活动时间线**：最近变更（谁编译了什么源文件、谁审核了什么、谁修改了配置）
- **Token 使用统计**：按成员 / 按任务的 token 消耗和费用
- **项目状态**：总文章数、总概念数、覆盖率、健康评分（复用 `sage-wiki status` / `doctor` 输出）

## 实施优先级

| 优先级 | 功能 | 解决的痛点 | 预估工作量 | 依赖的 API |
|--------|------|-----------|-----------|-----------|
| P0 | Setup Wizard | 新团队无法自主搭建 | 3-5 天 | 新增 `POST /v1/setup` |
| P0 | Settings 面板 | 不敢改配置，用默认值跑不起来 | 2-3 天 | 新增 `GET/PUT /v1/config` |
| P0 | Dashboard + 编译进度 | 编译黑盒，不知道 wiki 状态 | 3-5 天 | 已有 `POST /v1/jobs/compile` + 轮询 |
| P1 | Chat + Search 统一面板 | 日常使用仍需 CLI | 5-7 天 | 已有 `POST /v1/search` + SSE Q&A |
| P1 | Review 面板 | Trust 流程形同虚设 | 2-3 天 | 已有 outputs/verify 相关 API |
| P2 | 交互式图谱增强 | 图谱只能看不能操作 | 3-5 天 | 已有 graph 查询 API |
| P2 | Team 面板 | 团队协作透明 | 2-3 天 | 已有 status/metrics API |

## 新增 API 需求

现有 `/v1` API 已覆盖大部分功能，需新增：

```
POST /v1/setup              # Setup Wizard：一次性初始化（生成 config.yaml + 启动首次编译）
GET  /v1/config             # 读取当前配置（脱敏：API Key 打码）
PUT  /v1/config             # 更新配置（dry-run 校验 + 写入）
GET  /v1/dashboard          # Dashboard 聚合数据（编译历史、覆盖率、成本统计）
GET  /v1/activity           # 活动时间线（谁做了什么）
```

以上端点均需 Bearer 认证（已有 `SAGE_WIKI_TOKEN` 机制），与现有 API 错误模型一致。

## 技术实现要点

### 前端

- **三栏布局**：`react-resizable-panels`（或 Preact 等价物），左侧导航 + 中间主内容 + 右侧预览/详情
- **状态管理**：Preact Context 或轻量 store（`zustand` 约 1KB，与 Preact 兼容）
- **图表渲染**：Mermaid（聊天中的图表）+ force-graph（知识图谱，已有）
- **流式输出**：SSE 消费已有端点，逐 token 渲染
- **暗色模式**：Tailwind `dark:` 类 + 系统偏好检测（已有基础）
- **打包**：保持 `go:embed` 单二进制（当前 ~1.2MB，新增组件后预计 ~2-3MB，可接受）

### 后端

- Wizard / Config / Dashboard 端点遵循现有 `internal/api/` 模式（结构化错误码、幂等性）
- 编译进度复用异步 Job 机制（`job_id` 轮询），无需新的事件系统
- Activity 时间线写入已有 log 机制，查询时聚合
- 全部端点纳入 OpenAPI 3.1 契约（`api/openapi.yaml`），与 MCP 工具集 drift-check 保持一致

### 安全

- 所有写操作（config 更新、review 决定、setup）需 Bearer 认证
- API Key 仅在写入时接收，读取时脱敏显示（`sk-...abc`）
- Review 操作记录操作者身份（token 关联成员）
- Setup Wizard 仅在项目未初始化时可用，初始化后端点返回 409

## 不做的事

- ❌ 不做 Tauri 桌面应用（团队部署在服务器，桌面客户端无意义；且引入 Rust + Node 工具链，破坏单二进制构建）
- ❌ 不做多用户账号系统（超出当前 scope，先以 token 级认证 + 操作日志满足团队基本需求）
- ❌ 不做实时协作编辑（wiki 文章是 LLM 生成的，人工编辑是次要场景）
- ❌ 不复制 llm_wiki 的 Desktop GUI 特性清单（Chrome Extension、系统托盘等属于桌面场景）

## 验收标准

1. 一个新团队成员，从拿到服务器地址到完成首次 wiki 编译，全程浏览器操作，耗时 < 15 分钟
2. 团队成员日常查询、浏览、审核操作无需打开终端
3. 编译进度、失败原因、覆盖率在 Dashboard 一目了然
4. MCP / CLI 用户行为不受任何影响（API 契约向后兼容）
5. 单二进制部署保持不变（`go build -tags webui` 产物包含全部 GUI 功能）
