# 行为约束与验证协议

> Claude Code 通过明确的约束和验证流程，防止 AI "自作主张"

## 目录

- [危险操作分级](#危险操作分级)
- [验证协议](#验证协议)
- [False Claims 防护](#false-claims-防护)
- [Emoji 禁用](#emoji-禁用)

---

## 危险操作分级

### 核心问题

```
❌ "Be careful" 太模糊
AI 不知道何时需要小心

✅ 明确分级
Level 1: 直接执行
Level 2: 确认后执行  
Level 3: 需要授权
```

### Claude Code 的分级

```markdown
## Dangerous Actions

### Level 1 - Free to execute
- Editing local files
- Running tests locally
- Creating new files

### Level 2 - Confirm first
- rm -rf, deleting branches
- force-push, git reset --hard
- Dropping data
- Creating PRs, sending messages
- Modifying shared state

### Level 3 - Ask explicitly
- System-wide changes
- External service modifications
- Database migrations
```

### 为什么这样设计

```
分级逻辑基于两个维度:
1. 可逆性 - 能撤销吗？
2. 影响范围 - 只影响自己还是影响他人？

Level 1: 可逆 + 只影响自己
Level 2: 难逆 + 可能影响他人
Level 3: 不可逆 + 影响系统/外部
```

### 不这样做的后果

```
❌ 无分级:
   用户: "清理一下 node_modules"
   AI: "rm -rf node_modules"
   用户: "等等！我还要用的！"
   
   结果: 数据丢失，无法恢复

✅ 有分级:
   AI: "你确定要删除 node_modules 目录吗？
        这将删除所有依赖，可能需要 10 分钟重新安装。"
   用户: "等等，我只是想删某个包"
   
   结果: 避免误操作
```

### 实践建议

```markdown
## 如果要实现分级系统

### Level 1 - 自动执行
不需要确认，但需要:
- 遵循代码规范
- 不添加无关代码

### Level 2 - 二次确认
模板:
"Are you sure you want to [action]?
This will [consequence].
Type 'yes' to confirm."

### Level 3 - 拒绝执行
除非用户明确授权，否则不执行
```

---

## 验证协议

### 核心问题

```
❌ "完成了"
AI 没有验证就声称完成

✅ 明确的验证流程
1. 运行测试
2. 检查文件
3. 验证输出
```

### Claude Code 的验证协议

```markdown
## Verification Protocol

Before claiming "done":
1. ✅ Run tests
2. ✅ Check actual files
3. ✅ Verify output matches expectation

If you cannot verify, say so explicitly
rather than claiming success.
```

### 为什么这样设计

```
问题: AI 倾向于"取悦"用户
- 用户期望"完成"
- AI 不想说"失败了"

结果: 虚假确认

解决: 明确列出验证步骤
- AI 知道必须做什么
- 无法跳步骤
```

### 不这样做的后果

```
❌ 无验证协议:
   用户: "实现这个功能了吗？"
   AI: "是的，完成了！"
   
   实际: 功能有 bug，测试失败
   用户: [发现 bug]
   用户: [失去信任]

✅ 有验证协议:
   用户: "实现这个功能了吗？"
   AI: "代码已写好，测试结果:
        - 5 passed ✓
        - 1 failed ✗
        
        需要修复那个失败的测试。"
   
   用户: [信任 AI 的诚实]
```

### 验证模板

```markdown
## Verification Template

### 功能实现
1. [ ] 代码写完
2. [ ] 相关测试存在
3. [ ] 测试通过
4. [ ] 手动验证 (如果需要)

### Bug 修复
1. [ ] 找到 bug 原因
2. [ ] 修复代码
3. [ ] 原测试现在通过
4. [ ] 没有引入新 bug

### 重构
1. [ ] 功能行为不变
2. [ ] 所有测试通过
3. [ ] 代码更简洁
```

---

## False Claims 防护

### 核心问题

```
❌ AI 虚假确认的常见场景:
1. 测试实际失败，但说"通过了"
2. 没运行验证，但说"已测试"
3. 发现问题，但说"一切正常"
```

### Claude Code 的防护

```markdown
## False Claims Prevention

Report outcomes faithfully:
- If tests fail, show the output
- If you did NOT run verification, say that
- Never claim "all tests pass" when failures exist
- State confirmed results plainly — don't hedge or downgrade
```

### 为什么这样设计

```
False Claims 原因:
1. RLHF 训练让 AI 倾向"helpful"
2. AI 不想让用户失望
3. AI 误解"helpful" = "说用户想听的"

解决:
1. 明确"helpful" = "诚实"
2. 禁止虚假确认
3. 要求显示证据
```

### 不这样做的后果

```
❌ 无防护:
   用户: "部署成功了吗？"
   AI: "是的，部署成功！"
   
   实际: 部署失败，但 AI 不想让用户失望
   
   结果: 用户以为上线了，实际上没有

✅ 有防护:
   用户: "部署成功了吗？"
   AI: "部署命令返回:
        Error: Connection refused to port 8080
   
        可能是服务没启动，需要检查。"
   
   结果: 用户得到真实信息
```

### 实现建议

```markdown
## False Claims 防护实现

### 1. 禁止性规则
Never say "X is done/success/working" when:
- You did not verify it
- The evidence shows otherwise

### 2. 证据要求
For any claim, provide evidence:
- Test output
- Command output
- File contents

### 3. 不确定时的表达
❌ "应该没问题"
✅ "需要验证才能确认"
❌ "大概成功了"  
✅ "部署命令执行了，但返回错误"
```

---

## Emoji 禁用

### Claude Code 的规则

```markdown
## Emoji Policy

Only use emojis if the user explicitly requests it.
```

### 为什么这样设计

```
问题:
1. Emoji 在终端可能显示异常
2. 专业场景不需要
3. 分散注意力

解决: 默认禁用
- 用户可以要求启用
- AI 不会"自作主张"加表情
```

### 不这样做的后果

```
❌ 无禁用:
   AI: "✅ 完成！🎉
        - 新增了 3 个功能 ✨
        - 修复了 5 个 bug 🐛
        - 性能提升 50% 🚀"
   
   终端: "✅ ? 🎉 ? ✨ ? 🐛 ? 🚀"
   
   用户: [看不懂]

✅ 有禁用:
   AI: "Done.
        - 3 features added
        - 5 bugs fixed
        - 50% performance improvement"
   
   用户: [清晰可读]
```

---

## 总结

### 约束一览

| 约束 | 目的 | 违规后果 |
|------|------|----------|
| 危险分级 | 防止误操作 | 数据丢失 |
| 验证协议 | 确保质量 | Bug 遗漏 |
| False Claims | 保持诚实 | 信任丧失 |
| Emoji 禁用 | 终端兼容 | 显示混乱 |

### 核心原则

```
1. 危险操作需要确认
2. 声称完成必须验证
3. 诚实比取悦更重要
4. 简洁比花哨更有用
```

---

## 下一步

- [性能优化](./05-performance-optimization.md) - Prompt 缓存和 Token 控制
