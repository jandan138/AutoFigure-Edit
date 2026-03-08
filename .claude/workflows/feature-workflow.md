# Workflow: Feature Development (AutoFigure-Edit) — Claude Code 版

## flow
feature request
-> architect
-> feature-designer
-> implementation-engineer (+ frontend-engineer 并行，如涉及前端变更)
-> code-reviewer (可与 doc-writer、test-engineer 并行)
-> doc-writer (可与 code-reviewer、test-engineer 并行)
-> test-engineer (可与 code-reviewer、doc-writer 并行)
-> merge

## claude_code_execution

### 阶段 1: architect（general-purpose agent）
```
Agent tool:
  subagent_type: "general-purpose"
  description: "architecture design"
  prompt: |
    读取 .claude/agents/architect.md 获取角色定义。
    <任务描述>
    输出架构决策、子系统影响、写入所有权分配。
```
on_failure:
  action: retry
  max_retries: 2
  strategy: 重新调研相关代码和文档后重试，每次重试需补充额外上下文

### 阶段 2: feature-designer（general-purpose agent）
```
Agent tool:
  subagent_type: "general-purpose"
  description: "feature spec design"
  prompt: |
    读取 .claude/agents/feature-designer.md 获取角色定义。
    前置输入：<architect 输出>
    输出行为规格、实现映射、边界情况。
```
on_failure:
  action: escalate_back
  target_stage: architect
  strategy: 回退到 architect 阶段，要求补充上下文和约束条件后重新生成架构决策

### 阶段 3: implementation-engineer（general-purpose agent）
isolation: "worktree"
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
on_failure:
  action: decompose_and_retry
  strategy: 记录错误信息，尝试将任务分解为更小的子任务后逐一实现

### 阶段 3b（条件性）: frontend-engineer（general-purpose agent，与 implementation-engineer 并行）
触发条件: 需求涉及前端变更（web/ 目录下的文件）
isolation: "worktree"
```
Agent tool:
  subagent_type: "general-purpose"
  description: "implement frontend changes"
  run_in_background: true
  prompt: |
    读取 .claude/agents/frontend-engineer.md 获取角色定义。
    实现规格：<feature-designer 输出中的前端部分>
    Write Scope: web/index.html, web/app.js, web/canvas.html
    No-Write Scope: web/vendor/svg-edit/, autofigure2.py, server.py
```
on_failure:
  action: decompose_and_retry
  strategy: 记录错误信息，尝试将任务分解为更小的子任务后逐一实现

### 阶段 4-5-6: code-reviewer + doc-writer + test-engineer（可并行）
```
# 并行启动三个 agent（在同一条消息中发送三个 Agent tool call）

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

Agent tool 3:
  subagent_type: "general-purpose"
  description: "regression testing"
  run_in_background: true
  prompt: |
    读取 .claude/agents/test-engineer.md 获取角色定义。
    变更摘要：<implementation-engineer 输出>
    执行回归验证，确认变更未破坏现有功能。
```
on_failure:
  code-reviewer:
    action: escalate_back
    target_stage: implementation-engineer
    condition: 发现 BLOCKER 级别问题
    strategy: 回退到 implementation-engineer 修复 BLOCKER 问题后重新提交审查
  doc-writer:
    action: warn
    strategy: 标记为 WARN，不阻塞合并；在 merge 阶段记录文档待补充项
  test-engineer:
    action: escalate_back
    target_stage: implementation-engineer
    condition: 回归测试发现功能破坏
    strategy: 回退到 implementation-engineer 修复后重新验证

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
- Implement scoped changes in isolated worktree.
- Return evidence package: files changed, commands run, key outputs, risks/open items.

3b. frontend-engineer (条件性)
- Implement frontend changes in isolated worktree, parallel with implementation-engineer.
- Write scope limited to web/ (excluding vendor/svg-edit/).
- Return evidence package: files changed, UI behavior changes.

4. code-reviewer
- Perform code quality and safety audit.
- `BLOCKER` findings stop merge until fixed.
- Enforce Code Review Gate for API key safety and pipeline integrity.

5. doc-writer
- Update docs impacted by behavior change.
- Enforce Docs Gate: no behavior-change completion without docs update or no-doc-change rationale.

6. test-engineer
- Execute regression verification on changed functionality.
- Confirm existing pipeline stages still function correctly.
- Report pass/fail with evidence.

## completion_criteria
- All stages completed in sequence (4-5-6 may be parallel, 3-3b may be parallel).
- No unresolved `BLOCKER` findings.
- No unresolved regression test failures.
- User-visible behavior changes are documented or have an approved no-doc-change rationale.
