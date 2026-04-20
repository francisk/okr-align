---
name: okr-align
description: OKR 穿透对齐流程。当用户要求分解/分配 OKR、进行团队 OKR 对齐、或提及"穿透"时激活。执行者是 director（总监），通过 agent-to-agent 通信将 OKR 分配给 scout/editor/analyst 并收集反馈，最终向用户汇报。
---

# OKR 穿透对齐

## 角色与执行者

- **执行者**: director（总监🎬）
- **参与者**: scout 🔍、editor ✍️、analyst 📊、technomancer
- **汇报对象**: 用户
- **通信方式**: agent-to-agent（`sessions_send`），**不是 sub-agent**

## 前置条件

- `tools.agentToAgent.enabled = true`，allow 列表包含 director/scout/editor/analyst
- `tools.sessions.visibility = "all"`

## 执行流程

### Step 1: 读取并分析用户的 OKR

**输入**: 用户通过对话提供的 OKR 文档（在线文档链接 / 文件附件 / 文件路径 / 直接文本）

**操作**:
1. 读取并理解用户的 OKR（Objective + Key Results）
2. 理解业务背景和目标意图
3. 如对用户的 KR,业务背景,目标,核心指标有不清楚的地方，通过消息向用户提问
4. 向用户展示自己对 OKR 的整体理解

**输出**: 对用户 OKR 的完整理解

**失败处理**: 如果文档无法读取或内容不明确，向用户说明并请求补充

### Step 2: 并行分配 OKR（策略 A：直接读取 reply）

**输入**: 用户提供的 OKR + Step 1 的业务背景

**操作**:
1. 根据每个成员的职责领域，分配对应的 KR：
   - **scout**: 信息采集、竞品分析、趋势识别相关
   - **editor**: 内容生产、文案创作相关
   - **analyst**: 多平台发布, 效果评估、指标追踪相关
   - **technomancer**: 团队的技术核心,精通skill,编程,MCP 工具的开发和使用.
2. 对每个成员，准备分配消息，包含：
   - 继承的 O1
   - 该成员负责的具体 KR
   - 相关业务背景说明
   - 简化反馈指令（只要求：信心、KR、需要的帮助、风险）
3. **并行发送并等待（Strategy A）**: 使用 `Promise.all` 并行发送给所有成员，目标 sessionKey 分别为：
   - `agent:scout:main`
   - `agent:editor:main`
   - `agent:analyst:main`
   
   **关键参数**：`timeoutSeconds=60`，这样会等待每个成员返回回复

4. **收集返回结果**: 每个 `sessions_send` 返回的 `reply` 字段就是成员的完整反馈：
   - `result.reply` 包含成员的回复内容（JSON 格式）
   - `result.status` 为 `"ok"` 表示成功，`"timeout"` 表示超时，`"error"` 表示错误
   - **不需要**轮询 `sessions_history`，**不需要**等待 announce

**简化反馈指令**（第一轮）：
```
请针对分配给你的 KR，快速反馈以下内容（JSON格式）：
{
  "confidence": 0-10 的信心指数,
  "my_krs": "你负责的 KR 列表",
  "help_needed": "需要我（总监）提供什么支持",
  "risks": "你看到的潜在风险或不确定性"
}

注意：暂时不需要检查工具和 SKILL，快速回复即可。
```

**输出**: 三个成员的返回结果，每个包含：
- `sessionKey`: 成员的 session key
- `status`: "ok"/"timeout"/"error"
- `reply`: 成员的反馈内容（status="ok" 时）或错误信息（status="error" 时）

### Step 3: 综合风险和需求（形成初版）

**输入**: Step 2 收集的三个成员反馈

**操作**:
1. 汇总所有成员的 `help_needed`（需要的帮助）
2. 汇总所有成员的 `risks`（潜在风险）
3. 识别共性问题和跨成员依赖
4. 按优先级分类：
   - **P0（阻塞性）**: 必须立即解决，否则无法推进
   - **P1（高优先级）**: 影响进度或质量
   - **P2（中优先级）**: 可优化但不阻塞
5. 生成**初版支持方案**：
   - 我（director agent）能直接协调的资源
   - 需要用户提供的支持
   - 需要成员之间协作的事项
6. 计算整体信心指数：三个成员信心值的平均值

**输出**: 风险和需求汇总（按优先级分类）+ 初版支持方案

### Step 4: OKR 确认（可选）

**输入**: Step 3 的风险和需求汇总

**操作**:
1. 如果整体信心指数 ≥ 6，可以直接进入执行（跳过此步）
2. 如果整体信心指数 < 6，或用户要求，进行确认：
   - 向每个成员发送确认消息（`sessions_send`，timeoutSeconds=60）,询问成员具体对KR 里的哪一个描述信心不足.
   - 收集成员对分配和风险应对方案的确认意见
3. 如果有成员提出异议，调整分配或风险应对方案,KR 可以拆分,每个成员可以将不属于自己职责的 KR 部分交给更擅长的人员,如果无人适合,请先记在director的职责上.并标记为职责缺失的风险.

**输出**: 所有成员的确认状态 + 最终OKR分配

### Step 5: 向用户汇报（必须返回）

**输入**: 完整的 OKR 分配 + 所有反馈 + 风险和需求汇总

**操作**: 生成结构化汇报并**返回给用户**，包含：
1. **每个成员的 OKR 分配**（KR + 业务背景）
2. **每个成员的反馈**（第一轮简化反馈）：
   - 通信状态：✅ 已回复 / ⏱️ 超时 / ❌ 失败
   - 信心指数（0-10）
   - 负责的 KR
   - 需要的帮助
   - 潜在风险
3. **风险和需求汇总**（按优先级分类）：
   - **P0（阻塞性）**: 必须立即解决
   - **P1（高优先级）**: 影响进度
   - **P2（中优先级）**: 优化项
4. **初版支持方案**：
   - 总盟能协调的资源
   - 需要用户提供的支持
   - 跨成员协作事项
5. **整体信心指数**（三个成员的平均值）
6. **下一步行动**：是否需要深度检查工具和 SKILL（可选）

**输出**: **向用户发送完整汇报（这是 skill 的最终输出，必须完整返回）**

## 成员职责映射

| 成员 | Session Key | 职责领域 | 分配 KR 类型 |
|------|-------------|----------|-------------|
| scout 🔍 | agent:scout:main | 信息采集、竞品分析、趋势识别 | 采集类、调研类 |
| editor ✍️ | agent:editor:main | 内容生产、文案创作、发布运营 | 创作类、运营类 |
| analyst 📊 | agent:analyst:main | 数据分析、效果评估、指标追踪 | 分析类、指标类 |
| technomancer 🔧 | agent:main:main | 技术攻坚、coding、skill开发 | 技术开发类 |

**注意**：所有默认channel和当前会话channel可能不一致。这会导致 `delivery.announce` 跨 channel 时可能失败，因此请**直接读取 `reply` 字段**，不依赖 announce 机制。

## 通信稳定性保障

### 直接读取 reply

**Step 2（并行分配）**:
- **发送方式**: 使用 `Promise.all` 并行发送到所有成员
- **Timeout 设置**: `timeoutSeconds=60`，等待每个成员返回
- **返回读取**: 直接读取返回的 `reply` 字段，**不需要**轮询 `sessions_history`
- **announce**: 仍然会异步执行，但**不依赖**它获取数据

**Step 4（确认阶段）**:
- **Timeout**: 60秒（确认消息处理）
- **返回读取**: 直接读取 `reply` 字段


### 错误处理和重试机制

**单次发送后**：
1. 如果 `result.status === "ok"`：直接读取 `reply`，标记为成功
2. 如果 `result.status === "timeout"`：
   - 等待 **10 秒**（让成员 session 恢复空闲）
   - **自动重试发送**（相同参数）
   - 如果重试成功，读取 `reply`
   - 如果重试仍然 timeout，标记为超时
3. 如果 `result.status === "error"`：
   - 等待 **10 秒**
   - **自动重试发送**（相同参数）
   - 如果重试仍然 error，记录错误信息
4. 如果 `result.reply` 格式错误：
   - 通过 `sessions_send` 要求重新反馈（`timeoutSeconds=60`）
   - 不再自动重试（格式错误重试无意义）

**重试策略**：
- 每个成员最多重试 **1 次**（timeout 或 error 后）
- 重试前等待 **10 秒**
- 如果重试仍然失败，标记为超时/错误，不无限重试
- 重试后成功，视为正常处理

### 状态记录
- 记录每个成员的状态："已发送" → "收到反馈" / "超时" / "错误"
- 记录每次 `sessions_send` 的返回值（包括重试）
- 记录是否进行了重试
- 最终汇报包含完整的通信状态日志

## 通信模板

分配消息和反馈模板见 `{baseDir}/references/feedback-template.md`

## 调试指南

### 常见问题诊断

**问题1: 某个成员超时（重试后仍然超时）**
- 现象：`result.status === "timeout"`（包括重试后）
- 诊断步骤：
  1. 检查成员session是否活跃：`sessions_list`
  2. 检查成员是否有其他任务占用
- 解决方案：
  - 如果session不活跃，等待其恢复或手动激活
  - 如果被其他任务占用，建议等待或终止该任务
  - 在最终汇报中标注该成员超时（已重试 1 次）
- **注意**：Skill 会自动重试 1 次，无需手动重试

**问题2: 返回的 reply 格式错误**
- 现象：`result.reply` 不是有效的 JSON 或缺少字段
- 诊断步骤：
  1. 检查返回的 `reply` 字段内容
  2. 检查是否包含所有必需字段（confidence, my_krs, help_needed, risks）
- 解决方案：
  - 通过 `sessions_send` 要求成员重新反馈（`timeoutSeconds=60`）
  - 如果重复格式错误，在最终汇报中标注

**问题3: 用户没有收到汇总汇报**
- 现象：我已汇总并向你汇报，但你没收到
- 原因：`delivery.announce` 机制失败或超时
- 解决方案：
  - 我已经直接读取了 `reply` 字段并生成汇报
  - 不依赖 `delivery.announce` 机制
  - 如果问题持续，建议检查 Telegram channel 配置


### 日志记录

建议在执行过程中记录以下日志：
1. 每次sessions_send的时间戳、sessionKey、timeoutSeconds
2. 每个成员的返回值（status、reply、runId）
3. 最终的通信状态汇总（成功/超时/错误）
