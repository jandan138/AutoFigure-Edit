# Workflow: Research (AutoFigure-Edit) — Claude Code 版

## flow
research request
-> researcher
-> architect (可选，当调研结论涉及架构决策时启动)
-> 输出调研报告

## claude_code_execution

### 阶段 1: researcher（general-purpose agent）
```
Agent tool:
  subagent_type: "general-purpose"
  description: "research investigation"
  prompt: |
    读取 .claude/agents/researcher.md 获取角色定义。
    调研任务：<调研问题描述>
    要求：
    1. 探索相关代码和文档
    2. 调研外部技术方案（如需要）
    3. 产出结构化调研报告
    No-Write Scope: 所有文件（只读）
```
on_failure:
  action: retry
  max_retries: 2
  strategy: 扩大调研范围，尝试不同的搜索关键词和信息源后重试

### 阶段 2（条件性）: architect（general-purpose agent）
触发条件: 调研结论涉及架构决策或 pipeline 变更建议
```
Agent tool:
  subagent_type: "general-purpose"
  description: "architecture assessment"
  prompt: |
    读取 .claude/agents/architect.md 获取角色定义。
    调研报告：<researcher 输出>
    要求：
    1. 评估调研结论对现有架构的影响
    2. 判断是否需要启动 feature/pipeline-extension workflow
    3. 产出架构评估意见
    No-Write Scope: 所有文件（只读）
```
on_failure:
  action: warn
  strategy: 标记为 WARN，保留 researcher 的调研报告作为最终产出

## stage_contracts
1. researcher
- Investigate the research question thoroughly using codebase exploration and external research.
- Return structured research report with evidence-based conclusions.
- Read-only: no file modifications allowed.

2. architect (条件性)
- Evaluate research conclusions against current pipeline architecture.
- Assess feasibility and impact of recommended approaches.
- Provide go/no-go recommendation for follow-up implementation workflows.
- Read-only: no file modifications allowed.

## completion_criteria
- Research report produced with all required sections (Background, Findings, Comparison, Conclusion, Risks, Next Steps).
- Conclusions are grounded in evidence (code references, documentation, benchmarks).
- If architect stage triggered: architecture assessment completed with clear recommendation.
- Next steps clearly identify which workflow to use for follow-up (feature/bugfix/pipeline-extension), if applicable.
