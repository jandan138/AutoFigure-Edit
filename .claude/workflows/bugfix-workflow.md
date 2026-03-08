# Workflow: Bugfix (AutoFigure-Edit) — Claude Code 版

## flow
bug report
-> implementation-engineer
-> code-reviewer
-> doc-writer + test-engineer (并行)
-> merge

## claude_code_execution

### 阶段 1: implementation-engineer
```
Agent tool:
  subagent_type: "general-purpose"
  description: "fix bug"
  isolation: "worktree"
  prompt: |
    读取 .claude/agents/implementation-engineer.md 获取角色定义。
    Bug 描述：<bug report>
    要求：
    1. 定位最小根因
    2. 修复 bug
    3. 验证修复
    Write Scope: <受影响的文件>
    No-Write Scope: web/vendor/svg-edit/
    完成后验证：python autofigure2.py --help
  on_failure: |
    1. 分析根因，检查日志和错误堆栈
    2. 尝试替代修复方案
    3. 如多次失败，输出诊断报告并中止 workflow
```

### 阶段 2: code-reviewer
```
Agent tool:
  subagent_type: "general-purpose"
  description: "review bugfix"
  prompt: |
    读取 .claude/agents/code-reviewer.md 获取角色定义。
    Bug 修复：<implementation-engineer 输出>
    重点检查修复是否引入新问题。
  on_failure: |
    1. 如发现 BLOCKER 级别问题，回退到阶段 1 重新修复
    2. 将 BLOCKER 详情作为补充 bug report 传递给 implementation-engineer
    3. 最多回退 2 次，超过则中止 workflow 并上报
```

### 阶段 3: doc-writer + test-engineer（并行）

#### 3a: doc-writer（如需）
```
Agent tool:
  subagent_type: "general-purpose"
  description: "update docs for bugfix"
  prompt: |
    读取 .claude/agents/doc-writer.md 获取角色定义。
    Bug 修复摘要：<implementation-engineer 输出>
    判断是否需要更新文档。如不需要，提供 no-doc-change rationale。
    Write Scope: README.md, README_ZH.md, CLAUDE.md
    No-Write Scope: autofigure2.py, server.py, web/
  on_failure: |
    1. 标记 WARN，不阻塞 workflow
    2. 记录失败原因，留待后续手动处理
```

#### 3b: test-engineer（回归验证）
```
Agent tool:
  subagent_type: "general-purpose"
  description: "regression test for bugfix"
  prompt: |
    验证 bug 修复的回归影响：
    Bug 修复摘要：<implementation-engineer 输出>
    要求：
    1. 确认原始 bug 已修复
    2. 运行相关功能验证，确保无回归
    3. 验证：python autofigure2.py --help
    4. 输出回归测试报告
    No-Write Scope: autofigure2.py, server.py, web/
  on_failure: |
    1. 标记 WARN，记录未通过的验证项
    2. 如发现回归问题，回退到阶段 1 重新修复
```

## stage_contracts
1. implementation-engineer
- Locate and fix minimal root cause.
- Return evidence package: files changed, commands run, key outputs, risks/open items.
- Isolation: worktree (防止修复过程中的中间状态影响主分支).

2. code-reviewer
- Review fix for correctness and safety.
- Return severity-ordered findings and verdict.
- On BLOCKER: trigger rollback to implementation-engineer with detailed feedback.

3. doc-writer
- Update docs when bugfix changes user-visible behavior.
- If docs are not required, add a short no-doc-change rationale.
- On failure: WARN only, non-blocking.

4. test-engineer
- Run regression verification on the bugfix.
- Confirm original bug is resolved and no new issues introduced.
- On regression found: trigger rollback to implementation-engineer.

## completion_criteria
- Bug is fixed and verified.
- Code review passed with no `BLOCKER` findings.
- Regression test passed with no new issues.
- Docs updated for behavior changes or explicit no-doc-change rationale accepted.
