# PRD：AI 原生开发操作系统

**版本**：v0.1  
**日期**：2026-04-01  
**理论基础**：blog/1.md（认知结构 + LLM 物理定律）  
**工程参考**：Claude Code v2.1.88 源码、Paperclip v1、Slock.ai

---

## 一、问题陈述

当前 AI 辅助开发的主流做法是"凭直觉塞"——把能想到的都塞进 CLAUDE.md，或者开二十个 Claude Code 终端各自为战，不知道谁在做什么、花了多少钱、为什么这样做。

更本质的问题是：**人机边界没有被认真设计**。人类的理解和决策没有被编码成 LLM 可消费的结构化知识产物，导致 Agent 知道要做什么，但不知道为什么，遇到约束时不知道哪些可以打破、哪些不能动。

本系统要解决的问题：**在 Paperclip 控制平面 + Claude Code 执行引擎的基础上，填补知识层、协调层和归因层的缺口，让整条管线（理解→决策→表达→验证→反馈）真正闭合。**

---

## 二、设计原则

直接继承 blog/1.md 推导出的约束，不重复论证：

| 原则 | 来源 | 工程含义 |
|------|------|----------|
| 上下文即全部 | L1 | 所有影响 Agent 行为的知识必须显式加载，未加载 = 不存在 |
| 坍缩隔离 | L0 | 不同角色的 Agent 必须在独立 session 中运行，防止方向污染 |
| 状态必须外置 | L2 | 一切需要跨 heartbeat 存活的信息，必须持久化到控制平面 |
| 验证强度正比于后果 | L3 + 定理三 | 低风险自动通过，高风险强制人工确认 |
| 知识按变更频率分层 | 定理二 | 慢变 → Git，中变 → Paperclip DB，快变 → MCP，任务内 → Scratchpad |
| 人类负责理解和决策 | 认知模型 | Board 是唯一的理解和决策主体，产出物跨越人机边界 |

---

## 三、系统架构

```
┌──────────────────────────────────────────────────────────────────┐
│  治理层  Board UI                                                 │
│  输入: Initiative → 输出: PRD + ADR + 约束清单 + 模块定义        │
│  工具: Paperclip Board + 结构化知识编辑器                        │
├──────────────────────────────────────────────────────────────────┤
│  协调层  IM Channel + Paperclip 控制平面                         │
│  CEO Agent: 任务分解 → Issue 树 + 依赖图                         │
│  频道: 每个 Project 一个频道，进展自动 post，@mention 即唤醒     │
├──────────────────────────────────────────────────────────────────┤
│  执行层  Claude Code Heartbeat                                    │
│  Worker Agents: 6 维坍缩配置 × N 个角色                         │
│  Scratchpad: 同一 Issue 下跨 Worker 共享工作记忆                 │
├──────────────────────────────────────────────────────────────────┤
│  知识层  双轴设计                                                 │
│  物理轴: System Context（慢变）| User Context（快变）            │
│  语义轴: user / feedback / project / reference                   │
│  检索层: Haiku side-query 语义选择（≤5 条）                     │
├──────────────────────────────────────────────────────────────────┤
│  持久化层  频率匹配栈                                            │
│  Git（PRD/ADR）→ Paperclip DB（任务/评论）→ MCP（环境状态）     │
└──────────────────────────────────────────────────────────────────┘
```

---

## 四、功能模块

### 4.1 知识层（填补 Paperclip V1 最大缺口）

**问题**：Paperclip V1 的 `goals.description` 是纯文本，Agent 无法从中获取结构化的"为什么"。

**设计**：在 Paperclip 的 document 系统中定义五种知识产物类型，每种有固定 frontmatter schema：

```markdown
---
type: prd | adr | constraint | module | task-list
title: ""
status: draft | approved | deprecated
relates_to: [issue-id, ...]
created_by: board | agent-id
---
```

**五种类型的语义**：

| 类型 | 回答的问题 | 生产者 | 消费者 |
|------|-----------|--------|--------|
| `prd` | 做什么、不做什么 | Board | 所有 Agent |
| `adr` | 为什么这样做、为什么不选其他 | Board / CEO | 设计决策时的 Agent |
| `constraint` | 不能做什么、边界条件 | Board | 所有 Agent |
| `module` | 怎么划分、模块边界 | CEO Agent | Engineer Agent |
| `task-list` | 何时做、优先级 | CEO Agent | Worker Agent |

**ADR 触发规则**：当 CEO Agent 在 heartbeat 内做架构决策时，必须调用 `/adr` skill，自动生成 ADR 草稿并关联到当前 Issue。Board 可以选择 approve/reject。未经记录的架构决策视为无效。

---

### 4.2 语义化上下文注入（Heartbeat Step 5.5）

**问题**：`heartbeat-context` 返回结构化数据，但 Agent 不知道知识库里有哪些相关文档。

**设计**：在 Heartbeat 的 Checkout（Step 5）之后、开始工作（Step 6）之前，插入语义检索步骤：

```
Step 5.5 — Memory Probe（由 adapter 自动执行，不消耗主 Agent token）

输入: issue.title + issue.description
执行: Haiku side-query，扫描 company knowledge base 所有文档的 frontmatter
输出: ≤5 个最相关文档的路径和摘要
注入方式: 追加到 paperclipSessionHandoffMarkdown（execute.ts 已有此字段）
```

**实现细节**：
- 文档 frontmatter 必须包含 `description` 字段（≤100 字，用于检索）
- 检索结果按 `relevance × recency` 排序
- 已在本 session 中加载过的文档不再重复选择（`alreadySurfaced` 模式）

---

### 4.3 Agent 角色规范（6 维坍缩配置）

**问题**：当前 Agent 的角色定义是 adapter config 里的 jsonb blob，控制平面对其语义一无所知。

**设计**：每个 Agent 必须配置完整的 6 维坍缩配置，存储为结构化字段：

```typescript
roleConfig: {
  roleDefinition: string,      // 50-200 词，精确的坍缩方向
  toolWhitelist: string[],     // 该角色可用的工具列表
  maxTurnsPerRun: number,      // 单次 heartbeat 最大 turn 数
  model: "haiku" | "sonnet" | "opus",
  permissionMode: "auto" | "ask" | "plan",
  knowledgeLayers: string[],   // 优先加载的知识类型
}
```

**标准角色库**：

| 角色 | 坍缩方向 | 工具 | 模型 | permissionMode |
|------|---------|------|------|----------------|
| CEO | 全局编排、任务分解、不执行代码 | paperclip, read | sonnet | ask |
| Engineer | 代码实现、测试、重构 | bash, edit, read, paperclip | sonnet | auto |
| Reviewer | 业务逻辑审查、不修改代码 | read, paperclip | sonnet | plan |
| Security | 安全审计、权限检查 | read, paperclip | opus | plan |
| Researcher | 信息收集、文档整理 | read, web, paperclip | haiku | auto |

**约束**：`roleDefinition` 必须是确定性文本（无时间戳、无随机元素），保证 prompt cache 字节级一致性。

---

### 4.4 IM 协调层（Slock.ai 启发）

**问题**：Paperclip 的人工介入路径是 Board UI → 创建/修改 Issue，摩擦高；Agent 之间只能通过 Issue 评论异步通信。

**设计**：每个 Paperclip Project 对应一个 IM 频道，双向同步：

**频道 → Paperclip（下行）**：
- `@agent-name <message>` → 以最高优先级唤醒对应 Agent，注入 `PAPERCLIP_WAKE_REASON=channel_mention`
- `/approve <issue-id>` → 触发 Paperclip 审批通过
- 普通消息 → 自动创建或更新关联 Issue 评论

**Paperclip → 频道（上行）**：
- Heartbeat 完成 → 自动 post 进展摘要（`parsedStream.summary`，≤3 句）
- Issue 状态变更 blocked / done → 频道通知
- 预算告警 → 频道通知

**@mention 的优先级语义**：
```
PAPERCLIP_WAKE_REASON=channel_mention 时：
  Step 4 override: 以频道消息作为最高优先任务
  即使当前没有分配的 Issue 也执行
  执行完毕后继续正常 inbox 流程
```

**MVP 实现路径**：Slack webhook 双向同步；长期可参考 Slock.ai 构建 agent-native 原生频道。

---

### 4.5 Session 粒度修复

**问题**：`agent_task_sessions` 的 `taskKey` 粒度不明确，跨 Issue 的上下文可能污染或丢失。

**设计**：

```
Session 唯一键: (agentId, issueId)
```

- 同一 Agent 处理不同 Issue → 不同 session，坍缩态隔离
- 同一 Agent 多次 heartbeat 同一 Issue → `--resume sessionId`，上下文连续
- Issue 关闭 → session 标记 archived

---

### 4.6 Scratchpad（跨 Worker 工作记忆）

**问题**：CEO 将 Issue 拆分给多个 Engineer 并发执行时，Workers 之间无法共享中间状态。

**设计**：每个 Issue 附属一个轻量 KV 存储：

```
GET  /api/issues/{id}/workpad
POST /api/issues/{id}/workpad        { key, value, authorAgentId }
DELETE /api/issues/{id}/workpad/{key}
```

**语义**：
- 生命周期绑定 Issue
- 所有被分配到该 Issue 子任务的 Agent 均可读写
- 写入自动追加到 activity log
- 不参与语义检索，不通知 Board

典型用途：协调锁（`auth-module: in-progress by agent-X`）、中间产物路径、并发共识。

---

### 4.7 失败归因机制

**问题**：heartbeat 失败只记录 `lastError: text`，没有结构化归因，同样的错误必然重复。

**设计**：heartbeat 进入 blocked 时，Paperclip skill 触发归因流程：

```
/attribute-failure "<描述>"
  → Agent 从四个方向选择归因目标：
    [1] 修 Skill（执行流程有 bug）
    [2] 补 Spec/ADR（知识缺失）
    [3] 刷新环境状态（MCP 查询结果过期）
    [4] 调整角色配置（坍缩方向不匹配）
  → 自动创建对应 follow-up Issue 并 assign
  → 失败记录结构化存储:
     { issueId, failureType, attributedTo, createdFollowUpId }
```

**规则**：blocked 状态的 Issue 在创建 follow-up 之前不允许退出 heartbeat。"记住下次不要这样做"不是归因。

---

### 4.8 后果感知的审批门

**问题**：当前审批门只覆盖治理层操作，执行层的高风险操作（数据库迁移、生产部署）没有拦截。

**设计**：Issue 增加 `riskLevel: low | medium | high`，由 CEO Agent 在分解任务时设置：

| riskLevel | permissionMode | 需要 Review Agent | 需要 Board 确认 |
|-----------|---------------|------------------|----------------|
| low | auto | 否 | 否 |
| medium | ask | 是（Reviewer） | 否 |
| high | plan | 是（Reviewer + Security） | 是 |

`riskLevel=high` 的 Issue 进入 done 之前必须有 Board 的显式 approve 记录。

---

## 五、人机边界协议

Board 在启动 CEO Agent 之前，必须确保以下五个知识产物都已存在且 `status=approved`：

```
对于每个 Initiative：
  ✓ prd         — 做什么、不做什么、边界
  ✓ adr         — 关键架构决策及理由（至少一条）
  ✓ constraint  — 硬约束清单
  ✓ module      — 模块划分与边界（CEO Agent 产出，Board 审批）
  ✓ task-list   — 可执行的 Issue 树（CEO Agent 产出，Board 审批）
```

五个产物缺一不可。CEO Agent 的 bootstrap prompt 中明确：任一产物缺失时，不得开始执行，必须创建 `blocked` Issue 请求 Board 补充。

---

## 六、约束清单

**硬约束**：
- Agent 的 `roleDefinition` 必须是确定性文本，不含动态元素
- Session 唯一键必须是 `(agentId, issueId)`，不允许跨 Issue 复用
- `riskLevel=high` 的 Issue 未经 Board approve 不允许进入 done
- blocked heartbeat 必须在退出前完成归因并创建 follow-up Issue
- ADR 必须包含 What / Why / Alternatives considered 三段

**软约束**：
- `roleDefinition` 建议 50-200 词
- 语义检索每次最多选 5 个文档
- CEO Agent 单次 heartbeat maxTurns 建议 ≤30
- IM 频道消息摘要建议 ≤3 句话

**本版本边界外**：
- 多 Board 成员治理
- 跨公司 Agent 协作
- 非 token 成本的收入/支出追踪
- IM 频道原生实现（MVP 用 Slack webhook）

---

## 七、验收标准

一个 Initiative 从 Board 输入到 Agent 交付的完整管线跑通，且满足：

1. **知识完整性**：Agent 可以从 heartbeat-context + 语义检索中回答"为什么这个模块选择了这个方案"
2. **坍缩隔离**：同一 Agent 在两个不同 Issue 上运行时，session ID 不同，无上下文交叉
3. **失败可追溯**：任意 blocked heartbeat 都有结构化归因记录和 follow-up Issue
4. **成本可控**：每个 Agent 的 token 消耗可按 Issue 粒度查询，`riskLevel=high` 有 Board 审批记录
5. **人工介入零摩擦**：Board 通过 IM 频道 @mention 可在 30 秒内改变 Agent 下一个 heartbeat 的优先级

---

## 八、开放问题

1. **IM 层边界**：在 Paperclip 内部实现 IM，还是通过 webhook 集成 Slack/Discord？后者 MVP 快，前者长期体验更好。

2. **Haiku side-query 成本**：Agent 数量多时，每次 heartbeat 的语义检索成本不低。是否设置"知识库文档 > N 条才触发"的阈值？

3. **workpad schema**：纯 KV 够用，还是需要带类型（lock / artifact / consensus）？

4. **ADR 生产者边界**：是否允许 Worker Agent 创建 ADR 草稿，再由 Board/CEO approve？
