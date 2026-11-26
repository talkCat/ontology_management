# 私有化定制 · 设计问答 01

## 把「问数能力」以能力级 MCP 暴露给外部智能体

> 状态：草案 v0.1 ｜ 2026-11-26
> 性质：设计文档（只定契约与边界，不含代码改动）
> 读者：平台研发、私有化交付、外部智能体接入方
> 关联：《系统说明书 09-智能问答》《系统说明书 10-MCP服务与外部集成》《ADR 0018 对象检索走 PostgreSQL 索引》《ADR 0021 退役对象物化》

---

## 一、背景

### 1.1 平台内部已经有一条完整的问数链路

平台自己问答的效果，不由某一个工具决定，而是由**一整条链路**共同决定：

```
用户问题
  → 组装历史上下文（prepareHistory：最近 N 轮原样 + 更早的摘要）
  → buildInstructions：系统提示词 + 本轮可用工具说明 + 概念清单(schemaBrief)
  → ToolLoopAgent 工具循环（上限 100 步）
  → 工具执行：search_schema / get_object_type / list_metrics / run_query / run_sql …
  → 按 toolCallId 回收证据(evidence)
  → 结论(answer)
  → 审计(REASONING_RUN) + 落对话历史(saveTurn)
```

各环节的代码位置：

| 环节 | 位置 |
| --- | --- |
| 步数兜底上限 100 | `src/lib/reasoning/agent.ts:26`、`:106` |
| 工具循环 | `src/lib/reasoning/agent.ts:149`（`stopWhen: stepCountIs(maxSteps)`） |
| 概念清单 schemaBrief | `src/lib/reasoning/prompt.ts:87-149` |
| instructions 组装（提示词 + 工具说明 + 概念清单） | `src/lib/reasoning/prompt.ts:157-172` |
| 工具开关（全局默认 + 本体覆盖） | `src/lib/reasoning/tool-policy.ts:72-77`、`:85-94` |
| 单步结果留存上限 60k 字符 | `src/lib/reasoning/agent.ts:36-42` |
| 中止时保留已完成步骤 | `src/lib/reasoning/agent.ts:292-299` |
| 审计 | `src/lib/reasoning/run-registry.ts:294-299` |
| 落对话历史 | `src/lib/reasoning/run-registry.ts:306-321` |
| 运行载体（起一轮 + SSE 订阅） | `src/app/api/reasoning/runs/route.ts:14-22`、`src/app/api/reasoning/runs/[runId]/events/route.ts:14-27` |

结论：**这条链路只在 UI 内部闭环**。UI 之外，没有任何入口能一次拿到「答案 + 证据」。

### 1.2 对外只开了「工具级」出口

本体数据 MCP（`/api/mcp`、`/api/mcp/<本体 id>`）把这条链路拆成了零件。当前代码里定义了 **13 个**工具：

- 12 个推理工具：`search_schema`、`get_object_type`、`list_concept_groups`、`list_interfaces`、`traverse_object_types`、`get_table_ddl`、`run_sql`、`run_query`、`query_object_instance`、`query_instance_subgraph`、`list_actions`、`list_metrics`（`src/lib/reasoning/tools/registry.ts:107-264`）
- 1 个仅 MCP 的工具：`list_ontologies`（`src/lib/reasoning/mcp.ts:97-109`）

出厂默认关掉 4 个（`list_actions`、`run_query`、`query_object_instance`、`query_instance_subgraph`，见 `src/lib/reasoning/tool-policy.ts:72-77`），所以**默认暴露 9 个**；某个本体打开 `run_query` 后是 10 个。这个数字随策略变化，文档里不要写死。

### 1.3 接入方实测到的三条偏差

某私有化项目把平台 MCP 接进自己的智能体后，同一批问题出现系统性偏差，归因如下：

| # | 现象 | 根因 |
| --- | --- | --- |
| 1 | 复杂问题被截断 | 接入方 Agent 的步数预算通常只有 4～12 步；平台的 100 步预算只属于平台自己 |
| 2 | 首次调用即挂起，要求人工确认 | `tools/list` 不返回 `annotations`（`src/lib/reasoning/mcp-endpoint.ts:86-91` 只输出 name/title/description/inputSchema），客户端只能把工具一律当作「未知副作用」 |
| 3 | 零工具调用，或直接猜表名写 SQL | 接入方拿不到 `schemaBrief`，不知道本体里有哪些对象类型、指标口径、绑了哪张表 |

### 1.4 问题定性：工具级对接 ≠ 能力级对接

把 13 个零件交出去，等于要求接入方**用自己的模型、自己的提示词、自己的步数预算，把平台的推理链路重写一遍**。重写出来的东西与平台原链路不可能等价——这不是提示词调优能弥补的差距，而是**出口形态**的问题。

因此本次定制的核心不是「再加几个工具」，而是：**把「问数」本身作为一个能力出口**。

### 1.5 目标

| 编号 | 目标 |
| --- | --- |
| G1 | 同源：外部调用与 UI 问答走**同一条** `runReasoning` 链路，结果可对齐 |
| G2 | 极简：接入方只需「问」和「取」，不理解平台任何工具 |
| G3 | 不越权：不新增写入口，不绕过工具开关、权限与审计 |
| G4 | 可交付：内网可跑、可审计、可限流、可一键回滚 |

### 1.6 非目标

- 不做 MCP 的写 / 动作 / 发布能力（平台的写入口仍然只有平台内动作）；
- 不替代 UI 问答，不做新的提示词体系；
- 不改推理循环、工具语义、SQL 两条路径的既有分工；
- 不做多租户计费与账单。

### 1.7 设计约束

| 约束 | 说明 |
| --- | --- |
| 只读已发布版本 | 与 UI 一致，草稿不对外 |
| 必须复用推理层 | 不允许为 MCP 另写一套 prompt / 工具编排，否则「同源」失效 |
| 协议不越界 | MCP `2025-06-18`，Streamable HTTP **JSON 响应**形态；本实现无服务端推送（`src/app/api/mcp/route.ts:24-27` 的 GET 直接 405） |
| 私有化 | 模型走内网 OpenAI 兼容网关（`LLM_BASE_URL`），不依赖外网 |

---

## 二、设计问答

### Q1：为什么不能让接入方自己拼这 13 个工具？

可以拼，但拼出来的不是同一个东西。三处硬差距：

1. **概念清单缺失**：`schemaBrief` 是平台把本体压成一段话塞进 instructions（`src/lib/reasoning/prompt.ts:157-172`），接入方只能靠 `search_schema` 一次次试探；
2. **步数预算不对等**：平台 100 步（`agent.ts:26`），多数 Agent 框架默认 4～12 步；
3. **纠错回路不一致**：平台把工具错误放进 `isError` 让模型自我纠正（`mcp-endpoint.ts:138-147`），但接入方的模型未必会看。

结论：问数质量是平台的产品能力，应当由平台闭环；接入方只做「选能力 → 传问题 → 转述结果」。

### Q2：为什么在 MCP 上做，而不是提供一个 HTTP 接口？

因为接入方是 Agent，不是脚本。Agent 需要一个**可被发现、可被授权、可被审计**的能力清单，这正是 MCP 的职责。另开一套 REST 会让接入方维护两套鉴权、两套令牌、两套审计，交付成本更高。

### Q3：能力级出口放在哪个端点？

**新增一个独立的本体级端点，工具集只有两个：**

```
POST /api/mcp/ask/<本体 id>      ← 能力级端点（本次新增）
POST /api/mcp/<本体 id>          ← 细粒度端点（保持不变）
POST /api/mcp                    ← 平台级端点（保持不变）
```

理由：

1. **互斥优于文案**。如果把 `ask_ontology` 和 13 个细粒度工具放在同一个 `tools/list` 里，接入方的模型一定会在两者之间摇摆（这正是原需求文档自己也担心的问题）。用端点隔离，是唯一可靠的办法。
2. **复用既有机制**。静态段 `info` / `token` 已验证不会与 UUID 本体 id 冲突（`src/app/api/mcp/[ontologyId]/route.ts:15`），`ask` 同理；鉴权（`withOwnAuth`）、令牌、工具策略、审计全部复用，不新增一套。
3. **可回滚**。删掉一个路由目录即恢复原状，无数据库迁移。

平台级 `/api/mcp?profile=ask` 作为二期可选项，一期不做——平台级端点没有「钉死本体」的语义，能力级调用必须先确定本体。

### Q4：`ask` 是同步返回还是异步？

**内部异步，对外两段式。**

```
ask_ontology(question, …)  → 202 语义：{ run_id, status: "RUNNING" }   （P95 < 1s）
get_answer(run_id)         → { status, answer, steps, evidence, … }
```

三条理由：

1. **MCP 这边没有推送**。`GET /api/mcp` 明确 405（`src/app/api/mcp/route.ts:24-27`），协议层只有请求-响应；
2. **超时不由平台决定**。反向代理（nginx 默认 `proxy_read_timeout 60s`）和各 MCP 客户端自己的工具调用超时，都会先于平台掐断。一次 20～90 秒的同步 `tools/call` 在客户端侧就已经失败，服务端做得再好也没用；
3. **平台已有这个模式**。`POST /api/reasoning/runs` 立即返回 202 + `runId`，结果走 `GET .../runs/{runId}/events`（SSE，带 `after` 游标可续传）。能力级出口只是把它换成轮询语义。

为了让「简单问题」不多一次往返，`ask_ontology` 提供 `wait_ms`（0～20000，默认 0）：

- `wait_ms > 0`：服务端最多等这么久，跑完就直接返回 `DONE` + 完整结果，跑不完返回 `RUNNING` + `run_id`；
- 接入方拿到 `RUNNING` 就按 1～2 秒间隔轮询 `get_answer`。

这样「快问题一次拿到、慢问题不超时」，且不需要协议层任何扩展。

### Q5：超时和步数上限由谁兜？

**服务端兜，且不需要改推理循环。**

- 步数：`maxSteps` 参数已存在，服务端再夹一次 `[1, 100]`（`run-registry.ts:48-49`、`agent.ts:106`）；
- 超时：`runReasoning` 只接受 `abortSignal`（`agent.ts:96`），没有 `timeoutMs`。但**中止路径已经把部分结果保住了**：

```ts
// src/lib/reasoning/agent.ts:292-299
} catch (error) {
  if (!options.abortSignal?.aborted) throw error;
  stopped = true;
}
if (!finished && options.abortSignal?.aborted) stopped = true;
```

abort 不抛错、只置 `stopped`，然后照常收尾并返回 `steps` / `evidence` / `usage`。所以超时实现就是**在外层加一个定时器 + 自己记一个 `timedOut` 标记**，用来把 `TIMEOUT` 和用户主动 `STOPPED` 区分开。**推理层零改动这一点是成立的**——但前提是别把 `timeoutMs` 塞进 `runReasoning`，塞进去就要改它。

默认值建议：

| 参数 | 默认 | 上限 | 说明 |
| --- | --- | --- | --- |
| `max_steps` | 12 | 100 | 不默认 100：成本随步数近似线性放大（见 Q9） |
| `timeout_ms` | 120000 | 300000 | 下限 15000；超时返回部分结果 |

### Q6：只返回 answer 够不够？为什么非要 evidence？

不够。这是「能力」而非「接口」的差别：

- `answer` 是自然语言，下游无法校验；
- `evidence` 是**真实对象的标识**（`ReasoningEvidence: kind + id + label + step`，`src/lib/reasoning/types.ts:29-36`），接入方可以回链到平台对象页核对，也可以在转述时标注「依据哪几个对象」；
- `steps` 让调用方能解释「这个数是怎么来的」，是排障与信任的基础。

没有 evidence 的问数接口，本质上只是一个「会说话的 CSV」。

### Q7：steps 是不是越全越好？

不是，必须默认收窄。

`steps[].result` 是**给模型看的工具返回原文**，单条最多保留 60k 字符（`agent.ts:36`）。10 步就是 600k 字符，直接撑爆调用方的上下文窗口，而且没有任何收益——接入方的模型不需要再读一遍平台模型的中间结果。

设计：

| 参数 | 取值 | 默认 |
| --- | --- | --- |
| `include_step_results` | `none` / `truncated` / `full` | `none` |

- `none`：只返回 `index`、`tool`、`arguments`、`summary`、`ok`、`elapsed_ms`（`summary` 已有，`types.ts:47-53`）；
- `truncated`：每步附前 2000 字符；
- `full`：原样，仅用于排障，需要在平台侧开权限。

### Q8：「只读」到底怎么保证？

分三层，`ask` 不新增任何一层，只是穿过它们：

| 层 | 机制 |
| --- | --- |
| 工具层 | 13 个工具里没有任何执行动作/写入的入口；`list_actions` 只列动作**定义** |
| 执行层 | `run_query` → 语义解析/绑定 → `QueryDsl` → 只读执行；`run_sql` → SQL 守卫（只允许只读语句） |
| 端点层 | `/api/mcp/ask/<id>` 只注册 `ask_ontology` / `get_answer` 两个工具，细粒度工具在这个端点上不存在 |

**但必须把话说准**：数据只读 ≠ 平台无写入。`runReasoning` 会写两处：

```ts
// src/lib/reasoning/run-registry.ts:294-299  审计（不可关）
await writeAuditEntry({ action: "REASONING_RUN", details: { … } });

// src/lib/reasoning/run-registry.ts:306-321  对话历史（可选）
if (worthKeepingTurn(run)) { const conversationId = await saveTurn({ … }); }
```

所以设计上加一个 `persist` 参数：

- `persist: false`（**能力级端点默认**）：只写审计，不写对话历史，避免外部 Agent 把人类用户的问答历史刷满；
- `persist: true`：与 UI 行为完全一致，落历史，可用 `conversation_id` 续接多轮。

对外文档写「数据只读；平台侧保留审计，对话记录可选」，不要写「无任何副作用」——否则第一次审计对账就会被质疑。

### Q9：成本有多大？要不要限流？

单次成本 = 每一步都把整段上下文重发一次。代码注释里有实测数：

```ts
// src/lib/reasoning/types.ts:92-95
/** 累计输入 / 输出 / 合计：**所有模型调用相加**（工具循环每一步都要把整段上下文重发一遍，所以很大）。 */
// 「10 步的 833.9k 会让用户以为…」
```

即一次 10 步的问数，输入 token 量可以到 80 万级。这是能力级出口最容易被低估的一项。

现状盘点：

- **没有任何账号级 QPS / 配额 / 令牌预算实现**；
- 只有数据源级的并发闸门：同一数据源最多 4 个并发 SQL（`src/lib/datasource/sql.ts:80`、`:142-152`），超了直接拒绝。

一期建议做「并发上限」而不是 QPS（问数是长任务，QPS 没有意义）：

| 限制 | 建议值 | 超限返回 |
| --- | --- | --- |
| 每令牌同时在跑 | 1 | `BUSY` + `retry_after_ms` |
| 每本体同时在跑 | 4 | `BUSY` + `retry_after_ms` |
| 每令牌每日调用 | 由交付方配置 | `QUOTA_EXCEEDED` |

二期再做 token 预算（用 `run.usage.totalTokens` 累计，字段已存在）。

### Q10：错误码怎么定？哪些该走 `isError`？

原需求把所有失败都写成 `isError: true`，这是不对的。**「没跑完」和「调用失败」是两件事**：很多 MCP 客户端拿到 `isError` 就只报错、丢弃 `structuredContent`，那样 `TIMEOUT` 带回来的部分结果等于白做。

| 错误码 | 通道 | 说明 |
| --- | --- | --- |
| `EMPTY_QUESTION` | `isError: true` | 问题为空/纯空白（内部还会 trim，`agent.ts:102-103`） |
| `QUESTION_TOO_LONG` | `isError: true` | 超过 500 字符（`run-registry.ts:45` 的既有上限） |
| `ONTOLOGY_NOT_FOUND` | `isError: true` | 本体不存在/未发布 |
| `MODEL_NOT_CONFIGURED` | `isError: true` | 平台没配模型（`agent.ts:100-101` 的原文错误） |
| `BUSY` | `isError: true` | 并发/配额超限，可重试 |
| `EXPIRED` | `isError: true` | `run_id` 已过期（内存运行只留 10 分钟，见 Q11） |
| `INTERNAL` | `isError: true` | 其它异常 |
| `TIMEOUT` | **`isError: false`** | 超时但带回已完成 steps / evidence |
| `TRUNCATED` | **`isError: false`** | 步数用满（`run.truncated`） |
| `STOPPED` | **`isError: false`** | 被主动取消（`run.stopped`） |
| `NO_ANSWER` | **`isError: false`** | 跑完但没形成结论 |

### Q11：多轮对话怎么接？有哪些坑？

参数 `conversation_id` 可选，不传即单轮。三个必须写进文档的坑：

1. **会话按账号隔离**。运行记录带 `userId`，他人的运行一律按不存在处理（`run-registry.ts:34`、`src/app/api/reasoning/runs/[runId]/events/route.ts:23-24`）。MCP 令牌代表哪个平台账号，会话就属于谁——换令牌续不上。
2. **运行只存在内存，保留 10 分钟**（`run-registry.ts:30-38`、`RUN_KEEP_MS`）。所以 `get_answer` 的有效窗口是 10 分钟，超时返回 `EXPIRED`，而不是无限轮询。这是原需求文档完全没提、但接入方一定会撞上的一条。
3. **`persist: false` 时不能续多轮**（历史没落库，下一轮没有上下文）。要续就必须 `persist: true`。

### Q12：会不会影响平台内部智能问答？

分三种影响，如实写：

| 影响面 | 会不会 | 说明 |
| --- | --- | --- |
| 工具开关 | **会（设计上接受）** | MCP 调试页的开关写的是同一份 `reasoning.toolPolicy`（全局 / 本体覆盖），内部问答也读它（`run-registry.ts:280`）。所以「在 MCP 调试里关掉 `run_query`」确实会让内部问答也看不到这个工具——这是「同一能力、同一策略」的必然结果，不是 bug |
| 模型算力 | **会** | 两边共用同一份 `LLM_BASE_URL` / `LLM_API_KEY`。外部 Agent 高频问数会挤占内部用户体验。**私有化交付必须给外部接入单独配一份模型凭据，或在网关侧按 key 限流**——这是本次定制的头号交付风险 |
| 运行隔离 | **不会** | 运行按 `userId` 隔离；能力级端点构造 `ToolContext` 时只读已发布定义，不共享可变状态 |

### Q13：为什么 `annotations` 是 P0，而不是「顺手做」？

因为它不是体验优化，而是**能不能用**的开关。MCP 客户端按 `annotations.readOnlyHint` 决定是否要求人工确认；没有这个字段，客户端只能按「未知副作用」处理，结果是每次调用都挂起等确认——接入方实测就是这个现象。

改动量：`tools/list` 的映射对象里加一个字段（`src/lib/reasoning/mcp-endpoint.ts:86-91`）。

取值建议：

```json
"annotations": {
  "title": "…中文名…",
  "readOnlyHint": true,
  "destructiveHint": false,
  "openWorldHint": false,
  "idempotentHint": true
}
```

注意：数据查询类工具幂等，`ask_ontology` **不幂等**（模型输出非确定），要如实标 `idempotentHint: false`，否则客户端可能做结果缓存。

### Q14：私有化环境下模型怎么配？能不能按请求指定模型？

不能按请求指定。`ask_ontology` 不接受 `model` 参数——否则等于对外开放「任选模型 + 任意提示词」的组合，成本与合规都失控。

模型来源仍是平台配置（`src/lib/reasoning/provider.ts`，`LLM_BASE_URL` / `LLM_API_KEY` / `LLM_MODEL`）。私有化场景下它指向**内网** OpenAI 兼容网关。需要按本体换模型时，走平台侧「问答配置」，不接受请求级覆盖。

### Q15：会不会读到草稿、未发布内容？

不会。`ToolContext` 由已发布定义构建，与 UI 问答完全一致（《系统说明书 10》10.11 节：工具只读已发布版本）。

### Q16：怎么验收？

原需求里「53 题对齐率 ≥95%」「P95 ≤ 90s」这两条不建议直接承诺，原因很具体：

- 平台**没有设置 `temperature`**（`provider.ts` 只配 baseUrl / apiKey / modelId），同一问题连跑两次 UI 问答本身就不保证数值一致；
- 把「P95 ≤ 90s」写进验收，等于把客户端超时也算到服务端头上。

改为可执行的四条：

| # | 验收项 | 判定 |
| --- | --- | --- |
| 1 | 同源 | 同一问题，UI 与 `ask_ontology` 的数值结果一致（人工抽样 20 题，比对关键数值） |
| 2 | 基线噪声 | 先测 UI 自身重复 3 次的数值一致率，作为上限；要求 `ask` **不低于该基线** |
| 3 | 时延 | `ask_ontology` 返回 202 语义的 P95 < 1s；完整答案的 P50/P95 只记录不承诺 |
| 4 | 隔离与审计 | 并发 4 个不同本体/账号的 `ask` 不串结果；每笔可在审计里查到 `run_id` / 账号 / 令牌 / 本体 / 步数 / token 用量 |
| 5 | 回滚 | 关闭 `/api/mcp/ask/*` 即恢复，无数据库迁移残留 |

### Q17：出问题怎么回滚？

零迁移设计：新端点是一个独立路由目录 + 一个独立的能力层模块，不改表结构、不改既有工具语义。回滚方式 = 下线路由（或反向代理 404）。已产生的对话历史与审计记录保留，不影响既有功能。

---

## 三、接口契约（草案）

### 3.1 工具一：`ask_ontology`

> 命名规范：既有工具参数是 **snake_case**（`ontology_id`、`type_name`、`data_source`），新工具必须一致，不要用 camelCase——同一个 `tools/list` 里混两种风格会让接入方的模型频繁传错参数。

```jsonc
{
  "name": "ask_ontology",
  "title": "本体问数",
  "description": "用本体平台自己的推理链路完成一次完整问数：一次调用即可拿到结论、依据与执行轨迹。只读，无数据写入。不要再用其它本体工具自行拼装。",
  "inputSchema": {
    "type": "object",
    "required": ["question"],
    "additionalProperties": false,
    "properties": {
      "question":              { "type": "string", "minLength": 1, "maxLength": 500 },
      "conversation_id":       { "type": "string", "description": "可选；续接多轮。需 persist=true 且同一令牌账号" },
      "max_steps":             { "type": "integer", "minimum": 1, "maximum": 100, "default": 12 },
      "timeout_ms":            { "type": "integer", "minimum": 15000, "maximum": 300000, "default": 120000 },
      "wait_ms":               { "type": "integer", "minimum": 0, "maximum": 20000, "default": 0 },
      "thinking":              { "type": "boolean", "description": "可选；是否开启模型思考" },
      "persist":               { "type": "boolean", "default": false, "description": "是否写入平台问答历史；默认 false，只写审计" },
      "include_step_results":  { "type": "string", "enum": ["none", "truncated", "full"], "default": "none" }
    }
  },
  "annotations": { "readOnlyHint": true, "destructiveHint": false, "openWorldHint": false, "idempotentHint": false }
}
```

### 3.2 工具二：`get_answer`

```jsonc
{
  "name": "get_answer",
  "title": "取问数结果",
  "description": "按 run_id 取一次问数的当前状态与结果。RUNNING 时按 1-2 秒间隔重试；run_id 保留 10 分钟。",
  "inputSchema": {
    "type": "object",
    "required": ["run_id"],
    "additionalProperties": false,
    "properties": {
      "run_id":               { "type": "string" },
      "include_step_results": { "type": "string", "enum": ["none", "truncated", "full"], "default": "none" }
    }
  },
  "annotations": { "readOnlyHint": true, "destructiveHint": false, "openWorldHint": false, "idempotentHint": true }
}
```

### 3.3 返回结构

`content[0].text` 放同一份 JSON（与 `structuredContent` 一致，`mcp-endpoint.ts:128-136` 已用这个约定）。

```jsonc
{
  "status": "DONE",                 // RUNNING | DONE | TRUNCATED | TIMEOUT | STOPPED | NO_ANSWER | EXPIRED | BUSY | ERROR
  "run_id": "…",
  "answer": "周洋桦 / 财务经理 / 51,438.68 元/月 ……",
  "ontology": { "id": "…", "name": "…", "version": "v1.2.0" },
  "steps": [
    { "index": 1, "tool": "search_schema", "arguments": { "query": "月薪" }, "summary": "命中 66 个概念", "ok": true, "elapsed_ms": 812 }
  ],
  "evidence": [
    { "kind": "OBJECT_TYPE", "id": "Employee", "label": "员工", "step": 2 }
  ],
  "artifacts": { "sql": "SELECT …", "row_count": 1, "truncated": false },
  "usage": { "prompt_tokens": 12000, "completion_tokens": 1500, "total_tokens": 13500, "last_input_tokens": 4200 },
  "run": { "step_count": 6, "max_steps": 12, "elapsed_ms": 21800, "model": "…", "conversation_id": null, "persisted": false },
  "error": null
}
```

失败时：

```jsonc
{ "status": "ERROR", "run_id": null, "error": { "code": "MODEL_NOT_CONFIGURED", "message": "…", "retryable": false } }
```

### 3.4 状态机

```
ask_ontology
  ├─ 参数/权限/本体错误 ─────────────→ ERROR(code)              [isError:true]
  ├─ 并发/配额超限 ──────────────────→ BUSY(retry_after_ms)     [isError:true]
  ├─ wait_ms 内跑完 ────────────────→ DONE | TRUNCATED | TIMEOUT | STOPPED | NO_ANSWER
  └─ wait_ms 内没跑完 ──────────────→ RUNNING(run_id)

get_answer(run_id)
  ├─ 不存在 / 超 10 分钟 ───────────→ EXPIRED                 [isError:true]
  ├─ 还在跑 ────────────────────────→ RUNNING
  └─ 结束 ──────────────────────────→ DONE | TRUNCATED | TIMEOUT | STOPPED | NO_ANSWER | ERROR
```

### 3.5 内部字段映射

| payload 字段 | 来源 | 备注 |
| --- | --- | --- |
| `answer` | `ReasoningRun.answer` | `types.ts:116` |
| `steps[]` | `ReasoningRun.steps` | 默认剥掉 `result` |
| `evidence[]` | `ReasoningRun.evidence` | 已在 `agent.ts:309` 去重 |
| `usage` | `ReasoningRun.usage` | snake_case 转换 |
| `run.step_count` / `max_steps` | `ReasoningRun.stepCount` / `.maxSteps` | `types.ts:128-129` |
| `status` | 派生 | `truncated` → TRUNCATED；`stopped` + 本地 `timedOut` → TIMEOUT；`stopped` → STOPPED；`answer` 为空 → NO_ANSWER；否则 DONE |
| `artifacts.sql` | **派生**：扫 `steps` 里 `tool ∈ {run_query, run_sql}` 的最后一步 `.arguments.sql` | 不是现成字段 |
| `artifacts.row_count` | **派生**：解析该步 `result` JSON 的 `rows.length` | 解析失败则为 `null`，不要抛错 |
| `run.elapsed_ms` | `ReasoningRun.elapsedMs` | |
| `run.persisted` | 落库结果 | `saveTurn` 成功即 true |

---

## 四、运行时序

```
接入方 Agent                平台 MCP 端点                    推理层
     │                          │                             │
     │ ask_ontology(question)   │                             │
     ├─────────────────────────►│ 校验 reasoning.use          │
     │                          │ loadToolPolicy(ontologyId)  │
     │                          │ startReasoningRun ─────────►│ runReasoning(…)
     │                          │◄──────────── runId (202) ────┤  工具循环
     │◄── RUNNING(run_id) ──────┤                             │  ├─ search_schema
     │                          │                             │  ├─ get_object_type
     │ get_answer(run_id)       │                             │  └─ run_query
     ├─────────────────────────►│ 读内存运行（按 userId 校验）  │
     │◄── RUNNING ──────────────┤                             │
     │        …1~2s 轮询…        │                             │
     │ get_answer(run_id)       │                             │ 收尾：answer/evidence
     ├─────────────────────────►│◄──────────── ReasoningRun ──┤ 审计 + (可选)落历史
     │◄── DONE + 全量结果 ───────┤                             │
```

（`wait_ms > 0` 时，第一步就等在服务端，跑完直接返回 `DONE`，省掉轮询。）

---

## 五、改动清单

| # | 文件 | 改动 | 风险 |
| --- | --- | --- | --- |
| 1 | `src/lib/reasoning/mcp-endpoint.ts:86-91` | `tools/list` 输出 `annotations` | 低（纯增量字段） |
| 2 | `src/lib/reasoning/mcp-capability.ts`（新） | 两个工具定义 + `ReasoningRun → payload` 映射 + 超时定时器 + 状态派生 | 中（新逻辑集中在此，便于评审） |
| 3 | `src/app/api/mcp/ask/[ontologyId]/route.ts`（新） | 能力级端点，`tools/list` 只回两个工具 | 低 |
| 4 | `src/lib/reasoning/run-registry.ts:42-60` | `startReasoningRun` 增加 `persist?: boolean`（默认沿用现有行为，能力级端点传 false） | 中（要保证 UI 行为不变） |
| 5 | `src/lib/reasoning/run-registry.ts` | 新增 `getReasoningRunSnapshot(runId, userId)`：只读快照 + 所有权校验 | 低 |
| 6 | `src/lib/reasoning/agent.ts` | **不改**（超时在外层用 `abortSignal` + 本地 `timedOut`） | 无 |
| 7 | 审计 | `REASONING_RUN.details` 增加 `channel: "MCP_CAPABILITY"`、`tokenId`、`persist` | 低 |
| 8 | `docs/系统说明书/10-MCP服务与外部集成.md` | 增加 10.x「能力级端点」一节 | 低 |

**明确不做**：不改提示词（`prompt.ts`）、不改工具语义（`registry.ts`）、不加 MCP 服务端推送、不动数据库。

---

## 六、里程碑

| 阶段 | 内容 | 预估 |
| --- | --- | --- |
| P0 | `annotations` + 能力级端点骨架 + `ask_ontology` / `get_answer` + `persist:false` | 1～2 人日 |
| P1 | 超时（含部分结果）、并发闸、错误码、审计字段、文档 | 2～3 人日 |
| P2 | `wait_ms` 语义打磨、`include_step_results`、平台级 `?profile=ask` | 1～2 人日 |
| P3（可选） | Streamable HTTP SSE 推送 + `notifications/progress` | 5 人日以上，非必需 |

P0 完成即可让接入方跑通「一次调用拿到答案」，P2 之后体验接近 UI。

---

## 七、待确认

1. `max_steps` 默认 12 是否够（客户样本问题里最长用到几步）？
2. `persist` 默认 false 是否被接受（不接受则必须给外部 Agent 单独的会话空间）？
3. 并发/配额阈值由谁定（平台方还是交付方）？
4. 外部接入是否单独配一份模型凭据（Q12 的头号风险）？
5. 是否需要支持图片提问（内部 `runReasoning` 支持 attachments，`agent.ts:76-77`；但 MCP 传 base64 体积过大，建议一期不做）。

---

## 附录 A：与原始需求文档的差异

| 原需求写法 | 本设计 | 原因 |
| --- | --- | --- |
| 「暴露 10 个工具」 | 13 个定义 / 默认暴露 9 个 | 数字随策略变化，写死会被指出错误（`registry.ts:107-264` + `mcp.ts:97-109`） |
| 「推理层零改动」 | 成立，但前提是超时放外层 | `runReasoning` 无 `timeoutMs`；abort 路径已保留部分结果（`agent.ts:292-299`） |
| 同步返回 + 服务端兜超时 | 异步两段式 + `wait_ms` | MCP 无推送，客户端与网关 60s 级超时先掐断 |
| `artifacts` 字段「已存在」 | 需从 `steps` 派生 | `ReasoningRun` 无 sql / rowCount 字段（`types.ts:110-138`） |
| 「只读、无副作用」 | 「数据只读；审计必留，历史可选」 | `runReasoning` 会写审计与历史（`run-registry.ts:294-321`） |
| 「计入现有配额/限流」 | 无现成实现，需新增并发闸 | 全仓无 rate-limit，只有数据源并发 4（`datasource/sql.ts:80`） |
| camelCase 参数 | snake_case | 与既有 13 个工具一致 |
| `TIMEOUT` 走 `isError: true` | 走 `isError: false` | 否则客户端丢弃 `structuredContent`，部分结果白做 |
| 「P95 ≤ 90s」「53 题 ≥95%」 | 改为「不低于 UI 自身重复运行的基线」 | 未设 temperature，平台自身也不保证确定性 |
| 「MCP progress 通知 / 异步」列为 P2「中」 | 明确 P3，5 人日以上 | `GET /api/mcp` 恒 405，无推送通道 |
| 备选 `get_ontology_brief()` 可「补上一大截」 | 可做，但只解决「有什么」 | `schemaBrief` 只含概念名与绑表，**不含属性列**（`prompt.ts:87-149`） |

## 附录 B：术语

| 术语 | 含义 |
| --- | --- |
| 工具级对接 | 对外暴露细粒度查询工具，由调用方自行编排推理 |
| 能力级对接 | 对外暴露「问数」这一完整能力，调用方只传问题、取结果 |
| 运行（run） | 一次完整的推理执行，含 steps / evidence / answer / usage，内存保留 10 分钟 |
| 证据（evidence） | 工具执行中命中的真实对象/类型/指标标识，可回链核对 |
| 概念清单（schemaBrief） | 平台自动生成的本体摘要，注入 instructions |