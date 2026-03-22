# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AutoFigure-Edit generates editable SVG scientific figures from paper method text. It uses LLMs to generate a raster draft, YOLO-World (default) or OWL-ViT for icon segmentation, rembg (BiRefNet) for background removal, then assembles a final SVG with real icons replacing placeholders. Published at ICLR 2026.

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
2. **Icon Segmentation** (`segment_with_sam3`) — Detects icons/regions using YOLO-World (default), OWL-ViT, or SAM3 (local/fal/roboflow), merges overlapping boxes → `samed.png` + `boxlib.json`
3. **Background Removal** (`crop_and_remove_background`, `RembgRemover`) — Crops detected regions and removes backgrounds with rembg (BiRefNet, default) or BriaRMBG2 (fallback) → `icons/*.png`
4. **SVG Template Generation** (`generate_svg_template`) — LLM generates SVG with AF-style placeholders (`<AF>01`, `<AF>02`...) → `template.svg`
5. **SVG Validation & Fix** (`check_and_fix_svg`) — Validates SVG with lxml, optionally fixes via LLM
6. **SVG Optimization** (`optimize_svg_with_llm`) — Optional iterative LLM refinement → `optimized_template.svg`
7. **Icon Replacement** (`replace_icons_in_svg`) — Matches placeholders by label, computes scale factors, injects base64 icon images → `final.svg`

**LLM Provider abstraction:** Three providers (openrouter, bianxie, gemini) each have `_call_{provider}_text`, `_call_{provider}_multimodal`, and `_call_{provider}_image_generation` functions. The unified entry points are `call_llm_text()`, `call_llm_multimodal()`, and `call_llm_image_generation()`.

**Segmentation backends:** `yolo_world` (default, YOLO-World zero-shot detection via ultralytics), `owlvit` (Google OWL-ViT), `diagram` (multimodal LLM that classifies regions into `flow_box`, `text_block`, `rendered_image`, `code_block` — suited for flowchart-style paper figures with text boxes, bullet point annotations, arrows, and embedded renders), `local` (requires SAM3 installed), `fal` (fal.ai API), `roboflow` (Roboflow API), `api` (generic SAM API). Configured via `--sam_backend`. The `diagram` backend reuses the main LLM credentials automatically (no extra API key needed) and skips background removal to preserve visual styling.

**Auto-prompt generation:** When `--auto_prompts` is set and `--sam_backend yolo_world` is active, the pipeline calls `call_llm_text()` before segmentation to extract 5–15 concrete detection nouns from the method text (e.g., `robot,camera,server`). The extracted string is used as the YOLO-World prompt. If LLM extraction fails, the pipeline falls back to `--sam_prompt`. Disabled by default (backward-compatible). Adds ~1–2 s latency and ~500 tokens per run.

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
- `--sam_backend`: Segmentation backend (`yolo_world` [default], `owlvit`, `diagram`, `local`, `fal`, `roboflow`, `api`). Use `diagram` for flowchart-style figures with text boxes/arrows/embedded renders; it calls the multimodal LLM and requires no extra install or API key.
- `--sam_prompt`: Comma-separated detection nouns passed to YOLO-World (e.g., `robot,camera,server`)
- `--auto_prompts`: Let the LLM extract detection nouns from method text automatically (only effective with `yolo_world` backend); falls back to `--sam_prompt` on failure; disabled by default
- `--merge_threshold`: IoU threshold for merging overlapping detections (0 disables)
- `--optimize_iterations`: Number of LLM refinement passes on SVG template (0 skips)
- `--reference_image_path`: Optional style reference image for generation

## Recent Improvements (2026-03)

### Diagram Backend v2
Extended `diagram` backend with two key enhancements:

1. **`text_block` element type** (`_segment_with_diagram_llm`): Detects floating text regions outside flow boxes (bullet points, captions, figure titles) as separate elements.

2. **`skip_rembg` parameter** (`crop_and_remove_background`): When using `diagram` backend, crops are copied directly to `_nobg.png` without background removal, preserving borders and visual styling of flow boxes and rendered scenes.

### SVG Template Size Locking
`generate_svg_template` now validates SVG dimensions against the original image:
- Added `_check_svg_size_matches()` helper to verify width/height within 5% threshold
- Retry loop with warning prompt (up to `max_size_retries=2` times) when LLM generates wrong canvas size
- Prevents coordinate drift from scale factors (e.g., `scale_y=1.23`) in step 4.7

### Icon Aspect Ratio Preservation
`replace_icons_in_svg` now preserves original icon proportions:
- Calculates `scale = min(placeholder_w/icon_w, placeholder_h/icon_h)` for fit-in-box sizing
- Centers icon within placeholder using `preserveAspectRatio="xMidYMid meet"`
- Prevents flow_box icons from being stretched/distorted when aspect ratios differ

## Language Notes

Code comments and docstrings are primarily in Chinese. The codebase uses Python 3.10+ features (match statements, type unions with `|`).

## Agent Team

本项目配置了多 agent 协作系统，用于复杂任务的分工执行。

### 调用方式

读取 `.claude/orchestrator.md` 获取完整的路由和执行规则，然后按对应 workflow 执行：

```text
# Research
请读取 .claude/orchestrator.md 并按 research workflow 执行以下调研：<调研问题>

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
| researcher | general-purpose | 技术调研、方案可行性分析、代码探索（只读，不改代码） |
| architect | general-purpose | Pipeline 架构设计、LLM provider 抽象、segmentation backend 选择（仅设计，不改代码） |
| feature-designer | general-purpose | 将需求转化为实现规格（仅设计，不改代码） |
| implementation-engineer | general-purpose | 代码实现 |
| pipeline-engineer | general-purpose | Pipeline 阶段/backend/provider 集成 |
| code-reviewer | general-purpose | 代码质量、API key 安全、pipeline 完整性审查 |
| doc-writer | general-purpose | 文档更新 |
| test-engineer | general-purpose | 冒烟测试、回归验证、pipeline 集成测试 |
| frontend-engineer | general-purpose | Web 前端开发（web/ 目录，排除 vendor/svg-edit/） |

### Workflow（`.claude/workflows/`）

每个 workflow 均包含 **on_failure 失败处理策略** 和 **worktree 隔离标注**。

- `research-workflow.md`: researcher [→ architect 如涉及架构决策]
- `feature-workflow.md`: architect → feature-designer → implementation-engineer (worktree) [+ frontend-engineer 如涉及前端] → code-reviewer + doc-writer + test-engineer (并行)
- `bugfix-workflow.md`: implementation-engineer (worktree) → code-reviewer + doc-writer + test-engineer (并行)
- `pipeline-extension.md`: architect → pipeline-engineer (worktree) → code-reviewer + doc-writer + test-engineer (并行)

### 写入所有权规则

- `autofigure2.py` 同一时间只允许一个 agent 修改
- `web/vendor/svg-edit/` 所有 agent 只读
- 并行 agent 的写入范围不得重叠
