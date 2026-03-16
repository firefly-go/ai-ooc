# Claude Code 监控指标分析文档

> 基于 [Claude Code 官方监控文档](https://code.claude.com/docs/en/monitoring-usage) 整理的指标重点关注指南

---

## 目录

1. [概述](#概述)
2. [全量指标清单](#全量指标清单)
3. [重点关注指标（优先级排序）](#重点关注指标优先级排序)
4. [日志事件指标](#日志事件指标)
5. [告警建议](#告警建议)
6. [常用 PromQL 查询示例](#常用-promql-查询示例)

---

## 概述

Claude Code 通过 **OpenTelemetry（OTel）** 标准协议上报监控数据。数据类型分为两类：

| 类型 | 说明 | 用途 |
|------|------|------|
| **Metrics（指标）** | 时序数值数据，上报到 Prometheus/Victoria Metrics 等 | 趋势分析、告警 |
| **Logs/Events（日志事件）** | 离散事件，上报到 Loki/OpenSearch 等 | 详情排查、审计 |

---

## 全量指标清单

### Metrics 指标

| 指标名称 | 类型 | 含义 | 常用维度（Attributes） |
|----------|------|------|----------------------|
| `claude_code.session.count` | Counter | CLI 会话启动次数 | `user.account_uuid`, `organization.id`, `app.version` |
| `claude_code.token.usage` | Counter | Token 消耗量 | `token.type`（input/output/cache_read/cache_creation）, `model`, `session.id` |
| `claude_code.cost.usage` | Counter | API 调用累计费用（美元） | `session.id`, `user.account_uuid`, `organization.id`, `model` |
| `claude_code.active_time.total` | Counter | 与 Claude Code 交互的活跃时长（秒） | `session.id`, `user.account_uuid` |
| `claude_code.lines_of_code.count` | Counter | 代码行变更数量（新增/删除） | `session.id`, `user.account_uuid` |
| `claude_code.commit.count` | Counter | 在会话中创建的 Git commit 数量 | `session.id`, `user.account_uuid` |
| `claude_code.pull_request.count` | Counter | 在会话中创建的 PR 数量 | `session.id`, `user.account_uuid` |
| `claude_code.code_edit_tool.decision` | Counter | 代码编辑工具的决策次数（接受/拒绝） | `decision.source`（config/user/policy）, `session.id` |

### 通用维度（所有指标共有）

| 维度名 | 含义 |
|--------|------|
| `session.id` | 唯一会话标识，用于关联同一次会话的所有事件 |
| `app.version` | Claude Code 版本号 |
| `user.account_uuid` | 用户账户 UUID（已认证状态下） |
| `organization.id` | 组织 UUID（企业场景下） |
| `terminal.type` | 终端类型（iTerm、VSCode、tmux 等） |
| `model` | 使用的模型（sonnet、opus、haiku 等） |

---

## 重点关注指标（优先级排序）

### 🔴 P0 — 成本与用量控制（必须关注）

#### 1. `claude_code.cost.usage` — 累计费用

**为什么最重要：** 这是最直接反映 Claude Code 资源消耗的指标。费用超支不可回退，需要主动监控。

```
关注维度：
- 按 user.account_uuid 分组 → 找到费用最高的用户
- 按 model 分组 → 了解各模型成本占比
- 按 organization.id 分组 → 团队/项目层面费用分摊
- 按时间段（天/周/月）聚合 → 趋势分析与预算控制
```

**建议告警：** 单日费用超过预算阈值时告警（例如：日费用 > $10 则触发告警）

---

#### 2. `claude_code.token.usage` — Token 消耗量

**为什么重要：** Token 是费用的来源，细粒度分析有助于降本优化。

```
关注维度：
- token.type = "input" → 输入 Token，主要受提示词长度影响
- token.type = "output" → 输出 Token，通常单价更贵
- token.type = "cache_read" → 缓存命中读取（价格低，说明缓存有效）
- token.type = "cache_creation" → 缓存写入（价格适中）
```

**优化思路：** `cache_read / (input + cache_creation)` 比值越高，说明提示词缓存利用率越好，成本越低。

---

### 🟠 P1 — 生产力与效能评估（应该关注）

#### 3. `claude_code.lines_of_code.count` — 代码变更量

**为什么重要：** 结合费用可以计算"每美元产生的代码行数"，量化 AI 辅助编程的 ROI。

```
核心公式：
代码效能 = lines_of_code.count / cost.usage（美元）

解读：
- 比值高 → Claude Code 高效地产出了代码
- 比值低 → 可能存在大量非代码类对话（设计讨论、问答等）
```

---

#### 4. `claude_code.commit.count` + `claude_code.pull_request.count` — 工程交付量

**为什么重要：** 代码行数偏原始，commit 和 PR 更能反映工程实际产出。

```
关注场景：
- commit 量增长但 cost 未增长 → 效率提升
- PR 数量增长 → AI 辅助完成了更多功能交付
- 结合时间段统计每周/每月的工程产出趋势
```

---

#### 5. `claude_code.active_time.total` — 活跃使用时长

**为什么重要：** 帮助理解开发者实际投入在 Claude Code 上的时间，结合 token 消耗分析"每小时的 token 消耗效率"。

```
活跃时长 vs 费用：
- 长时间低费用 → 用户在做低 token 消耗的对话（高性价比）
- 短时间高费用 → 有大量上下文的密集调用（需留意）
```

---

#### 6. `claude_code.session.count` — 会话次数

**为什么重要：** 反映使用频率与用户活跃度。

```
关注场景：
- 用于计算"每会话平均费用" = cost.usage / session.count
- 识别异常会话（单次会话费用异常高，可能存在死循环或失控调用）
- 按时间段绘制使用趋势图，了解采用率变化
```

---

### 🟡 P2 — 辅助分析（有余力可关注）

#### 7. `claude_code.code_edit_tool.decision` — 工具决策统计

**为什么有用：** 统计 AI 建议被采纳（accept）vs 被拒绝（reject）的比例。

```
关注维度：
- decision.source = "user" → 用户手动允许/拒绝
- decision.source = "config" → 配置策略自动决策
- decision.source = "policy" → 组织策略决策

解读：
- 拒绝率高 → AI 建议质量可能不符合期望，需要调整提示策略
- 接受率高 → AI 建议被广泛采纳，说明效果好
```

---

## 日志事件指标

Events 是非聚合的离散日志，适合做详情排查，不适合直接做告警，但可以在日志系统（如 Loki）中建立索引查询。

| 事件名称 | 含义 | 主要排查场景 |
|----------|------|-------------|
| `claude_code.api_request` | 每次 API 调用记录 | 排查哪些请求消耗了大量 token |
| `claude_code.api_error` | API 调用错误事件 | 排查频繁失败的原因（限流、网络、认证等） |
| `claude_code.user_prompt` | 用户发送给模型的提示词（默认脱敏） | 审计和内容分析（需开启非脱敏模式） |
| `claude_code.tool_result` | 工具执行结果（含成功/失败、耗时） | 排查哪些工具调用失败率高 |
| `claude_code.tool_decision` | 工具权限决策记录 | 安全审计、合规检查 |

> ⚠️ **隐私提示：** `claude_code.user_prompt` 默认会脱敏用户输入内容。如需完整记录，需要在环境变量中显式开启，且需遵守数据合规要求。

---

## 告警建议

根据不同场景，建议设置以下告警规则：

### 成本告警

```yaml
# 示例：Grafana 告警规则（伪代码）

告警1: 日费用超限
  条件: sum(increase(claude_code_cost_usage_total[24h])) > 20  # 每日 $20
  严重级别: Warning

告警2: 单会话费用异常
  条件: max(claude_code_cost_usage_total) by (session_id) > 5  # 单会话 $5
  严重级别: Critical
  说明: 可能存在死循环或失控调用

告警3: 每小时费用突增
  条件: rate(claude_code_cost_usage_total[1h]) > 2  # 每小时 $2
  严重级别: Warning
```

### 错误告警

```yaml
告警4: API 错误率过高
  条件: count(claude_code_api_error) / count(claude_code_api_request) > 0.1
  严重级别: Warning
  说明: 错误率超过 10% 需要排查

告警5: 连续 API 错误
  条件: count_over_time(claude_code_api_error[5m]) > 10
  严重级别: Critical
```

---

## 常用 PromQL 查询示例

> 以下查询适用于 Prometheus 或 VictoriaMetrics

### 费用分析

```promql
# 今日总费用（美元）
sum(increase(claude_code_cost_usage_total[24h]))

# 按用户分组的费用 Top 10
topk(10, sum by (user_account_uuid) (increase(claude_code_cost_usage_total[7d])))

# 按模型分组的费用占比
sum by (model) (increase(claude_code_cost_usage_total[7d]))
```

### Token 分析

```promql
# 缓存命中率（越高越省钱）
sum(increase(claude_code_token_usage_total{token_type="cache_read"}[7d]))
/
sum(increase(claude_code_token_usage_total{token_type=~"input|cache_creation"}[7d]))

# 按 token 类型的消耗对比
sum by (token_type) (increase(claude_code_token_usage_total[7d]))
```

### 生产力分析

```promql
# 每美元产出的代码行数（ROI 指标）
sum(increase(claude_code_lines_of_code_count_total[7d]))
/
sum(increase(claude_code_cost_usage_total[7d]))

# 每用户平均每次会话费用
sum by (user_account_uuid) (increase(claude_code_cost_usage_total[7d]))
/
sum by (user_account_uuid) (increase(claude_code_session_count_total[7d]))
```

### 使用趋势

```promql
# 每日活跃会话数
sum(increase(claude_code_session_count_total[1d]))

# 每日产生的 commit 数
sum(increase(claude_code_commit_count_total[1d]))
```

---

## 总结：重点指标优先级一览

| 优先级 | 指标 | 核心用途 |
|--------|------|---------|
| 🔴 P0 | `claude_code.cost.usage` | 成本控制，防止超支 |
| 🔴 P0 | `claude_code.token.usage` | Token 消耗分析，缓存优化 |
| 🟠 P1 | `claude_code.lines_of_code.count` | ROI 量化（每美元代码产出） |
| 🟠 P1 | `claude_code.commit.count` | 工程产出量度量 |
| 🟠 P1 | `claude_code.active_time.total` | 使用强度分析 |
| 🟠 P1 | `claude_code.session.count` | 使用频率与异常会话识别 |
| 🟡 P2 | `claude_code.pull_request.count` | 工程交付趋势 |
| 🟡 P2 | `claude_code.code_edit_tool.decision` | AI 建议采纳率分析 |

**建议先搭建以 P0 指标为核心的仪表盘，配置成本告警，再逐步完善 P1/P2 的生产力分析看板。**
