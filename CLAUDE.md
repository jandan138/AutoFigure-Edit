# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AutoFigure-Edit generates editable SVG scientific figures from paper method text. It uses LLMs to generate a raster draft, OWL-ViT (default) or SAM3 for icon segmentation, RMBG-2.0 for background removal, then assembles a final SVG with real icons replacing placeholders. Published at ICLR 2026.

## Commands

```bash
# Install dependencies
pip install -r requirements.txt

# (Optional) Install SAM3 — only needed if using --sam_backend local
git clone https://github.com/facebookresearch/sam3.git && cd sam3 && pip install -e .

# Run CLI pipeline (uses OWL-ViT backend by default, no extra install needed)
python autofigure2.py \
  --method_file paper.txt \
  --output_dir outputs/demo \
  --provider bianxie \
  --api_key YOUR_KEY

# Run web server (FastAPI + uvicorn on port 8000)
python server.py
```

There are no tests or linting configured in this project.

## Architecture

The project has two Python files and a static web frontend:

### `autofigure2.py` — Core Pipeline (~2600 lines, single file)

The `method_to_svg()` function at line ~2390 orchestrates the full pipeline:

1. **Figure Generation** (`generate_figure_from_method`) — Calls LLM to generate a raster figure from method text → `figure.png`
2. **Icon Segmentation** (`segment_with_sam3`) — Detects icons/regions using OWL-ViT (default) or SAM3 (local/API: fal.ai/Roboflow), merges overlapping boxes → `samed.png` + `boxlib.json`
3. **Background Removal** (`crop_and_remove_background`, `BriaRMBG2Remover`) — Crops detected regions and removes backgrounds with RMBG-2.0 → `icons/*.png`
4. **SVG Template Generation** (`generate_svg_template`) — LLM generates SVG with AF-style placeholders (`<AF>01`, `<AF>02`...) → `template.svg`
5. **SVG Validation & Fix** (`check_and_fix_svg`) — Validates SVG with lxml, optionally fixes via LLM
6. **SVG Optimization** (`optimize_svg_with_llm`) — Optional iterative LLM refinement → `optimized_template.svg`
7. **Icon Replacement** (`replace_icons_in_svg`) — Matches placeholders by label, computes scale factors, injects base64 icon images → `final.svg`

**LLM Provider abstraction:** Three providers (openrouter, bianxie, gemini) each have `_call_{provider}_text`, `_call_{provider}_multimodal`, and `_call_{provider}_image_generation` functions. The unified entry points are `call_llm_text()`, `call_llm_multimodal()`, and `call_llm_image_generation()`.

**Segmentation backends:** `owlvit` (default, Google OWL-ViT zero-shot detection, auto-downloads from HuggingFace), `local` (requires SAM3 installed), `fal` (fal.ai API), `roboflow` (Roboflow API), `api` (generic SAM API). Configured via `--sam_backend`.

### `server.py` — FastAPI Web Backend

- Spawns `autofigure2.py` as a subprocess per job
- Monitors output directory for artifacts and streams SSE events to the frontend
- Job management via in-memory `JOBS` dict with `Job` dataclass
- API endpoints: `/api/run`, `/api/events/{job_id}`, `/api/artifacts/{job_id}/{path}`, `/api/upload`, `/api/config`
- Serves static files from `web/` directory

### `web/` — Static Frontend

- `index.html` / `app.js` — Configuration page and job launcher
- `canvas.html` — SVG editor (embedded svg-edit vendor) with artifact panel
- `vendor/svg-edit/` — Bundled SVG-Edit editor (do not modify)

## Key Configuration Flags

- `--placeholder_mode`: `label` (recommended, gray fill + numbered labels), `box` (coordinates), `none`
- `--sam_backend`: Segmentation backend (`owlvit` [default], `local`, `fal`, `roboflow`, `api`)
- `--merge_threshold`: IoU threshold for merging overlapping detections (0 disables)
- `--optimize_iterations`: Number of LLM refinement passes on SVG template (0 skips)
- `--reference_image_path`: Optional style reference image for generation

## Language Notes

Code comments and docstrings are primarily in Chinese. The codebase uses Python 3.10+ features (match statements, type unions with `|`).

## Agent Team

本项目配置了多 agent 协作系统，用于复杂任务的分工执行。

### 调用方式

读取 `.claude/orchestrator.md` 获取完整的路由和执行规则，然后按对应 workflow 执行：

```text
# Feature
请读取 .claude/orchestrator.md 并按 feature workflow 执行以下需求：<需求描述>

# Bugfix
请读取 .claude/orchestrator.md 并按 bugfix workflow 执行：<bug 描述>

# Pipeline extension (新增 provider/backend/pipeline 阶段)
请读取 .claude/orchestrator.md 并按 pipeline-extension workflow 执行：<描述>
```

### Agent 角色（`.claude/agents/`）

| Agent | 类型 | 职责 |
|-------|------|------|
| architect | Plan | Pipeline 架构设计、LLM provider 抽象、SAM backend 选择 |
| feature-designer | Plan | 将需求转化为实现规格 |
| implementation-engineer | general-purpose | 代码实现 |
| pipeline-engineer | general-purpose | Pipeline 阶段/backend/provider 集成 |
| code-reviewer | general-purpose | 代码质量、API key 安全、pipeline 完整性审查 |
| doc-writer | general-purpose | 文档更新 |

### Workflow（`.claude/workflows/`）

- `feature-workflow.md`: architect → feature-designer → implementation-engineer → code-reviewer + doc-writer (并行)
- `bugfix-workflow.md`: implementation-engineer → code-reviewer → doc-writer (如需)
- `pipeline-extension.md`: architect → pipeline-engineer → code-reviewer + doc-writer (并行)

### 写入所有权规则

- `autofigure2.py` 同一时间只允许一个 agent 修改
- `web/vendor/svg-edit/` 所有 agent 只读
- 并行 agent 的写入范围不得重叠
