# Claude Code 源码深度学习

> 深入剖析 Anthropic Claude Code 的源码，探索 AI 编程助手的最佳实践

## 📚 目录

### 核心系统

| 文档 | 内容 |
|------|------|
| [Prompt 设计哲学](./docs/01-prompt-design-philosophy.md) | 动静分离、数字锚定、反模式约束 |
| [记忆系统设计](./docs/02-memory-system.md) | Typed Memory、四层持久化、遗忘机制 |
| [协作模式](./docs/03-collaboration-mode.md) | Agent 协作、任务分配、跨会话通信 |
| [行为约束](./docs/04-behavior-constraints.md) | 危险操作分级、验证协议、False Claims 防护 |
| [性能优化](./docs/05-performance-optimization.md) | Prompt Cache、Token 控制、上下文管理 |

### 源码对照

| 文件 | 功能 |
|------|------|
| `src/constants/prompts.ts` | System Prompt 核心定义 |
| `src/utils/systemPrompt.ts` | Prompt 动态构建逻辑 |
| `src/memdir/memdir.ts` | Memory 系统实现 |
| `src/memdir/memoryTypes.ts` | 记忆类型定义 |
| `src/agent/loop.ts` | Agent 主循环 |

## 🎯 快速导航

### Prompt 设计核心原则

```
1. 精确 > 模糊
   ❌ "Be concise"
   ✅ "Keep text between tool calls ≤25 words"

2. 不要 > 要
   ❌ "Add error handling"
   ✅ "Don't add error handling for impossible scenarios"

3. 静 > 动
   静态内容 → 可缓存
   动态内容 → 运行时注入
```

### 记忆系统四类型

```
user       → 用户角色、偏好、知识
feedback   → 纠正和确认
project    → 项目上下文、决策
reference  → 外部系统指针
```

## 🚀 开始学习

```bash
git clone https://github.com/LaugHan/Claude-Code-Learning
cd Claude-Code-Learning
```

## 📖 阅读顺序

1. 先读 [Prompt 设计哲学](./docs/01-prompt-design-philosophy.md) - 理解核心思路
2. 再读 [记忆系统设计](./docs/02-memory-system.md) - 理解长期记忆
3. 最后按需阅读其他文档

## 🤝 贡献

欢迎提交 Issue 和 PR！

## 📄 License

MIT
