# Workflow: Pipeline Extension (AutoFigure-Edit) — Claude Code 版

## flow
new pipeline stage / backend / provider request
-> architect
-> pipeline-engineer
-> code-reviewer (可与 doc-writer 并行)
-> doc-writer (可与 code-reviewer 并行)
-> merge

## claude_code_execution

### 阶段 1: architect（Plan agent）
```
Agent tool:
  subagent_type: "Plan"
  description: "pipeline architecture"
  prompt: |
    读取 .claude/agents/architect.md 获取角色定义。
    目标：设计 <新 pipeline 阶段/backend/provider> 的架构方案。
    要求：
    1. 定义集成点和数据流
    2. CLI 参数设计
    3. 与现有 pipeline 的兼容性
    4. 写入所有权分配
```

### 阶段 2: pipeline-engineer
```
Agent tool:
  subagent_type: "general-purpose"
  description: "build pipeline stage"
  prompt: |
    读取 .claude/agents/pipeline-engineer.md 获取角色定义。
    架构决策：<architect 输出>
    要求：
    1. 实现新 pipeline 阶段/backend/provider
    2. 集成到 method_to_svg() 流程
    3. 添加 CLI 参数（如需要）
    Write Scope: autofigure2.py
    No-Write Scope: web/vendor/svg-edit/
    完成后验证：python autofigure2.py --help
```

### 阶段 3-4: code-reviewer + doc-writer（可并行）
```
# 并行启动

Agent tool 1:
  subagent_type: "general-purpose"
  description: "review pipeline changes"
  run_in_background: true
  prompt: |
    读取 .claude/agents/code-reviewer.md 获取角色定义。
    Pipeline 变更：<pipeline-engineer 输出>
    重点检查：provider 接口一致性, pipeline 数据流完整性, 错误处理

Agent tool 2:
  subagent_type: "general-purpose"
  description: "pipeline docs"
  run_in_background: true
  prompt: |
    读取 .claude/agents/doc-writer.md 获取角色定义。
    Pipeline 变更：<pipeline-engineer 输出>
    更新相关文档，使用 placeholder API key。
    Write Scope: README.md, README_ZH.md, CLAUDE.md
    No-Write Scope: autofigure2.py, server.py, web/
```

## stage_contracts
1. architect
- Define integration points, data flow, and CLI parameter design.
- Assign write ownership with no overlap.

2. pipeline-engineer
- Implement pipeline stage/backend/provider and integrate into `method_to_svg()`.
- Follow existing function naming and provider patterns.
- Include evidence package on completion.

3. code-reviewer
- Verify pipeline data flow integrity and provider interface consistency.
- Enforce Code Review Gate for pipeline correctness.

4. doc-writer
- Update docs with new pipeline capabilities, CLI options, and usage examples.
- Enforce Docs Gate for user-visible changes.

## completion_criteria
- New pipeline stage/backend/provider is functional via CLI.
- Code review passed with no `BLOCKER` findings.
- Docs updated with usage examples using placeholder API keys.
