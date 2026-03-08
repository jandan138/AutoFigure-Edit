# Workflow: Bugfix (AutoFigure-Edit) — Claude Code 版

## flow
bug report
-> implementation-engineer
-> code-reviewer
-> doc-writer (如需)
-> merge

## claude_code_execution

### 阶段 1: implementation-engineer
```
Agent tool:
  subagent_type: "general-purpose"
  description: "fix bug"
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
```

### 阶段 3: doc-writer（如需）
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
```

## stage_contracts
1. implementation-engineer
- Locate and fix minimal root cause.
- Return evidence package: files changed, commands run, key outputs, risks/open items.

2. code-reviewer
- Review fix for correctness and safety.
- Return severity-ordered findings and verdict.

3. doc-writer
- Update docs when bugfix changes user-visible behavior.
- If docs are not required, add a short no-doc-change rationale.

## completion_criteria
- Bug is fixed and verified.
- Code review passed with no `BLOCKER` findings.
- Docs updated for behavior changes or explicit no-doc-change rationale accepted.
