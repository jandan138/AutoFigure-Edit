# Workflow: Pipeline Extension (AutoFigure-Edit) — Claude Code 版

## flow
new pipeline stage / backend / provider request
-> architect
-> pipeline-engineer
-> code-reviewer (可与 doc-writer, test-engineer 并行)
-> doc-writer (可与 code-reviewer, test-engineer 并行)
-> test-engineer (可与 code-reviewer, doc-writer 并行)
-> merge

## claude_code_execution

### 阶段 1: architect（general-purpose agent）
```
Agent tool:
  subagent_type: "general-purpose"
  description: "pipeline architecture"
  prompt: |
    读取 .claude/agents/architect.md 获取角色定义。
    目标：设计 <新 pipeline 阶段/backend/provider> 的架构方案。
    要求：
    1. 定义集成点和数据流
    2. CLI 参数设计
    3. 与现有 pipeline 的兼容性
    4. 写入所有权分配
  on_failure: |
    重新调研相关技术文档和现有 pipeline 代码，分析失败原因后重试。
    最多重试 2 次，仍失败则终止 workflow 并报告原因。
```

### 阶段 2: pipeline-engineer
```
Agent tool:
  subagent_type: "general-purpose"
  description: "build pipeline stage"
  isolation: "worktree"
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
  on_failure: |
    分析错误日志，尝试将任务分解为更小的子步骤逐步实现。
    若为依赖问题，先解决依赖再重试实现。
    最多重试 2 次，仍失败则终止 workflow 并报告原因。
```

### 阶段 3-5: code-reviewer + doc-writer + test-engineer（可并行）
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
  on_failure: |
    若发现 BLOCKER 级别问题，回退到 pipeline-engineer 阶段进行修复后重新审查。
    若为非 BLOCKER 问题，记录为 WARN 并继续。

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
  on_failure: |
    标记为 WARN，不阻塞 workflow 完成。
    记录未更新的文档列表，后续手动补充。

Agent tool 3:
  subagent_type: "general-purpose"
  description: "pipeline integration test"
  run_in_background: true
  prompt: |
    Pipeline 变更：<pipeline-engineer 输出>
    要求：
    1. 验证新 pipeline 阶段/backend/provider 的 CLI 参数可用
    2. 验证 method_to_svg() 数据流完整性（dry-run 或 --help 级别）
    3. 检查与现有 backend/provider 的兼容性（无回归）
    4. 报告验证结果：PASS / FAIL + 详情
    Read Scope: autofigure2.py, requirements.txt
    No-Write Scope: 所有文件（只读验证）
  on_failure: |
    记录失败的测试用例和错误信息。
    若为关键集成问题，标记为 BLOCKER 并回退到 pipeline-engineer 修复。
    若为环境问题，标记为 WARN 并继续。
```

## stage_contracts
1. architect
- Define integration points, data flow, and CLI parameter design.
- Assign write ownership with no overlap.

2. pipeline-engineer
- Implement pipeline stage/backend/provider and integrate into `method_to_svg()`.
- Follow existing function naming and provider patterns.
- Include evidence package on completion.
- Runs in worktree isolation to avoid conflicts with parallel work.

3. code-reviewer
- Verify pipeline data flow integrity and provider interface consistency.
- Enforce Code Review Gate for pipeline correctness.
- BLOCKER findings trigger rollback to pipeline-engineer.

4. doc-writer
- Update docs with new pipeline capabilities, CLI options, and usage examples.
- Enforce Docs Gate for user-visible changes.

5. test-engineer
- Validate pipeline integration via CLI verification and data flow checks.
- Report PASS/FAIL with details for each validation item.

## completion_criteria
- New pipeline stage/backend/provider is functional via CLI.
- Code review passed with no `BLOCKER` findings.
- Pipeline integration test passed (test-engineer).
- Docs updated with usage examples using placeholder API keys.
