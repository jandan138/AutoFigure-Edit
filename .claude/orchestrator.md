# AutoFigure-Edit Multi-Agent Orchestrator（Claude Code 版）

## purpose
在 Claude Code 环境下，将任务路由到专业化 agent，协调科学图形生成流水线的开发工作。

## repository_context
- Pipeline core: `autofigure2.py` (~2600 lines, single file)
- Web server: `server.py` (FastAPI + SSE)
- Frontend: `web/` (index.html, app.js, canvas.html, vendor/svg-edit/)
- Entry function: `method_to_svg()` at line ~2390
- LLM providers: openrouter, bianxie, gemini (unified via `call_llm_text/multimodal/image_generation`)
- SAM3 backends: local, fal, roboflow
- No tests or linting configured

## claude_code_execution_model

Claude Code 通过 Agent tool 分派子任务：
- 每个子 agent 通过 `Agent` tool 启动，`prompt` 参数包含角色定义和具体任务
- 使用 `subagent_type: "general-purpose"` 执行实现/审查任务
- 使用 `subagent_type: "Plan"` 执行架构和设计任务
- 使用 `subagent_type: "Explore"` 执行代码调研任务
- 独立任务可通过 `run_in_background: true` 并行执行
- 使用 `isolation: "worktree"` 为需要独立代码修改的 agent 提供隔离环境

## agent_selection_rules
- Use `architect` when request changes pipeline architecture, adds new stages, or modifies provider/backend model.
- Use `feature-designer` when feature intent must be converted into implementation spec.
- Use `implementation-engineer` for coding tasks and refactors.
- Use `pipeline-engineer` for pipeline stage additions, SAM backend integrations, or SVG processing changes.
- Use `code-reviewer` before final completion of any coding workflow.
- Use `doc-writer` for all documentation updates.

角色定义文件位于 `.claude/agents/*.md`，作为 Agent tool prompt 的模板。

## task_routing
1. 分类任务类型：
- Feature -> `.claude/workflows/feature-workflow.md`
- Bugfix -> `.claude/workflows/bugfix-workflow.md`
- Pipeline extension -> `.claude/workflows/pipeline-extension.md`

2. 按 workflow 阶段顺序执行：
- 读取对应 workflow 文件获取阶段定义
- 读取对应 agent 角色文件获取 prompt 模板
- 通过 Agent tool 启动子 agent，将角色定义 + 具体任务作为 prompt
- 前一阶段输出作为后一阶段的输入上下文
- 独立阶段可并行启动（如 code-reviewer 和 doc-writer）

3. Policy enforcement:
- Forbidden:
  - exposing API keys in outputs/docs/commits
  - breaking LLM provider unified interface (`call_llm_text/multimodal/image_generation`)
  - modifying `web/vendor/svg-edit/` (bundled third-party code)
  - shipping pipeline changes without manual verification
- Severity model:
  - `BLOCKER`: must be fixed before merge
  - `WARN`: may continue, but must be documented in final summary

## task_contract_rules
- Every delegated task must include a contract with:
  - goal
  - allowed write scope
  - forbidden/no-write scope
  - required deliverables
  - validation commands

## write_ownership_rules
- Parallel agents must have non-overlapping write scopes.
- Shared files must be assigned a single owner thread.
- `autofigure2.py` is the main pipeline file — only one agent should modify it at a time.
- `web/vendor/svg-edit/` is read-only for all agents.
- Claude Code 中通过在 Agent tool prompt 中明确指定 write scope 实现。

## evidence_based_done_rules
- A stage is not done unless it reports:
  - files changed
  - commands run (if applicable)
  - key output summary
  - risks/open items
  - docs impact (updated docs or explicit no-doc-change rationale)

## gate_rules
- Docs Gate:
  - Behavior changes (CLI options, pipeline stages, API endpoints) require docs updates.
- Code Review Gate:
  - API key handling, provider interface, and pipeline correctness changes require code-reviewer verdict.
- Manual Verification Gate:
  - Pipeline changes should be verified with a test run when feasible.

## final_summary_template
- Final summary must include these sections in order:
  1. Changes
  2. Verification
  3. Documentation
  4. Code Quality
  5. Risks
  6. Next Steps

## completion_criteria
A task is complete only when all are true:
- Workflow stages completed.
- Manual verification performed when feasible.
- Documentation updates are merged for all behavior changes, or a documented no-doc-change rationale is accepted.
- Code review completed.
- No unresolved `BLOCKER` findings.
- Final summary includes: changes, verification, risks/warnings, and follow-ups.

## invocation_examples

Feature request:
```text
请读取 .claude/orchestrator.md 并按 feature workflow 执行以下需求：
<需求描述>
要求产出架构/设计/实现/审查结果。
```

Pipeline extension:
```text
请读取 .claude/orchestrator.md 并按 pipeline-extension workflow 执行：
目标：<新增 pipeline 阶段或 backend 描述>
```

Bugfix:
```text
请读取 .claude/orchestrator.md 并按 bugfix workflow 执行：
<bug 描述>
要求回归验证和代码审查判定。
```
