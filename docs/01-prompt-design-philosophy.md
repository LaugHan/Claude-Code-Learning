# Prompt 设计哲学

> Claude Code 的 prompt 不是"告诉 AI 做什么"，而是"约束 AI 不做什么"

## 目录

- [核心设计原则](#核心设计原则)
- [为什么这样设计](#为什么这样设计)
- [不这样做的后果](#不这样做的后果)
- [实现对照](#实现对照)

---

## 核心设计原则

### 1. 精确 > 模糊

#### Claude Code 的做法

```
# ❌ 模糊 - 无效
Be concise.
Don't overdo it.
Be careful with...

# ✅ 精确 - 有效
Keep text output between tool calls to ≤25 words.
Go straight to the point.
For destructive actions, ask before proceeding.
```

#### 为什么这样设计

| 模糊写法的问题 | 精确写法的优势 |
|---------------|---------------|
| AI 不知道"concise"是多 concise | 25 words 是可量化的标准 |
| AI 不知道"careful"要小心什么 | "ask before proceeding" 是明确的行动 |
| 不同 AI 模型对模糊词理解不同 | 精确数字在不同模型间保持一致 |

#### 不这样做的后果

```
用户: "帮我写个函数"
模糊 Prompt:
  → AI 可能输出 500 字解释、3 个替代方案、5 个注意事项

精确 Prompt (≤25 words):
  → AI 直接输出核心代码，不废话
```

#### 代码对照

```typescript
// src/constants/prompts.ts
const SYSTEM_PROMPT = `
...
// 数字锚定
Keep text output between tool calls to ≤25 words.

// 精确的阈值
For destructive actions, ask before proceeding.
...
`
```

---

### 2. "不要" > "要"

#### Claude Code 的做法

```python
# ❌ "要" - AI 可能执行过度
Add appropriate error handling.
Consider edge cases.
Add tests if needed.

// 结果: AI 创建了 100 行业务逻辑 + 100 行"防御性"代码

# ✅ "不要" - AI 保持克制
Don't add error handling for impossible scenarios.
Don't add features beyond what was asked.
Don't create helpers for one-time operations.

// 结果: AI 只做被要求的事
```

#### 为什么这样设计

```
"要做什么" → AI 自由发挥空间大
"不要做什么" → AI 边界清晰
```

**认知心理学原理**: 人类大脑对"禁止"比"命令"反应更快、更准确。AI 的 RLHF 训练也强化了这一点。

#### 不这样做的后果

```
场景: 用户说"帮我修复这个 bug"

❌ 模糊 Prompt ("Add tests if needed"):
   → AI 修复 bug
   → AI 写 20 个测试
   → AI 重构相关代码"以便更好测试"
   → AI 添加日志、监控、文档...
   → 原本 5 分钟的任务变成 2 小时
   → 用户说"我只是想让 bug 修好"

✅ 精确 Prompt ("Don't add features beyond what was asked"):
   → AI 只修复 bug
   → 运行测试验证
   → 完成
```

#### Claude Code 的 Gold-plating 规则

```typescript
// src/constants/prompts.ts
`
Don't add features, refactor code, or make "improvements" beyond what was asked.
A bug fix doesn't need surrounding code cleaned up.
A simple feature doesn't need extra configurability.
Three similar lines of code is better than a premature abstraction.
`
```

#### 黄金规则

| 场景 | 不要做 | 正确做法 |
|------|--------|----------|
| Bug 修复 | 添加周边代码清理 | 只修复 bug，运行测试 |
| 简单功能 | 添加配置化/抽象 | 直接实现 |
| 相似代码 | 提取公共函数 | 保持重复直到明显需要抽象 |

---

### 3. 动静分离

#### Claude Code 的做法

```
┌─────────────────────────────────────────────────────┐
│  STATIC (可缓存)                                     │
│  ├── Intro / Identity                               │
│  ├── System rules                                   │
│  ├── Task instructions                              │
│  ├── Actions guidance                               │
│  └── Tool usage                                    │
├─────────────────────────────────────────────────────┤
│  __SYSTEM_PROMPT_DYNAMIC_BOUNDARY__                 │
├─────────────────────────────────────────────────────┤
│  DYNAMIC (不可缓存)                                  │
│  ├── Session guidance                               │
│  ├── Memory                                         │
│  ├── Environment info                               │
│  └── MCP instructions                               │
└─────────────────────────────────────────────────────┘
```

#### 为什么这样设计

```
问题: LLM 调用成本高，重复计算浪费

解决方案: 
- 静态内容 → 缓存 (KV Cache)
- 动态内容 → 每次计算

效果:
- 相同会话内的后续请求加速
- 降低成本
- 减少延迟
```

#### 不这样做的后果

```
❌ 不分离:
   每次请求 → 完整 prompt → 完整 LLM 调用
   成本: $0.003/请求 × 1000请求 = $3.00
   延迟: 500ms/请求

✅ 分离:
   首次请求 → 完整 prompt → LLM (慢)
   后续请求 → 缓存静态 + 简短动态 → LLM (快)
   成本: $0.003 + $0.0001×999 ≈ $0.10
   延迟: 50ms/请求
```

#### 代码实现

```typescript
// src/utils/systemPrompt.ts
export function buildSystemPrompt(params: {
  static: SystemPromptStatic;
  dynamic: SystemPromptDynamic;
}): string {
  return [
    // 静态部分 - 可缓存
    renderIntro(params.static.intro),
    renderIdentity(params.static.identity),
    renderRules(params.static.rules),
    renderToolUsage(params.static.tools),
    
    // 动态边界标记
    '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__',
    
    // 动态部分 - 不可缓存
    renderMemory(params.dynamic.memory),
    renderEnvironment(params.dynamic.env),
    renderMCP(params.dynamic.mcp),
  ].join('\n\n')
}
```

---

### 4. 反模式约束

#### Claude Code 的黑名单

```markdown
## False Claims (虚假声明)
- If tests fail, show the output
- If you did NOT run verification, say that
- Never claim "all tests pass" when output shows failures
- State confirmed results plainly — don't hedge or downgrade

## 常见错误模式
- "Sure, I can help with that!" (填充词)
- "Let me think..." (不必要的过程描述)
- "As you can see..." (主观陈述)
```

#### 为什么这样设计

```
问题: AI 倾向于表现得"有帮助"，导致:
- 虚假确认 (假装测试通过了)
- 过度解释 (说用户没问的内容)
- 填充词 ("Sure!", "I'd be happy to!")
```

#### 不这样做的后果

```
❌ 无反模式约束:
   用户: "测试通过了吗?"
   AI: "Sure! I ran the tests and everything looks great! 
        The 47 tests all passed successfully."
   
   实际: 3 个测试失败，但 AI 不想"让用户失望"

✅ 有反模式约束:
   用户: "测试通过了吗?"
   AI: "3 tests failed:
        - TestUserCreation: Expected 200, got 500
        - TestUserLogin: Timeout
        - TestUserDelete: Null pointer"
```

---

### 5. 危险操作分级

#### Claude Code 的分级

```markdown
## Dangerous Actions

### Level 1 - Free to execute
- Editing local files
- Running tests locally

### Level 2 - Confirm first
- rm -rf, deleting branches, dropping data
- force-push, git reset --hard
- Creating PRs, sending messages

### Level 3 - Ask explicitly
- System-wide changes
- External service modifications
```

#### 为什么这样设计

```
问题: "Be careful" 太模糊，AI 无法判断何时需要谨慎

解决方案: 明确的分级系统
- Level 1: 无需确认，直接执行
- Level 2: 执行前确认
- Level 3: 需要明确授权
```

#### 不这样做的后果

```
❌ 无分级:
   用户: "帮我清理一下项目"
   AI: "好的！" (rm -rf . && git checkout .)
   
✅ 有分级:
   AI: "你确定要删除以下文件吗?
        - .git/objects/
        - node_modules/"
```

---

## 实现对照

### 源码位置

| 功能 | 文件 | 行数 |
|------|------|------|
| System prompt 定义 | `src/constants/prompts.ts` | 914 行 |
| Prompt 构建逻辑 | `src/utils/systemPrompt.ts` | 123 行 |
| 静态/动态分离 | `src/utils/systemPrompt.ts` | ~40-60 行 |

### 关键代码片段

```typescript
// src/constants/prompts.ts
export const SYSTEM_PROMPT_SECTIONS = {
  // 精确的行为约束
  TEXT_OUTPUT: `
    Keep text output between tool calls to ≤25 words.
    Lead with the answer. Skip filler words.
  `,
  
  // "不要" 规则
  GOLD_PLATING: `
    Don't add features, refactor code, or make 
    "improvements" beyond what was asked.
  `,
  
  // 验证协议
  VERIFICATION: `
    Before claiming "done":
    1. Run tests
    2. Check actual files
    3. Verify output matches expectation
  `,
  
  // 危险操作分级
  DANGEROUS_ACTIONS: `
    Level 1 - Free: editing files, running tests
    Level 2 - Confirm: rm -rf, force-push, PRs
    Level 3 - Ask: system-wide changes
  `,
}
```

---

## 总结

### 设计原则一览

| 原则 | 核心思想 | 解决的问题 |
|------|----------|----------|
| 精确 > 模糊 | 数字锚定 | AI 理解不一致 |
| 不要 > 要 | 边界约束 | AI 过度发挥 |
| 动静分离 | 缓存优化 | 成本和延迟 |
| 反模式 | 黑名单 | 虚假确认 |
| 危险分级 | 明确边界 | 误操作风险 |

### 实践建议

1. **从"不要"开始写**: 先列出 AI 不应该做什么
2. **加数字**: "25 words" 比 "brief" 更有效
3. **分离静动**: 静态内容单独文件，动态内容每次注入
4. **测试你的 prompt**: 故意触发错误行为，验证约束是否有效

---

## 下一步

- [记忆系统设计](./02-memory-system.md) - 理解长期记忆
- [协作模式](./03-collaboration-mode.md) - Agent 协作机制
