# OKR 对齐 Skill

一个多 agent 协同的 OKR 穿透对齐流程，将高层的组织 OKR 分解、分配给不同角色的 agent，并收集反馈形成执行计划。

## 核心机制

**穿透式分配**：不通过层层汇报，而是直接将 KR 分配给负责的 agent，收集其一轮反馈后快速对齐。

**agent-to-agent 通信**：使用 `sessions_send` 实现跨 session 通信，等待每个成员的 `reply`，而非依赖异步 announce。

## 角色体系

**注意**: 使用者可以根据自己的行业,职责设定来修改下面的agent列表. 

| 角色 | 标识 | Session Key | 职责 |
|------|------|-------------|------|
| 总监（Director） | 🎬 | 当前会话 | 统筹分配、收集反馈、风险汇总、向用户汇报 |
| 探索者（Scout） | 🔍 | agent:scout:main | 信息采集、竞品分析、趋势识别 |
| 创作者（Editor） | ✍️ | agent:editor:main | 内容生产、文案创作、发布运营 |
| 分析师（Analyst） | 📊 | agent:analyst:main | 数据分析、效果评估、指标追踪 |
| 技术专家（Technomancer） | 🔧 | agent:main:main | 技术攻坚、coding、skill 开发 |

## 执行流程

```
用户 OKR 输入
    ↓
Step 1: 读取并理解 OKR（总监分析）
    ↓
Step 2: 并行分配 KR（scout/editor/analyst）
    ↓
Step 3: 综合风险和需求（汇总反馈）
    ↓
Step 4: 确认分配（可选，如信心指数 < 6）
    ↓
Step 5: 向用户汇报（完整汇总）
```

## 输入格式

支持以下任一方式：
- 在线文档链接（Feishu/Google Doc/企业微信文档, 但需要你的agent有权限和工具读取）
- 文件附件（OKR 文档）
- 本地文件路径
- 直接文本描述

## 输出结构

最终汇报包含：

1. **每个成员的 OKR 分配**
2. **每个成员的反馈**
   - 通信状态（✅ 已回复 / ⏱️ 超时 / ❌ 失败）
   - 信心指数（0-10）
   - 负责的 KR
   - 需要的帮助
   - 潜在风险
3. **风险和需求汇总**（按 P0/P1/P2 优先级分类）
4. **初版支持方案**
   - 总盟能协调的资源
   - 需要用户提供的支持
   - 跨成员协作事项
5. **整体信心指数**（所有成员平均值）
6. **下一步行动**建议

## 通信机制

### 并行发送与等待

使用 `sessions_send` 并行发送给所有成员，设置 `timeoutSeconds=60` 等待回复：

```
Promise.all([
  sessions_send({ sessionKey: "agent:scout:main", message: "...", timeoutSeconds: 60 }),
  sessions_send({ sessionKey: "agent:editor:main", message: "...", timeoutSeconds: 60 }),
  sessions_send({ sessionKey: "agent:analyst:main", message: "...", timeoutSeconds: 60 })
])
```

直接读取返回的 `reply` 字段，**不依赖**异步 announce。

### 错误处理与重试

- **超时**（`status === "timeout"`）：等待 10 秒后自动重试 1 次
- **错误**（`status === "error"`）：等待 10 秒后自动重试 1 次
- **格式错误**（`reply` 非 JSON）：要求成员重新反馈

## 前置要求

- OpenClaw 配置：
  - `tools.agentToAgent.enabled = true`
  - allow 列表包含 `director/scout/editor/analyst`
  - `tools.sessions.visibility = "all"`

## 依赖文件

- `SKILL.md` - 完整执行流程和调试指南
- `references/feedback-template.md` - 通信模板
- `references/perspectives.md` - 角色视角定义（如果存在）

## 触发方式

在对话中提及：
- "OKR 对齐"
- "穿透"
- "分解/分配 OKR"
- "团队 OKR 对齐"

总监 agent 会自动启动对齐流程。
