# Sumus 拆解报告（工作流层）

> 用一个反向工程工具（REA）对 [sumus.im](https://sumus.im) 做的一次黑盒拆解，目的是**判断它的多 Agent 工作流哪些部分值得自建**，不是复刻它。
>
> 拆解时间：2026-10-28 · 样本：已登录工作区页面 · 方法：CDP 直连浏览器 + 静态包分析（不执行页面 JS）

---

## 0. 一句话结论

Sumus 的工作流内核由 **5 个原语**组成：单通道事件总线、Turn 状态机、群组消息模型、@ 护栏、Runtime 抽象。**没有模型能力，全部是工程**。商业模式是 OpenRouter 转售 + 提示词模板市场 + 积分制。

要自建一个"多 Agent 协作工作流"，需要复刻的是这 5 个原语，不需要它的任何一行代码。

---

## 1. 方法与可信度

| 项 | 值 |
|---|---|
| 工具 | REA `rea-agents` 6.5.0（逆向工程 CLI + MCP 服务） |
| 观测点 | 本机 Chrome 155.0.8059.86，CDP `ws://127.0.0.1:9222/devtools/browser/<redacted>` |
| 分析操作 | `list_browser_targets`、`analyze_web_bundle`（静态解析，**JS 从不执行**） |
| 证据编号 | `ev_ee0a0d2b…`（包分析）、`ev_29827610…`（目标列表） |
| 解析完整性 | 52 个脚本全解析，**0 失败**，1,242,824 个 AST 节点，`completeness: complete` |
| 另取样 | 页面实际加载 59 个 JS 文件，5.1 MB，用于关键词定位 |

**证据分级**（本报告严格区分）：

- **[实测]** — 我在登录态页面里真实调用接口拿到的返回
- **[静态]** — 前端代码里读到的字符串/结构，未执行验证
- **[推断]** — 由上述两者推导，明确标注

> 未做：突破权限、抓取他人数据、反编译后端（后端未暴露字节码，全部为 REST）。

---

## 2. 技术栈指纹

| 层 | 结论 | 依据 |
|---|---|---|
| 前端框架 | **Next.js**（高置信）+ **React**（中置信） | REA vendor_fingerprint，多处命中 |
| 监控 | **Sentry** | `sentry-trace` / `traceparent` / `baggage` 注入点 |
| 实时通道 | **SSE**（Server-Sent Events） | `new EventSource("/api/app/events")` |
| 计费 | **Stripe** | `/api/app/billing` 返回 `provider: "stripe"` |
| 模型网关 | **OpenRouter** | 模型对象的 `provider` 字段直接写着 `openrouter` / `openrouter-latest` |

---

## 3. 数据模型（实测）

### 3.1 Agent 实体

`GET /api/app/agents?workspace_id=<id>` 返回的完整字段（实测，脱敏）：

```
id, workspace_id, callsign, tagline, description, starter_prompts,
avatar, tags, can_do, cant_do,
runtime_kind          # cloud | local
local_runtime_id      # local 时指向本机 runtime
local_agent_driver    # 本机驱动，如 codex / claude code
model                 # null = 用模板默认
thinking_level        # 推理强度
monthly_budget_credits# 该 agent 每月积分上限，null = 不限
teammate_access       # 关键：能否被其他 agent 调用
archived_at, system_owned, template_id, created_by,
created_at, updated_at, last_engaged_at,
can: {edit, delete, talk, view_sessions}
```

**要点**：`teammate_access` 是"AI 自己拉队友"那个卖点的开关；`monthly_budget_credits` 是成本闸门；`runtime_kind` 决定跑在谁家机器上。

### 3.2 路由（静态）

```
/app/w/:workspaceId
/app/w/:workspaceId/agent-marketplace
/app/w/:workspaceId/connectors/{open,done}
/app/w/:workspaceId/members
/app/w/:workspaceId/members/:id
/app/w/:workspaceId/members/:id/settings     ← 单个 Agent 的模型/思考等级在这里改
/app/w/:workspaceId/members/api
/app/w/:workspaceId/members/people/:userId
/app/w/:workspaceId/overview
/app/w/:workspaceId/scheduled
```

> 交互细节：聊天输入框里**没有**模型选择器。模型绑在 Agent 上，不在对话上。对话右上角「对话信息」只能看当前模型，不能改。

### 3.3 API 面（实测 + 静态）

| 端点 | 说明 |
|---|---|
| `GET /api/app/events` | **SSE 单通道**，全站事件都从这里推 |
| `GET /api/app/me` | 当前用户 |
| `GET /api/app/agents?workspace_id=` | Agent 列表与配置 |
| `GET /api/app/models` | 可用模型清单（21 个） |
| `GET /api/app/billing` | 套餐、积分汇率、用量档 |
| `GET /api/app/agent-marketplace` | Agent 市场（需登录） |
| `GET/POST /api/app/connector-servers` | MCP 连接器管理 |
| `POST /api/app/events` `GET /api/app/analytics/purchases` | 埋点 |
| `/api/auth`、`/api/surveys/` | 认证、问卷 |

---

## 4. 工作流五大原语（本文核心）

### 4.1 单通道事件总线 + resync

```js
t = new EventSource("/api/app/events")
t.onopen = () => { y++ > 0 && b({ type: "resync" }) }   // 重连即全量重同步
```

全站**只连一条 SSE**，而不是每个会话一条。断线重连后客户端主动发 `resync`，让服务端重新下发全量状态。

**为什么值得抄**：N 个会话 = N 条连接的做法在移动端和弱网下必炸。单通道 + resync 是更稳的形态。

### 4.2 Turn 状态机

事件类型（静态）：`turn_started` → `message_delta`（流式增量）→ `turn_done` / `turn_error`，均带 `session_id`；另有 `stream_resuming`（续接）。

**注意 `session_id` 不是 `chat_id`**：一个会话里可以有多轮 turn，turn 是一等对象，可单独重连、单独失败、单独重试。

### 4.3 群组消息模型

事件类型：`group_message`、`group_round`（一轮里多个 agent 依次发言）、字段 `thread_root_id`（把追问挂到某个根消息下）。

**为什么值得抄**：AI 群里"一轮多人说话"是核心显示问题，`group_round` 把它变成一等事件，前端不用猜谁该在上谁该在下。

### 4.4 @ 护栏（最容易踩的坑）

静态命中：`group_mention_all`、`mention_all_forbidden`、`group_mentioned`。

**推断**：@全体被服务端限制，不允许 agent 无限 @ 全体——这是防多 Agent 互相触发死循环的第一道闸。自建时必须在服务端做：最大递归深度 + 每轮调用次数上限。

### 4.5 Runtime 抽象

字段 `runtime_kind: cloud | local`，配套静态文案：`runtime_pairing_hint`、`runtime_reconnect_code_hint`、`runtime_offline_count`、`runtime_waiting`、`runtime_retry`、`runtime_preparing`、`runtime_registration_failed`、`runtime_command_label`。

**含义**：本机 agent（Codex / Claude Code）通过"配对码"注册成一个 runtime，掉了要重连，离线会计数。这就是"把你的 Codex 拉进来当队友"的真实机制——不是 API 直连，是**本机常驻进程 + 配对**。

---

## 5. 模型与计费（实测）

### 5.1 模型清单：21 个，全走 OpenRouter

Claude Opus 5.5 / Sonnet 5.5 / Fable 5.1 / Haiku 5.5；Grok 4.7；GPT-6 Astra / 6.1 Sol / 6 Luna / Chat Latest / gpt-oss-120b；Gemini 3.8 Flash / 4 Argon；DeepSeek V4.1 Flash 1017 / V4 Pro 0813 / V4 Flash；GLM 5.3；Kimi K3 / K2.7 Code；Qwen3.7 Plus；MiniMax M3；Mercury 3。

### 5.2 计费

| 项 | 值 |
|---|---|
| 汇率 | **200 积分 = 1 美元** |
| Free | $0，1,000 积分/月，3 人，10 个 agent |
| Plus | $24/月（年付 $20），3,600 积分，25 人，300 个 agent |
| Pro（未公开） | $100/月，15,000 积分，50 人 |
| Team（未公开） | $15/月，2,250 积分，10 个 agent |
| 加购 | $20 → 4,000 积分（档位 4K / 10K / 20K / 40K / 100K） |
| 结转 | 不结转 |

**判断**：3,600 积分 ≈ $18 额度，几乎按成本价给。它赚的是订阅差和加购，不是模型差价。

### 5.3 Agent 市场

24 个预置角色：科研助手、文案润色师、客户猎手、实验分析师、视频剪辑师、增长搭子、家庭管家、行政助理、策略回测员、个股调研员、财报解读员、异动盯盘员、行情晨报员、交易复盘员、策略审查员、指标编写员、文献研究员、编程搭子、剪辑工作台、约会策划师、纽约家长助手、广告守门员、点子陪审员、竞品哨兵。

**观察**：一半是生活场景（家庭管家 / 纽约家长助手 / 约会策划师）。说明目标是**消费级"AI 家人"**，不是专业团队。这是它的定位，也是它的天花板。

---

## 6. 抄什么 / 不抄什么

| 抄 | 理由 |
|---|---|
| 单通道 SSE + resync | 弱网稳定性，工程量小，收益大 |
| Turn 状态机（session_id 独立） | 让"正在干活"可表达、可恢复、可重试 |
| group_round + thread_root_id | 多 Agent 对话的显示与串接基础 |
| 服务端 @ 护栏（深度 + 次数） | 不自建这个必被死循环打爆 |
| runtime_kind 二态（云 / 本机） | 大任务下放到本机，成本结构立刻不同 |
| Agent 上的 monthly_budget 闸门 | 成本可预测 |

| 不抄 | 理由 |
|---|---|
| 积分制 / Stripe 多租户 | 我们不需要卖，只需要控成本 |
| OpenRouter 转售 | 直接对接自有 key 更便宜、更可控 |
| 消费级角色模板 | 与业务无关 |
| 它的前端交互细节 | 抄结构即可，UI 我们自己定 |

---

## 7. 自建方案（v0.1 范围）

**技术选型**：Python 3 标准库（`http.server` + `ThreadingHTTPServer` + `sqlite3` + 手写 SSE），零第三方依赖；前端单页 HTML + 原生 JS；模型走 OpenAI 兼容接口（DeepSeek / 自建 vLLM 均可）。

```
浏览器 ──EventSource──► Python stdlib 服务
                            ├─ SQLite: groups / agents / messages / turns
                            ├─ 模型层: OpenAI 兼容 HTTP
                            └─ 工作目录: 产物落文件
```

**v0.1 最小集（能验证工作流好不好用）**：

1. 一个群 + 多个 agent（名字 / 人设 / 模型 / prompt 可配）
2. @ 谁谁答，可 @ 多个并行
3. agent 之间可派活（`ask_teammate`），**服务端硬护栏**：最大 2 层、每轮 ≤5 次
4. 事件总线 + turn 状态机 + 断线 resync
5. 消息与 turn 落 SQLite，可回看
6. 产物落文件，预留接入现有工作台

**规模估计**：后端 ~600 行，前端 ~400 行。

---

## 8. 未解 / 待验证

- 后端语言与部署形态未知（前端只能看到 REST 表面，看不到实现）
- `group_round` 的排序依据（按发言顺序还是按 agent 优先级）未验证
- 派活是否真有递归、深度上限多少 —— 只有 `mention_all_forbidden` 这一处护栏证据
- 并发 turn 的上限、限流策略未见
- 云端沙箱的实际隔离强度（容器？VM？）无法从外部判断

## 9. 边界声明

- 本报告为**黑盒行为观察与架构归纳**，用于自研参考。
- 未抓取、未存储、未分发 Sumus 的源代码；仓库内不含其任何 JS 产物。文中代码片段均为**单行行为证据引用**（如 `new EventSource(...)`），属事实性引用。
- 未复制其提示词原文、角色文案、品牌资产。
- 未绕过任何权限、未获取他人数据；所有观测发生在报告者本人已登录的账号内。

---

*生成于 2026-10-28 · 工具：REA 6.5.0 · 结构数据见 [`findings.json`](./findings.json)*
