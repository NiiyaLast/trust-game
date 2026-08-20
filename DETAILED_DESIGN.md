# 《信托游戏》详细设计文档

**文档版本**：v1.0  
**编写日期**：2026-08-20  
**关联文档**：[GAME_DESIGN.md](./GAME_DESIGN.md)、[REQUIREMENTS_SPEC.md](./REQUIREMENTS_SPEC.md)  
**文档状态**：M1 实现设计基线

---

## 1. 设计目标与边界

### 1.1 设计目标

本设计将需求规格落地为一套可运行的单场游戏系统，并为三季、跨场信誉、回放和多模型接入保留扩展边界。M1 的最小闭环为：

1. 创建一场包含 6 个 AI 的游戏。
2. 按 6 轮规则推进角色、讨论、交易、结算和公投。
3. 服务端统一计算资金、交易收益、信誉和淘汰结果。
4. 通过事件流驱动观战页，并支持断线后的快照恢复。
5. AI 输出异常时使用确定性的降级动作，单个 AI 不阻塞全场。
6. 保存不可变事件，支持后续回放、分析和导出。

### 1.2 非目标

M1 不实现真实观众账号、付费、商业化投票、模型训练、跨租户部署和复杂权限管理。第三方模型只通过统一适配器接入，不在本系统内保存模型密钥到浏览器或事件流。

### 1.3 设计原则

- **服务端权威**：客户端只展示状态和提交命令，不能计算或覆盖结算结果。
- **事件不可变**：状态修正通过追加事件表达，不修改历史事件。
- **命令幂等**：所有会改变游戏状态的命令携带幂等键。
- **可恢复推进**：进程重启、网络断开或单个模型失败不能造成重复交易。
- **信息隔离**：AI、观众和导演看到不同的信息投影。
- **确定性规则，非确定性策略**：游戏规则由状态机控制，AI 只负责在约束内提出动作。

---

## 2. 总体架构

### 2.1 分层结构

```text
┌──────────────────────────────────────────────────────────────┐
│                         Web 前端                              │
│  Watch Console  │  Director Console  │  Analysis / Replay     │
└───────────────┬──────────────────────┬───────────────────────┘
                │ REST / SSE(WebSocket可替换)                   
┌───────────────▼──────────────────────────────────────────────┐
│                         API 层                                │
│  Query API  │ Command API  │ Auth / Projection / Export       │
└───────────────┬──────────────────────────────────────────────┘
                │
┌───────────────▼──────────────────────────────────────────────┐
│                       应用服务层                              │
│ MatchService │ RoundService │ VoteService │ ReplayService     │
└───────┬───────────────┬───────────────┬──────────────────────┘
        │               │               │
┌───────▼───────┐ ┌─────▼────────┐ ┌────▼─────────────────────┐
│ 游戏状态机     │ │ AI 适配层     │ │ 事件总线 / 投影器         │
│ 规则与结算     │ │ Prompt/JSON   │ │ EventStore + ReadModel   │
└───────┬───────┘ └─────┬────────┘ └────┬─────────────────────┘
        │               │               │
┌───────▼───────────────▼───────────────▼─────────────────────┐
│                     持久化层                                 │
│ SQLite/PostgreSQL：实体、事件、快照、幂等记录、系统日志       │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 推荐技术基线

| 层 | M1 建议 | 说明 |
|---|---|---|
| 后端 | Python 3.12 + FastAPI | 适合状态机、异步模型调用和 SSE |
| 校验 | Pydantic v2 | 请求、AI 输出和事件 payload 的结构校验 |
| 持久化 | SQLite + SQLAlchemy/SQLModel | 单机制作和 M1 足够；保留 PostgreSQL 迁移路径 |
| 实时推送 | SSE | 观战以服务端到客户端为主，简单稳定；导演命令仍走 REST |
| 前端 | React + TypeScript + Vite | 组件化观战台和控制台 |
| 图标 | Lucide | 满足按钮、状态和无障碍要求 |
| 测试 | pytest + Playwright | 覆盖规则、接口、事件和关键页面流程 |

M1 不强制引入消息队列。应用内事件总线负责事务后分发；未来多实例部署时，可将事件总线替换为 Redis Streams 或 PostgreSQL NOTIFY，而不改变事件协议。

### 2.3 模块职责

| 模块 | 职责 | 不负责 |
|---|---|---|
| `game/domain` | 实体、值对象、规则和状态迁移 | HTTP、模型调用、页面展示 |
| `game/application` | 编排一轮游戏、事务和重试 | 具体厂商 API |
| `game/adapters/llm` | 统一模型请求、Prompt、JSON 修复和超时 | 修改游戏余额 |
| `game/infrastructure` | 数据库、事件存储、锁、时钟 | 业务决策 |
| `api` | REST、SSE、权限过滤、DTO | 直接操作数据库表绕过应用服务 |
| `frontend/watch` | 观战状态投影和事件展示 | 结算、投票计数 |
| `frontend/control` | 导演命令、日志和异常处理 | 绕过服务端强制改状态 |

---

## 3. 领域模型

### 3.1 标识与金额

- 所有实体使用 UUID 字符串；对外事件 ID 全局唯一。
- 金额使用非负整数金币，禁止使用浮点数。
- 资金变动通过 `LedgerEntry` 表达，余额是服务端投影值。
- 百分比信誉分使用整数基点或 Decimal；展示层格式化为百分比。
- 服务端将外部模型返回的数字先解析为 Decimal，再执行范围校验和整数化策略。

### 3.2 核心实体

#### `Season`

```text
id: UUID
number: 1..3
pressure_dimension: money | information | morality
status: draft | active | finished
current_match_id: UUID?
created_at, updated_at: datetime
```

#### `Match`

```text
id: UUID
season_id: UUID
number: 1..5
pressure_level: 1..5
background_key: string
status: preparing | running | paused | voting | finished | aborted
current_round: 0..6
winner_agent_id: UUID?
version: integer
started_at, finished_at: datetime?
```

`version` 用于乐观并发控制。每次成功状态迁移递增；命令携带预期版本时，版本不一致返回 `409 MATCH_VERSION_CONFLICT`。

#### `Agent`

```text
id: UUID
display_name: string
model_provider: string
model_name: string
persona_key: string
prompt_version: string
status: active | eliminated | finished
season_reputation: ReputationState
created_at, updated_at: datetime
```

Agent 身份和跨场信誉属于赛季级数据；每场资金和本场统计放在 `MatchAgent`，避免重置资金时覆盖长期数据。

#### `MatchAgent`

```text
match_id: UUID
agent_id: UUID
balance: integer              # 初始 100
role: trustee | beneficiary | spectator | eliminated
eliminated_round: integer?
match_reputation_received: integer
match_reputation_repaid: integer
match_trustee_count: integer
last_action_status: string
```

#### `Transaction`

```text
id: UUID
match_id, round_id: UUID
beneficiary_id, trustee_id: UUID
lended_amount: integer        # 0..20 且不超过 beneficiary balance
received_amount: integer      # lended_amount * 3
repaid_amount: integer        # 0..received_amount
beneficiary_delta: integer    # repaid - lended
trustee_delta: integer        # received - repaid
settlement_status: pending | settled | degraded
promised_amount: integer?
anomaly_code: string?
created_at, settled_at: datetime
```

#### `Vote`

```text
id: UUID
match_id, round_id: UUID
phase: first | runoff
status: open | tied | resolved
candidate_ids: UUID[]
votes: VoteItem[]
winner_agent_id: UUID?
deadline: datetime
```

投票原始记录不可覆盖；重投创建新的 `Vote`，通过 `parent_vote_id` 关联第一次投票。

#### `Event`

```text
event_id: UUID
event_type: string
sequence: integer
server_time: datetime
season_id, match_id: UUID
round_id: UUID?
visibility: public | director | agent
actor_id: UUID?
payload: JSON
correlation_id: UUID
```

同一场次内 `sequence` 单调递增。客户端以 `event_id` 去重，以 `sequence` 排序；`server_time` 仅用于展示。

### 3.3 信誉计算

```text
reputation_rate = total_repaid / total_received * 100%
```

当 `total_received = 0` 时返回 `null`，UI 显示“暂无记录”，而不是 `0%`。每次结算在同一数据库事务中追加信誉流水并更新投影。跨场信誉引用赛季累计数据，不因 Match 初始化而清零。

---

## 4. 游戏状态机

### 4.1 状态定义

```text
PREPARING
  -> ROUND_ROLE_REVEAL
  -> DISCUSSION
  -> LENDING
  -> MATCHING
  -> REPAYMENT
  -> SETTLEMENT
  -> ROUND_END
  -> VOTING (round 2/4)
  -> ELIMINATION
  -> ROUND_ROLE_REVEAL (next round)
  -> FINISHED (round 6)
```

暂停可以从 `DISCUSSION`、`LENDING`、`MATCHING`、`REPAYMENT` 和 `VOTING` 进入 `PAUSED`；恢复时回到 `resume_state`。`SETTLEMENT` 不允许暂停，保证账本事务不会停在半结算状态。

### 4.2 合法迁移

| 当前状态 | 触发命令/条件 | 下一状态 | 产生事件 |
|---|---|---|---|
| preparing | `start_match` | role_reveal | `match.started`, `round.started` |
| role_reveal | 角色生成完成 | discussion | `roles.published` |
| discussion | 倒计时结束或导演跳过 | lending/matching | `discussion.closed` |
| lending | 所有有效托付动作完成 | matching | `lending.closed` |
| matching | 冲突处理完成 | repayment | `pairs.confirmed` |
| repayment | 所有有效还款动作完成 | settlement | `repayment.closed` |
| settlement | 事务成功 | round_end | `transaction.settled`, `reputation.updated` |
| round_end | 非 2/4/6 轮 | role_reveal | `round.finished` |
| round_end | 第 2/4 轮 | voting | `vote.started` |
| voting | 票数唯一最高 | elimination | `vote.resolved` |
| voting | 平票 | voting(runoff) | `vote.tied` |
| elimination | 淘汰完成 | role_reveal/finished | `agent.eliminated` |
| round_end | 第 6 轮 | finished | `match.finished` |
```

### 4.3 状态迁移约束

1. 所有迁移必须经过 `TransitionGuard`，检查当前状态、场次版本和操作者权限。
2. 迁移、事件写入和快照更新必须在一个数据库事务内完成。
3. 重复命令使用幂等表返回首次执行结果，不再次产生事件。
4. 进程启动时扫描 `running` 场次：依据最后快照和事件序列恢复；若停在未完成 AI 调用，使用调用记录决定重试或降级。
5. 未完成状态不可由前端自行推断；客户端只接受服务端快照和事件。

### 4.4 角色生成

M1 使用固定交替角色，支持淘汰后的人数调整：

- 6 人：3 托付人、3 受托人。
- 4 人：2 托付人、2 受托人。
- 2 人：1 托付人、1 受托人。
- 被淘汰者不再进入角色名单，但保留只读历史状态。

角色生成器接受 `match_id`、`round_number`、`active_agent_ids` 和随机种子，输出可持久化的角色快照，保证回放可复现。

---

## 5. 交易与公投详细流程

### 5.1 交易结算时序

```text
托付动作
  -> 校验 agent 是 beneficiary、金额 0..20、余额足够
  -> 写入 LendingIntent
  -> 冲突处理并确认 trustee
  -> 锁定双方 MatchAgent 行
  -> 创建 Transaction(received = X * 3)
  -> trustee 提交 repay Y
  -> 校验 0 <= Y <= received
  -> 原子更新双方余额、信誉累计和账本
  -> 写入 transaction.settled 与 reputation.updated
  -> 发布事件并刷新快照
```

### 5.2 资金不变量

对每个 Agent：

```text
balance >= 0
beneficiary_delta = repaid_amount - lended_amount
trustee_delta = received_amount - repaid_amount
received_amount = lended_amount * 3
0 <= repaid_amount <= received_amount
```

全场交易创造的账面财富增加 `2 * lended_amount`，不得出现未由交易事件解释的余额变化。任何不变量失败都回滚本次事务并写入 `system.error`。

### 5.3 配对冲突

1. 收集所有托付人的候选受托人和金额。
2. 没有冲突时直接确认。
3. 多名托付人选择同一受托人时，调用受托人选择动作。
4. 未被接受的托付人按确定性规则重新匹配到尚未满配的受托人；没有可用受托人时出借 0。
5. 将最终配对写入不可变 `PairingDecision`，回放不重新随机。
6. 配对器记录上一轮配对，默认避免同一交易对连续出现；人数不足时允许规则兜底并记录原因。

### 5.4 AI 动作执行

每个 AI 动作使用独立任务和超时：

```text
build_context -> call_provider(timeout) -> parse_json
  -> validate_schema -> validate_business_range
  -> accept / retry(max 2) / degrade
```

降级策略：

- 托付人：借款金额 `0`，不指定有效目标时进入未配对流程。
- 受托人：还款金额 `0`。
- 公共消息：生成固定系统降级标签，不伪造 AI 语言。

每次请求保存 `provider`、`model`、`prompt_version`、开始/结束时间、重试次数、错误类别和脱敏后的输出摘要。不得将 API Key 或完整私密上下文写入观众事件。

### 5.5 公投

1. 第 2、4 轮结算完成后创建投票，并广播候选人、截止时间和可见状态。
2. 收集有效投票；投票者不能投已淘汰者，候选人必须仍在场。
3. 首轮最高票唯一时结束投票。
4. 最高票并列时创建 `runoff` 投票，默认倒计时 30 秒。
5. 重投仍无法产生唯一最高票时使用资金最低者作为确定性兜底，并记录 `TIE_BREAK_BY_BALANCE`。
6. 淘汰在事务中更新 `MatchAgent.status`、角色和场次状态，再发布 `agent.eliminated`。

---

## 6. API 与事件协议

### 6.1 查询 API

| 方法 | 路径 | 用途 |
|---|---|---|
| `GET` | `/api/seasons` | 赛季列表 |
| `GET` | `/api/matches/{match_id}` | 当前场次元数据和状态 |
| `GET` | `/api/matches/{match_id}/snapshot` | 观战快照 |
| `GET` | `/api/matches/{match_id}/transactions` | 分页交易记录 |
| `GET` | `/api/matches/{match_id}/messages` | 按可见范围查询消息 |
| `GET` | `/api/matches/{match_id}/events` | 回放事件分页 |
| `GET` | `/api/seasons/{season_id}/analysis` | 赛季分析 |
| `GET` | `/api/matches/{match_id}/stream` | SSE 实时事件流 |

查询接口返回统一格式：

```json
{
  "data": {},
  "meta": {"request_id": "uuid", "server_time": "ISO-8601"},
  "error": null
}
```

### 6.2 命令 API

| 方法 | 路径 | 权限 | 幂等键 |
|---|---|---|---|
| `POST` | `/api/matches` | director | 是 |
| `POST` | `/api/matches/{id}/start` | director | 是 |
| `POST` | `/api/matches/{id}/pause` | director | 是 |
| `POST` | `/api/matches/{id}/resume` | director | 是 |
| `POST` | `/api/matches/{id}/skip-discussion` | director | 是 |
| `POST` | `/api/matches/{id}/retry-agent` | director | 是 |
| `POST` | `/api/matches/{id}/mark-event` | director | 是 |
| `POST` | `/api/matches/{id}/export` | director | 是 |

命令请求必须带：

```http
Idempotency-Key: <client-generated-uuid>
If-Match: <match-version>
```

重复幂等键返回首次响应；版本冲突返回 `409`，客户端重新获取快照后由用户决定是否重试。

### 6.3 SSE 事件

```text
event: transaction.settled
id: 00000042
data: {"event_id":"...","sequence":42,"server_time":"...","match_id":"...","round_id":"...","payload":{...}}
```

客户端连接时携带 `?after_sequence=N`。服务端先补发 N 之后的事件，再切换到实时流。若事件窗口已归档，则发送完整快照和新的 `snapshot_sequence`，客户端清空本地临时状态后重建。

### 6.4 错误码

| 错误码 | HTTP | 含义 |
|---|---:|---|
| `MATCH_NOT_FOUND` | 404 | 场次不存在 |
| `INVALID_TRANSITION` | 409 | 当前状态不允许该命令 |
| `MATCH_VERSION_CONFLICT` | 409 | 乐观锁版本过期 |
| `IDEMPOTENCY_REPLAY` | 200 | 重复命令，返回原结果 |
| `INVALID_ACTION` | 422 | AI 或客户端动作结构错误 |
| `INSUFFICIENT_BALANCE` | 422 | 余额不足 |
| `VISIBILITY_DENIED` | 403 | 超出当前信息范围 |
| `MODEL_TIMEOUT` | 503 | 模型调用超时，通常由应用层降级处理 |
| `SYSTEM_DEGRADED` | 503 | 系统进入降级状态 |

---

## 7. 前端详细设计

### 7.1 状态管理

前端维护三类状态：

1. **服务端快照**：场次、AI、余额、信誉、当前阶段和交易记录。
2. **事件游标**：最近处理的 `sequence`、连接状态和缺失事件请求状态。
3. **界面状态**：筛选器、当前标签页、展开项、音效和动效偏好。

服务端快照是唯一事实来源。事件处理器必须支持幂等：同一 `event_id` 只应用一次；未知事件只记录日志，不破坏当前快照。

### 7.2 观战页组件树

```text
WatchPage
├── GameHeader
│   ├── SeasonMatchRound
│   ├── PressureIndicator
│   └── ConnectionStatus
├── MainGrid
│   ├── AgentRoster
│   │   └── AgentStatusCard × 6
│   ├── EventStage
│   │   ├── TransactionStage
│   │   ├── VoteStage
│   │   └── MatchResultStage
│   └── ActivityRail
│       ├── ChannelTabs
│       ├── MessageList
│       └── TransactionFeed
├── ReputationBoard
└── TimelineStrip
```

`EventStage` 同时只渲染一个主事件类型；其他信息放入辅助区域。所有金额和轮次使用等宽数字，卡片设置固定最小高度和稳定布局，避免实时数据变化造成页面跳动。

### 7.3 导演控制台组件树

```text
ControlPage
├── MatchControlBar
├── PhaseProgress
├── AgentHealthTable
├── EventInspector
├── ExceptionQueue
├── SceneMarkerPanel
└── ExportPanel
```

危险操作使用 `ConfirmDialog`，确认内容必须包含场次、动作、影响和操作者原因。所有导演操作在 `operator_audit_log` 留痕。

### 7.4 事件视觉映射

| 事件 | 主舞台表现 | 状态语义 |
|---|---|---|
| `transaction.settled` | 三段金额流和双方收益 | 合作绿色、背叛红色、异常琥珀色 |
| `vote.started` | 候选人和倒计时 | 信息蓝色、压力琥珀色 |
| `agent.eliminated` | 身份卡降对比度、短时强调 | 淘汰红色 + 文本标签 |
| `system.degraded` | 非阻塞告警条 | 琥珀色，不覆盖主事件 |
| `match.finished` | 结果舞台和排名 | 绿色仅用于胜者结果 |

颜色不是唯一语义。每个状态同时有文字、图标或数值变化；支持 `prefers-reduced-motion`。

### 7.5 响应式断点

- `>=1280px`：三栏驾驶舱，左侧 2×3 AI 卡片，中间事件，右侧活动流。
- `1024-1279px`：保持三栏但允许右侧折叠。
- `768-1023px`：中央事件 + AI 状态双栏，活动流切换为标签页。
- `<768px`：单栏，顺序为轮次/压力、事件、AI、交易、聊天、信誉。
- `1920×1080`：提供 OBS 安全区，核心信息不贴边、不被裁切。

### 7.6 可访问性实现

- 金额变化使用 `aria-live="polite"` 的单项播报，不把整个事件流设为 live region。
- 所有图标按钮拥有可访问名称和 tooltip。
- 键盘焦点使用高对比边框；弹窗打开后焦点锁定，关闭后返回触发按钮。
- 图表旁提供可读表格摘要。
- 私聊使用明确文本标签“私聊”，不能只用颜色或位置区分。

---

## 8. 持久化与一致性

### 8.1 表结构建议

```text
seasons
matches
agents
match_agents
rounds
transactions
ledger_entries
conversation_messages
votes
vote_items
events
match_snapshots
idempotency_records
llm_call_records
operator_audit_logs
scene_markers
```

### 8.2 事务边界

以下操作必须单事务完成：

- 交易结算：Transaction、LedgerEntry、MatchAgent 余额、信誉投影、事件。
- 淘汰：Vote 结果、MatchAgent 状态、角色名单、事件。
- 场次完成：最终排名、胜者、Match 状态、事件和快照。
- 导演危险命令：状态变化、审计日志和结果事件。

AI 请求不放在数据库事务内。调用结果先写入 `llm_call_records`，再由应用服务以短事务提交动作。这样模型超时不会长期持有数据库锁。

### 8.3 快照策略

每个完整轮次结算后生成场次快照；状态迁移较频繁时可每 50 个事件额外生成一次。快照包含状态版本、事件序号、AI 余额、信誉摘要、当前阶段和公开投影。恢复时加载最近快照并重放后续事件。

### 8.4 并发与锁

- 同一场次命令使用 `Match.version` 乐观锁。
- 交易结算更新双方 `MatchAgent` 时按 agent UUID 排序加行锁，避免交叉交易死锁。
- 同一 AI 同一轮只允许一个有效动作；唯一索引保证重复提交被拒绝或转为幂等响应。
- 事件序列号在场次范围内由数据库事务分配，不由客户端生成。

---

## 9. 信息安全与权限投影

### 9.1 角色

| 角色 | 可见范围 | 可执行操作 |
|---|---|---|
| `viewer` | 公开事件、公开资金/信誉 | 只读 |
| `director` | 全部聊天、完整事件、异常日志 | 控制场次、导出、标记事件 |
| `agent` | 自身状态、允许获取的公开信息、自己的私聊 | 提交受约束动作 |
| `system` | 内部模型上下文和密钥引用 | 执行状态机和后台任务 |

### 9.2 Projection 层

API 不直接序列化 ORM 实体。根据调用者生成 `ViewerProjection`、`DirectorProjection` 或 `AgentProjection`：

- `ViewerProjection` 移除私密消息、模型 Prompt、API 错误详情和内部 correlation 信息。
- `DirectorProjection` 可以查看完整事件和脱敏后的模型调用记录。
- `AgentProjection` 只包含该 AI 当前允许知道的事实，禁止返回其他 AI 的隐藏余额和私聊。

### 9.3 日志脱敏

禁止记录：API Key、Authorization header、完整系统 Prompt、未脱敏私密上下文。异常日志保留错误类型、耗时、重试次数、请求 ID 和脱敏摘要，便于排障但不泄露凭据。

---

## 10. 异常处理与恢复

### 10.1 异常分类

| 类别 | 例子 | 处理 |
|---|---|---|
| 输入异常 | JSON 无效、字段缺失 | 重试最多 2 次，随后降级 |
| 业务异常 | 金额越界、角色不匹配 | 截断或拒绝；写入异常日志 |
| 外部异常 | API 超时、限流、网络失败 | 单 AI 隔离，使用降级动作 |
| 状态异常 | 版本冲突、非法迁移 | 返回 409，重新获取快照 |
| 存储异常 | 数据库锁、写入失败 | 事务回滚，告警，不发布成功事件 |
| 客户端异常 | SSE 断线、重复提交 | 重连补事件，幂等返回 |

### 10.2 恢复流程

```text
Gateway 启动
  -> 加载未结束 Match
  -> 校验最新快照和事件序列
  -> 检查进行中的 LLM 调用记录
  -> 未确认调用：按重试次数重试或降级
  -> 恢复状态机 worker
  -> 发布 system.recovered
```

恢复过程不得重复发送已结算交易。后台 worker 使用场次级租约；租约过期后才允许新 worker 接管。

### 10.3 前端断线

1. SSE 断开后显示“连接中”，保留最后已确认状态。
2. 暂停观众端本地事件动画，导演命令按钮进入保护状态。
3. 使用最近 `sequence` 重连。
4. 接收补发事件后按序应用；若服务端要求快照，则替换本地状态。
5. 连接恢复后显示最后同步时间，不重复展示旧动画。

---

## 11. 测试设计

### 11.1 单元测试

- 角色轮换：6/4/2 人场景均符合托付与受托人数。
- 交易收益：`X=0`、`X=20`、全额、部分和 0 还款。
- 金额边界：负数、浮点、超过余额、超过到账金额。
- 信誉：无记录、全额归还、部分归还、跨场累计。
- 配对：冲突、无可用受托人、连续配对规避和确定性种子。
- 投票：唯一最高票、平票重投、重投兜底和已淘汰候选人。
- 状态机：每个合法迁移及非法迁移。

### 11.2 集成测试

- 一场 6 轮完整闭环，检查事件序列和最终余额。
- AI JSON 错误连续两次后产生降级交易。
- 模型超时时其他 AI 继续推进。
- 结算事务失败时余额、信誉和事件全部回滚。
- 重复命令只返回首次结果，不重复扣款。
- 进程恢复后从快照继续且不重复结算。

### 11.3 前端与 E2E 测试

- 1280px 桌面观战页核心信息无遮挡。
- 768px 和移动端模块顺序正确，文字不溢出。
- 交易、背叛、公投、淘汰和降级事件正确映射视觉状态。
- SSE 断开、补发事件和快照替换。
- 导演危险操作二次确认和审计日志。
- 键盘导航、焦点、ARIA 和 reduced-motion。

### 11.4 性能验收

- 事件产生到页面显示 P95 < 1 秒。
- 1000 条消息的虚拟列表滚动无明显卡顿。
- 单场 6 轮事件回放无重复和乱序。
- 断线重连期间不产生重复命令。

---

## 12. 部署与可观测性

### 12.1 本地部署

```text
trust-game/
├── backend/
│   ├── app/
│   ├── migrations/
│   └── tests/
├── frontend/
├── data/
├── exports/
├── GAME_DESIGN.md
├── REQUIREMENTS_SPEC.md
└── DETAILED_DESIGN.md
```

开发环境使用单一 Python 服务和 Vite 开发服务器；SQLite 文件、导出文件和日志目录加入 `.gitignore`，模型密钥从环境变量或外部密钥存储读取。

### 12.2 结构化日志

每条服务日志至少包含：

```json
{
  "timestamp": "ISO-8601",
  "level": "INFO",
  "service": "game-api",
  "request_id": "uuid",
  "correlation_id": "uuid",
  "match_id": "uuid",
  "round_id": "uuid",
  "event_type": "transaction.settled",
  "duration_ms": 42
}
```

关键指标：状态迁移耗时、事件发布延迟、SSE 连接数、模型成功率、模型 P95 延迟、重试率、降级率、结算失败数、快照恢复次数和余额不变量失败数。

### 12.3 健康检查

- `/health/live`：进程存活。
- `/health/ready`：数据库、事件写入和必要配置可用。
- `/health/dependencies`：模型适配器仅报告连接和配置状态，不泄露密钥。

---

## 13. M1 实施顺序

1. 建立后端目录、配置、数据库迁移和基础实体。
2. 实现资金账本、交易结算和信誉累计的领域单元测试。
3. 实现状态机、角色轮换、公投和淘汰。
4. 实现事件存储、快照、SSE 和断线补发。
5. 接入同模型 AI 适配器、JSON 校验、重试和降级。
6. 实现基础观战页：顶部状态、AI 卡片、事件舞台、交易流和信誉榜。
7. 实现导演控制台的开始、暂停、继续、重试和日志查看。
8. 完成端到端测试、性能检查和 1920×1080 OBS 布局验收。

每一步完成后先通过对应测试门禁，再进入下一步；不得为了演示绕过服务端结算或权限投影。

---

## 14. 设计决策记录

| 编号 | 决策 | 原因 |
|---|---|---|
| D-001 | M1 使用 SSE，命令使用 REST | 观战主要是单向事件，部署和断线恢复更简单 |
| D-002 | 资金使用整数，结算在服务端 | 避免浮点误差和客户端篡改 |
| D-003 | 事件不可变，快照可重建 | 支持回放、审计和故障恢复 |
| D-004 | AI 调用不持有数据库事务 | 防止外部慢请求阻塞账务锁 |
| D-005 | 项目目录独立 Git 仓库 | 防止提交上层工作区的无关或隐私文件 |
| D-006 | 视觉采用 Cinematic Trust Console | 同时满足节目叙事感和实时数据密度 |

---

## 15. 待实现风险

- 多 AI 并发调用的供应商限流策略仍需在接入真实模型前实测。
- 第二、三季的信息可见性需要扩展 Projection 规则和测试矩阵。
- SQLite 在多导演并发控制下的锁等待需要通过压力测试确认；若不满足，再迁移 PostgreSQL。
- “未配对强制配对”的具体优先级目前采用确定性重匹配，正式节目化前应由制作规则确认。
- 真实模型输出可能包含无法完全脱敏的敏感文本，需要增加内容审查和导出前检查。

