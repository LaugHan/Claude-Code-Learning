# 记忆系统设计

> Claude Code 的记忆不是"随便记"，而是"分类记忆 + 结构化存储"

## 目录

- [设计理念](#设计理念)
- [四层记忆类型](#四层记忆类型)
- [存储结构](#存储结构)
- [为什么这样设计](#为什么这样设计)
- [不这样做的后果](#不这样做的后果)
- [实现细节](#实现细节)

---

## 设计理念

### 核心问题

```
问题: AI 应该记住什么？

❌ 错误答案: "记住所有事情"
→ 记忆爆炸，无法管理
→ 过时信息污染上下文

✅ 正确思路: "只记不可推导的事"
→ 记忆是补充，不是副本
→ 定期清理，避免过时
```

### Claude Code 的原则

```
可从代码推导的 → 不记录
  - 代码模式 (grep 可以找到)
  - 架构设计 (README 可以找到)
  - Git 历史 (git log 可以找到)

不可推导的 → 必须记录
  - 用户偏好 ("用户喜欢用 Python 不喜欢 Go")
  - 团队规则 ("我们不用 mock 数据库")
  - 项目上下文 ("Q2 要上线移动端")
```

---

## 四层记忆类型

### 概览

| 类型 | 用途 | 何时保存 | 示例 |
|------|------|----------|------|
| **user** | 用户画像 | 了解用户角色时 | "用户是 10 年 Go 开发者，首次接触 React" |
| **feedback** | 行为指导 | 纠正或确认时 | "不要在结尾总结，用户可以看 diff" |
| **project** | 项目上下文 | 了解项目动态时 | "3/5 开始 merge freeze" |
| **reference** | 外部指针 | 了解外部系统时 | "Bug 在 Linear 项目 INGEST 追踪" |

### 详细设计

#### 1. user - 用户类型

```markdown
## 保存时机
当了解用户的角色、目标、职责、知识时

## 使用时机
当需要"以用户视角"回答时

## 示例
user: "我是数据科学家，研究日志系统"
→ 保存: user is a data scientist, focused on observability/logging

user: "我写 Go 10 年了，但第一次碰 React"
→ 保存: deep Go expertise, new to React — frame frontend in backend terms
```

#### 2. feedback - 反馈类型

```markdown
## 保存时机
- 用户纠正时 ("不要那样", "停止做 X")
- 用户确认时 ("对，就是这样", "完美")

## 重要: 同时记录成功和失败
- 只记失败 → AI 变得过于谨慎
- 只记成功 → AI 不学习错误

## 结构要求
1. 规则本身
2. **Why:** 原因 (为什么用户这么说)
3. **How to apply:** 何时应用

## 示例
user: "不要 mock 数据库，之前出过事故"
→ 保存: integration tests must hit real DB, not mocks
     Why: mock 测试通过但 prod 迁移失败
     How to apply: 所有涉及数据库的测试

user: "对，单个 PR 合并是对的，拆开会浪费时间"
→ 保存: for refactors in this area, user prefers bundled PR
     Why: 拆分只是 churn
     How to apply: 重构类任务
```

#### 3. project - 项目类型

```markdown
## 保存时机
当了解项目动态、目标、限制时

## 重要: 转换相对时间为绝对时间
- "Thursday" → "2026-03-05"
- "下周" → "2026-W14"
- 原因: 记忆会被未来阅读，需要绝对时间

## 结构要求
1. 事实/决策
2. **Why:** 动机
3. **How to apply:** 如何影响建议

## 示例
user: "周四之后冻结非关键合并，mobile 要切分支"
→ 保存: merge freeze starts 2026-03-05 for mobile release
     Why: mobile team cutting release branch
     How to apply: flag non-critical PRs after that date

user: "重写 auth 中间件是因为法务要求"
→ 保存: auth rewrite driven by compliance, not tech-debt
     Why: legal flagged session token storage
     How to apply: favor compliance over ergonomics
```

#### 4. reference - 引用类型

```markdown
## 保存时机
当了解外部系统位置和用途时

## 结构
[系统名] - [位置/链接] - [用途]

## 示例
user: "Linear 项目 INGEST 追踪所有 pipeline bug"
→ 保存: pipeline bugs → Linear project "INGEST"

user: "Grafana 看板 grafana.internal/d/api-latency 是 oncall 看的"
→ 保存: oncall latency dashboard → grafana.internal/d/api-latency
```

---

## 存储结构

### 目录结构

```
~/.claude/memory/
├── MEMORY.md              # 索引文件 (入口)
├── user_profile.md        # 用户信息
├── user_experience.md     # 用户经验
├── feedback_communication.md
├── feedback_testing.md
├── project_current.md
├── project_deadlines.md
├── ref_linear.md
└── ref_grafana.md
```

### 索引文件 (MEMORY.md)

```markdown
# Auto Memory

[Index - keep concise]
- [User: data scientist, logging focus](user_profile.md)
- [Feedback: real DB in tests, no mocks](feedback_testing.md)
- [Project: mobile release 2026-03-05](project_current.md)
- [Ref: Linear INGEST for pipeline bugs](ref_linear.md)
```

### 记忆文件格式

```markdown
---
name: User Profile - Data Scientist
description: User is investigating logging infrastructure
type: user
created: 2026-03-15
updated: 2026-03-31
---

# User: Data Scientist Focus

## Role
Data scientist investigating logging/observability

## Knowledge
- Deep expertise in data analysis
- Learning about production logging

## Communication Preferences
- Technical but accessible
- Show concrete examples
```

---

## 为什么这样设计

### 1. 分类防止记忆混乱

```
❌ 无分类:
   MEMORY.md:
   - 用户是数据科学家
   - 不要 mock 数据库
   - Q2 要上线移动端
   - Linear 追踪 bug
   
   问题: 难以搜索、可能冲突、不知道类型

✅ 有分类:
   user:    用户是数据科学家
   feedback: 不要 mock 数据库
   project: Q2 上线移动端
   ref:     Linear 追踪 bug
   
   问题: 清晰、可独立更新、不冲突
```

### 2. 独立文件支持细粒度管理

```
单个 MEMORY.md 的问题:
- 文件变大 (10KB → 100KB)
- 无法版本控制 (每次改一小点)
- 无法只读某类记忆

独立文件的优势:
- 每个文件 < 5KB
- 可以 git 追踪变化
- 可以只加载需要的类型
```

### 3. 遗忘机制

```markdown
## Claude Code 支持"遗忘"

用户说 "forget X" → AI 找到相关记忆 → 删除

原因: 记忆可能过时或错误
- 用户换了角色
- 团队规则变了
- 项目终止了
```

---

## 不这样做的后果

### 场景: 无结构记忆

```
用户 A 项目用了某规则
用户 B 项目继承了 A 的规则 ✗
用户 C 项目完全无关但也有这条 ✗

结果: 记忆污染，AI 在错误项目用错误规则
```

### 场景: 无遗忘机制

```
2024年: 用户在 X 公司
2025年: 用户换到 Y 公司
2026年: AI 仍然说"记得你在 X 公司..."

结果: AI 信息过时，不可靠
```

### 场景: 记录可推导信息

```
AI 记住: "项目使用 React 框架"
用户代码: "import React from 'react'"

问题:
- 记忆是冗余的
- 代码改了，记忆不会自动更新
- 误导: AI 认为 React 是"事实"而非代码
```

---

## 实现细节

### 源码对照

| 功能 | 文件 |
|------|------|
| 记忆类型定义 | `src/memdir/memoryTypes.ts` |
| 记忆读写 | `src/memdir/memdir.ts` |
| 记忆搜索 | `src/memdir/findRelevantMemories.ts` |
| 记忆过期 | `src/memdir/memoryAge.ts` |

### 关键代码

```typescript
// src/memdir/memoryTypes.ts
export const MEMORY_TYPES = [
  'user',      // 用户信息
  'feedback',  // 行为指导
  'project',   // 项目上下文
  'reference', // 外部指针
] as const

// 记忆文件 frontmatter
interface MemoryFrontmatter {
  name: string
  description: string
  type: MemoryType
  created: string // ISO date
  updated: string // ISO date
}
```

```typescript
// src/memdir/memdir.ts
export function buildMemoryPrompt(params: {
  memoryDir: string
}): string {
  const entrypoint = memoryDir + 'MEMORY.md'
  const index = readFile(entrypoint)
  
  return [
    '# Auto Memory',
    renderMemoryInstructions(),
    '# Index',
    truncate(index, { maxLines: 200 }), // 限制 200 行
  ].join('\n\n')
}
```

### KAIROS 模式 (长会话)

对于极长的会话 (Assistant 模式)，Claude Code 使用不同的策略:

```
普通模式: 直接写入 MEMORY.md
KAIROS模式: 每日追加到日志，定期蒸馏

目录结构:
memory/
├── logs/
│   └── 2026/
│       └── 03/
│           └── 2026-03-31.md  # 今天的记忆
└── MEMORY.md                   # 蒸馏后的索引
```

**夜间蒸馏**: 每天结束时，AI 阅读日志文件，提取关键信息到 MEMORY.md。

---

## 实践建议

### 1. 开始新项目时

```bash
mkdir -p memory
touch memory/MEMORY.md
```

### 2. 保存记忆的时机

| 触发 | 保存 |
|------|------|
| 用户说 "I'm a..." | user 类型 |
| 用户说 "don't", "stop" | feedback 类型 |
| 用户说 "yes exactly" | feedback 类型 |
| 用户说 "we're planning to..." | project 类型 |
| 用户说 "check X for..." | reference 类型 |

### 3. 记忆文件命名

```
user_*.md        # 用户相关
feedback_*.md    # 反馈相关
project_*.md     # 项目相关
ref_*.md         # 引用相关
```

### 4. 定期清理

```
每月检查:
□ 过时的项目记忆是否删除?
□ 重复的记忆是否合并?
□ MEMORY.md 索引是否超 200 行?
```

---

## 总结

### 设计对比

| 方面 | 无结构记忆 | Claude Code 风格 |
|------|-----------|-----------------|
| 存储 | 单文件 | 分类 + 索引 |
| 查找 | 全文搜索 | 按类型筛选 |
| 更新 | 全文修改 | 单文件更新 |
| 遗忘 | 不支持 | 支持删除 |
| 过时 | 累积 | 可定期清理 |

### 核心原则

```
1. 只记不可推导的
2. 分类存储
3. 支持遗忘
4. 定期清理
```

---

## 下一步

- [协作模式](./03-collaboration-mode.md) - Agent 协作机制
- [行为约束](./04-behavior-constraints.md) - 危险操作和验证
