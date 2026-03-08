# Workflow: Feature Development (AutoFigure-Edit) — Claude Code 版

## flow
feature request
-> architect
-> feature-designer
-> implementation-engineer
-> code-reviewer (可与 doc-writer 并行)
-> doc-writer (可与 code-reviewer 并行)
-> merge

## claude_code_execution

### 阶段 1: architect（Plan agent）
```
Agent tool:
  subagent_type: "Plan"
  description: "architecture design"
  prompt: |
    读取 .claude/agents/architect.md 获取角色定义。
    <任务描述>
    输出架构决策、子系统影响、写入所有权分配。
```

### 阶段 2: feature-designer（Plan agent）
```
Agent tool:
  subagent_type: "Plan"
  description: "feature spec design"
  prompt: |
    读取 .claude/agents/feature-designer.md 获取角色定义。
    前置输入：<architect 输出>
    输出行为规格、实现映射、边界情况。
```

### 阶段 3: implementation-engineer（general-purpose agent）
```
Agent tool:
  subagent_type: "general-purpose"
  description: "implement feature"
  prompt: |
    读取 .claude/agents/implementation-engineer.md 获取角色定义。
    实现规格：<feature-designer 输出>
    Write Scope: <由 architect 分配>
    No-Write Scope: <由 architect 分配>
    完成后验证：python autofigure2.py --help
```

### 阶段 4-5: code-reviewer + doc-writer（可并行）
```
# 并行启动两个 agent（在同一条消息中发送两个 Agent tool call）

Agent tool 1:
  subagent_type: "general-purpose"
  description: "code review"
  run_in_background: true
  prompt: |
    读取 .claude/agents/code-reviewer.md 获取角色定义。
    变更摘要：<implementation-engineer 输出>

Agent tool 2:
  subagent_type: "general-purpose"
  description: "update docs"
  run_in_background: true
  prompt: |
    读取 .claude/agents/doc-writer.md 获取角色定义。
    Docs Delta：<implementation-engineer 的文档变更需求>
    Write Scope: README.md, README_ZH.md, CLAUDE.md
    No-Write Scope: autofigure2.py, server.py, web/
```

## stage_contracts
1. architect
- Output architecture decision and subsystem impact.
- Define constraints around LLM provider interface and pipeline stage boundaries.
- Assign non-overlapping write ownership for parallel implementation threads.

2. feature-designer
- Convert request into behavior spec and implementation map.
- Provide edge cases and acceptance criteria.
- Include task-contract fields: goal, write scope, no-write scope, deliverables, validation commands.

3. implementation-engineer
- Implement scoped changes.
- Return evidence package: files changed, commands run, key outputs, risks/open items.

4. code-reviewer
- Perform code quality and safety audit.
- `BLOCKER` findings stop merge until fixed.
- Enforce Code Review Gate for API key safety and pipeline integrity.

5. doc-writer
- Update docs impacted by behavior change.
- Enforce Docs Gate: no behavior-change completion without docs update or no-doc-change rationale.

## completion_criteria
- All stages completed in sequence (4-5 may be parallel).
- No unresolved `BLOCKER` findings.
- User-visible behavior changes are documented or have an approved no-doc-change rationale.
