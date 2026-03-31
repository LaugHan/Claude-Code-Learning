# 协作模式

> Claude Code 的 AI 不是"执行者"，而是"协作者"

## 目录

- [协作理念](#协作理念)
- [主动 vs 被动](#主动-vs-被动)
- [任务分配](#任务分配)
- [跨会话协作](#跨会话协作)
- [实现细节](#实现细节)

---

## 协作理念

### 核心问题

```
❌ 传统 AI 助手:
   用户 → AI (执行命令)
   
   问题: AI 只是被动响应，不主动思考

✅ Claude Code:
   用户 ←→ AI (双向协作)
   
   特点: AI 会指出误解、提出建议、主动发现问题
```

### Claude Code 的协作原则

```markdown
## Collaboration

If you notice the user's request is based on a misconception,
or spot a bug adjacent to what they asked about, say so.

You're a collaborator, not just an executor.
```

---

## 主动 vs 被动

### 被动模式的问题

```
❌ 被动 AI:
   用户: "帮我写个函数"
   AI: [写函数]
   用户: "它有 bug"
   AI: [修 bug]
   用户: "还有其他问题"
   AI: [继续修]
   ...
   
   问题: 用户被迫当 QA，效率低
```

### 主动模式的优势

```
✅ 主动 AI:
   用户: "帮我写个函数"
   AI: [写函数]
   AI: "注意：这个实现在并发场景下可能有 race condition，
        需要加锁。如果你需要处理并发，我可以加上。"
   
   用户: [选择是否需要]
   
   优势: AI 主动发现问题，用户做决策
```

### Claude Code 的主动行为

```markdown
## Proactive Behavior

If you have nothing useful to do → SLEEP
If tick arrives and you're idle → SLEEP immediately
Never respond with only "still waiting"
```

---

## 任务分配

### Claude Code 的标签系统

```markdown
## Task Distribution

- @research → 后台研究任务
- @writer → 写作任务
- @coder → 编码任务

## Usage
Use `subagent` to run tasks in background.
Use `sessions_send` for cross-session communication.
```

### 实际使用

```
用户: "@research 帮我调研 RAG 技术的发展趋势"
AI: [创建 research subagent]
   → 后台运行
   → 用户可以继续其他任务
   → 完成后通知用户

用户: "@coder 帮我实现这个功能"
AI: [创建 coder subagent]
   → 专注于编码
   → 可以并行 @research
```

### 并行 vs 串行

```markdown
## Parallel Execution

Complex tasks should be decomposed for parallel execution:

Task: "调研 + 实现 RAG 功能"

❌ 串行:
   1. 调研 RAG
   2. 实现 RAG
   总时间: T1 + T2

✅ 并行:
   @research: 调研 RAG
   @coder: 准备项目结构
   完成后: 合并实现
   总时间: max(T1, T2)
```

---

## 跨会话协作

### 问题

```
用户在不同时间、不同会话和 AI 交互

传统 AI:
- 每个会话从零开始
- 用户需要重复解释上下文

Claude Code:
- 记忆系统保持上下文
- 跨会话传递信息
```

### 解决方案

```markdown
## Memory + Sessions

### 短期 (当前会话)
- Session messages (自动保存)
- 上下文窗口内

### 中期 (跨会话)
- MEMORY.md (长期记忆)
- 项目级记忆

### 长期 (用户偏好)
- USER memory type
- FEEDBACK memory type
```

### 信息传递机制

```markdown
## Cross-Session Communication

1. MEMORY.md → 通用信息
   - 用户偏好
   - 项目规则

2. Project memory → 项目特定信息
   - 当前项目状态
   - 进行中的任务

3. Session history → 近期对话
   - 可以搜索
   - 自动过期
```

---

## 实现细节

### 源码对照

| 功能 | 文件 |
|------|------|
| Agent 定义 | `src/agent/` |
| Subagent | `src/agent/subagent.ts` |
| Session 管理 | `src/session/` |

### Subagent 模式

```typescript
// 伪代码 - subagent 概念
interface Subagent {
  name: string
  task: string
  status: 'pending' | 'running' | 'done'
  result?: string
}

async function createSubagent(task: string): Promise<Subagent> {
  // 创建后台任务
  // 返回 subagent 引用
}
```

### 协作触发器

```markdown
## 触发协作的信号

### 用户明确要求
- "@research", "@writer", "@coder"
- "run in background"
- "parallel"

### AI 主动建议
- "这个任务很大，我可以后台运行"
- "我们可以并行处理这几个部分"

### 复杂任务检测
- 任务涉及多个领域
- 任务可以分解为独立子任务
- 任务耗时预计较长
```

---

## 实践建议

### 1. 何时主动

| 情况 | 主动行为 |
|------|---------|
| 发现明显 bug | 指出并建议修复 |
| 发现误解 | 澄清而非顺从 |
| 任务可分解 | 建议并行 |
| 缺少信息 | 询问而非猜测 |

### 2. 何时被动

| 情况 | 被动行为 |
|------|---------|
| 用户明确指令 | 直接执行 |
| 任务很简单 | 不需要讨论 |
| 用户很忙 | 简洁回复 |

### 3. 协作提示词示例

```markdown
## 如果要实现协作模式

### System Prompt

You are a collaborator, not just an executor.

If you notice:
- The user's request is based on a misconception → say so
- There's a bug adjacent to what they asked about → flag it
- The task can be parallelized → suggest it

But also:
- Don't over-explain simple tasks
- Don't suggest improvements the user didn't ask for
- Don't be patronizing
```

---

## 总结

### 协作 vs 执行

| 方面 | 纯执行者 | 协作者 |
|------|---------|--------|
| 误解处理 | 顺从执行 | 指出并澄清 |
| 问题发现 | 等用户说 | 主动发现 |
| 任务分解 | 串行执行 | 建议并行 |
| 信息缺失 | 盲目猜测 | 主动询问 |

### 核心原则

```
1. 主动但不烦人
2. 协作者而非仆人
3. 发现问题及时说
4. 任务可分解就并行
```

---

## 下一步

- [行为约束](./04-behavior-constraints.md) - 危险操作和验证
- [性能优化](./05-performance-optimization.md) - Prompt 缓存和 Token 控制
