# 性能优化

> Claude Code 在架构层面就考虑了性能，而不只是"优化 prompt"

## 目录

- [Prompt 缓存](#prompt-缓存)
- [Token 控制](#token-控制)
- [上下文管理](#上下文管理)
- [实现细节](#实现细节)

---

## Prompt 缓存

### 核心问题

```
问题: 每次 LLM 调用都要重新计算完整 prompt

场景: 用户发了 50 条消息
- 传统做法: 50 次完整 prompt 计算
- Claude Code: 1 次完整 + 49 次增量

成本差异: 50x → 几乎 1x
```

### Claude Code 的做法

```
┌─────────────────────────────────────────┐
│  STATIC (缓存)                           │
│  ├── System rules (不变)                │
│  ├── Tool definitions (不变)             │
│  └── Identity (不变)                    │
├─────────────────────────────────────────┤
│  __SYSTEM_PROMPT_DYNAMIC_BOUNDARY__     │
├─────────────────────────────────────────┤
│  DYNAMIC (每次计算)                      │
│  ├── Memory (可能变)                     │
│  ├── Session context (变化)             │
│  └── Recent messages (变化)              │
└─────────────────────────────────────────┘
```

### 为什么这样设计

```
1. LLM API 定价按 token 数
2. 静态内容每次重复计算浪费
3. KV Cache 是 LLM 的核心优化

Claude Code 利用 API 的缓存:
- 静态内容 → 命中 Cache
- 动态内容 → 每次计算
```

### 不这样做的后果

```
❌ 不分离:
   每次请求: 2000 tokens (完整 prompt)
   50 请求: 100,000 tokens
   成本: $0.10

✅ 分离:
   首次请求: 2000 tokens
   后续请求: 500 tokens (仅动态)
   50 请求: 2,000 + 49×500 = 26,500 tokens
   成本: $0.026

节省: 73%
```

### 代码实现

```typescript
// src/utils/systemPrompt.ts
export function buildSystemPrompt(params: {
  static: SystemPromptStatic;
  dynamic: SystemPromptDynamic;
}): string {
  const staticPart = [
    renderIdentity(params.static),
    renderRules(params.static),
    renderTools(params.static),
  ].join('\n\n')
  
  const dynamicPart = [
    '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__',
    renderMemory(params.dynamic),
    renderContext(params.dynamic),
  ].join('\n\n')
  
  return staticPart + '\n\n' + dynamicPart
}

// API 调用时利用缓存
await llm.complete({
  prompt: buildSystemPrompt(...),
  cacheControl: {
    // API 自动识别可缓存的前缀
  }
})
```

---

## Token 控制

### 核心问题

```
问题: LLM 有上下文窗口限制 (如 200K tokens)

场景: 长会话
- 历史消息累积
- 最终超出限制
- 无法继续对话
```

### Claude Code 的策略

```markdown
## Token Management

1. Context Window: 保留一半给输出
   - 200K window → 只用 100K 输入

2. 主动压缩:
   - 检测 token 接近限制
   - 触发记忆压缩
   - 归档旧消息

3. 记忆系统:
   - 关键信息 → MEMORY.md
   - 历史消息 → 可被遗忘
```

### 为什么这样设计

```
原因:
1. 输出也需要空间
2. 模型在满上下文时性能下降
3. 保留 buffer 避免截断
```

### 实现: 消息压缩

```typescript
// 伪代码 - 消息压缩逻辑
class TokenManager {
  async compressIfNeeded(messages: Message[]) {
    const tokens = await countTokens(messages)
    const limit = this.contextWindow / 2
    
    if (tokens < limit) return
    
    // 1. 选择要压缩的消息
    const toCompress = this.selectBoundary(messages)
    
    // 2. 调用 LLM 摘要
    const summary = await llm.summarize(toCompress)
    
    // 3. 替换为摘要
    messages.replace(toCompress, summary)
    
    // 4. 保存到记忆
    await memory.archive(toCompress)
  }
}
```

### 不这样做的后果

```
❌ 无 Token 控制:
   200 条消息后 → 超出限制
   错误: "Context length exceeded"
   用户: [被迫开新会话]
   上下文: [丢失]

✅ 有 Token 控制:
   接近限制时 → 压缩旧消息
   MEMORY.md → 保存关键信息
   用户: [继续正常对话]
   上下文: [平滑过渡]
```

---

## 上下文管理

### 分层存储

```
┌─────────────────────────────────────┐
│  Layer 1: Working Context           │ ← 当前对话
│  - Recent messages                  │   (自动管理)
│  - Active files                     │
├─────────────────────────────────────┤
│  Layer 2: Session Memory            │ ← 当前会话
│  - Session history                  │   (可搜索)
│  - Recent decisions                 │
├─────────────────────────────────────┤
│  Layer 3: Persistent Memory          │ ← 跨会话
│  - MEMORY.md                        │   (长期)
│  - Project memories                 │
├─────────────────────────────────────┤
│  Layer 4: External Knowledge        │ ← 项目外
│  - Code files                       │   (只读)
│  - Documentation                    │
└─────────────────────────────────────┘
```

### 每层用途

| 层级 | 内容 | 生命周期 | 用途 |
|------|------|----------|------|
| L1 | 最近消息 | 当前对话 | 立即上下文 |
| L2 | 会话历史 | 当前会话 | 搜索回溯 |
| L3 | MEMORY.md | 永久 | 跨会话 |
| L4 | 代码/文档 | 项目生命周期 | 参考 |

### Claude Code 的上下文注入

```markdown
## Context Injection

### 自动注入
- Current directory structure
- Recent git changes
- Active file contents

### 按需注入
- 特定文件: 用户指定
- 搜索结果: AI 决定
- 记忆内容: AI 决定
```

---

## 实现细节

### 源码对照

| 功能 | 文件 |
|------|------|
| Prompt 构建 | `src/utils/systemPrompt.ts` |
| Token 计算 | `src/utils/tokenCounter.ts` |
| 记忆压缩 | `src/memdir/memdir.ts` |
| 上下文管理 | `src/context/` |

### 关键代码

```typescript
// Token 预算管理
const CONTEXT_BUDGET = {
  // Claude 200K 模型
  maxInput: 100_000, // 保留一半给输出
  
  // Claude 100K 模型  
  maxInput100k: 45_000,
  
  // Claude Sonnet
  maxInputSonnet: 80_000,
}

// 动态边界标记
const DYNAMIC_BOUNDARY = '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'

// 压缩触发条件
const COMPRESSION_TRIGGER = 0.8 // 80% 时触发
```

---

## 实践建议

### 1. 设计 Prompt 时

```
✅ 做:
- 分离静态和动态内容
- 静态内容越少越好
- 动态内容放在末尾

❌ 不做:
- 每次重复相同内容
- 把所有规则都加到 prompt
- 依赖长 prompt 解决问题
```

### 2. Token 预算

```
建议:
- 预算 = 上下文窗口 × 0.4 (输入)
- 保留 60% 给输出和 buffer

示例 (200K 模型):
- 输入预算: 80K tokens
- Buffer: 20K tokens
- 实际可用: 60K tokens
```

### 3. 记忆系统

```
何时保存到 MEMORY:
- 信息跨会话需要
- 无法从代码推导
- 用户明确要求记住

何时不保存:
- 当前对话的临时状态
- 可以从代码/日志推导
- 过时或可能被替代
```

---

## 总结

### 优化策略对比

| 策略 | 效果 | 复杂度 |
|------|------|--------|
| Prompt 缓存 | 50%+ 成本降低 | 中 |
| Token 控制 | 避免上限错误 | 高 |
| 分层上下文 | 保持响应质量 | 中 |
| 记忆系统 | 跨会话连续性 | 中 |

### 核心原则

```
1. 静态内容最小化
2. 动态内容放在末尾
3. 保留 50%+ buffer
4. 主动压缩而非被动报错
5. 关键信息存入记忆
```

---

## 相关文档

- [Prompt 设计哲学](./01-prompt-design-philosophy.md) - 动静分离详解
- [记忆系统设计](./02-memory-system.md) - 分层存储详解
