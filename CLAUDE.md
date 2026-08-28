# OpenMontage — Claude Code Guide

**MANDATORY: Read [`AGENT_GUIDE.md`](AGENT_GUIDE.md) before responding to ANY video-production request.**

Do not act on a production request (make/create/produce/generate a video, animation, or clip) until you have read AGENT_GUIDE.md. It contains the routing rules and agent contract (pipeline selection, stage director skills, checkpoint/approval gates, decision logging) that determine correct behavior. Skipping it WILL cause you to take the wrong action — most commonly, writing ad-hoc scripts instead of going through the pipeline system.

This file does not restate that contract. It orients Claude Code inside the repo itself: where things live, how to set up and test the project, and the conventions that keep code consistent with the agent-first architecture. For a non-production request (fixing a bug, adding a tool, running tests, reading code), the sections below are enough to start; you don't need AGENT_GUIDE.md unless the task also involves running or extending a pipeline.

## What This Repository Is

OpenMontage is an open-source, agent-orchestrated video production platform (AGPLv3). An LLM coding assistant — not a Python runtime — is the orchestrator: it reads YAML pipeline manifests, follows Markdown "director" skills stage by stage, calls Python tools through a registry, and writes JSON checkpoints. There is no Python orchestration layer to trace; the control flow lives in instructions, and Python exists only for tools and persistence.

```
Agent reads pipeline manifest (YAML) → reads stage director skill (MD)
  → uses tools (Python BaseTool) → self-reviews (meta skill)
  → checkpoints (Python utility) → presents to human for approval
```

## Documentation Map (single source of truth — do not fork these)

Each doc owns one concern. When updating documentation, edit the owning file rather than duplicating its content here or elsewhere:

| File | Owns |
|------|------|
| [`AGENT_GUIDE.md`](AGENT_GUIDE.md) | The agent behavior contract: pipeline selection, stage execution, checkpoints, decision logging, communication rules |
| [`PROJECT_CONTEXT.md`](PROJECT_CONTEXT.md) | Condensed architecture + key-files reference shared by all agent tools (Claude, Codex, Cursor, Copilot) |
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | Full technical deep-dive: repository layout, core principles, data flow |
| [`docs/PROVIDERS.md`](docs/PROVIDERS.md) | Provider setup, pricing, free tiers for every integration |
| [`CONTRIBUTING.md`](CONTRIBUTING.md) | Contribution terms (license, PR expectations) |
| `skills/INDEX.md` | Index of Layer 2 (OpenMontage-specific) skills |
| `CLAUDE.md` (this file) | Claude-Code-specific orientation: repo map, dev/test workflow, coding conventions |

`CODEX.md`, `CURSOR.md`, `COPILOT.md`, and `.windsurfrules` are thin pointers to the same two source-of-truth files (`AGENT_GUIDE.md`, `PROJECT_CONTEXT.md`) — keep this file in that spirit rather than growing it into a duplicate.

## Repository Structure

```
OpenMontage/
├── lib/                   # Core runtime infra: config (Pydantic), checkpoints, pipeline/manifest loading,
│                           #   media profiles, corpus/scoring helpers, env loading
├── tools/                 # Python tool implementations, one capability package per family
│   ├── base_tool.py       #   BaseTool — the tool contract every tool inherits
│   ├── tool_registry.py   #   Auto-discovery singleton; source of truth for what's available
│   ├── cost_tracker.py    #   Budget governance (estimate → reserve → reconcile)
│   ├── audio/ video/ graphics/ analysis/ avatar/ enhancement/ subtitle/ character/ capture/ publishers/
├── pipeline_defs/         # Declarative YAML pipeline manifests (stages, tools_available, review_focus, gates)
├── schemas/               # JSON Schemas: artifacts/, checkpoints/, pipelines/, styles/, tools/
├── skills/                # Layer 2 — OpenMontage-specific agent instructions
│   ├── core/ creative/ meta/   #   shared skills (reviewer, checkpoint-protocol, ffmpeg, remotion, ...)
│   └── pipelines/<name>/       #   stage-director skills per pipeline (idea → publish)
├── .agents/skills/        # Layer 3 — vendor/technology skills (Remotion, HyperFrames, GSAP, provider APIs)
├── .claude/skills/        # Claude Code skill files surfaced via the Skill tool
├── styles/                # Visual style playbooks (YAML) + loader/validator
├── remotion-composer/     # Node.js/React Remotion composition renderer
├── ink-theater/           # Ink Theater / Ink Puppet hand-drawn animation engine
├── backlot/                # Backlot — local "living storyboard" board server
├── tests/                 # contracts/, qa/, eval/, pipelines/, tools/, lib/, styles/, backlot/
├── docs/                  # Architecture, providers, PR review guide, platform notes
└── config.yaml             # Global runtime configuration
```

Every production run writes to a gitignored `projects/<project-id>/` workspace (artifacts, assets, renders, checkpoints) — never to the repo root or cwd. See AGENT_GUIDE.md's "Project Directory Convention."

## Development Workflow

**Requirements:** Python ≥ 3.10 (`.python-version` pins 3.10), Node.js (≥ 22 for HyperFrames) for `remotion-composer/`, `ffmpeg` on PATH.

```bash
make setup          # create venv, install requirements.txt, install remotion-composer (npm),
                     #   install Piper (offline TTS), cache-warm HyperFrames via npx, create .env
make install-dev     # + pytest, pytest-asyncio, httpx
make install-gpu     # + local diffusion/video deps (diffusers, transformers, accelerate)

make test            # full pytest suite (tests/ -v)
make test-contracts   # contract tests only — no API keys required (tests/contracts/)
make lint            # py_compile sanity check on core modules
make preflight       # print the live tool/provider capability envelope
make demo            # render zero-key demo videos (Remotion-only, no API keys)
make hyperframes-doctor   # validate the HyperFrames runtime (node/ffmpeg/npx)
make clean            # remove __pycache__ / *.pyc (skips venv)
```

`requirements.txt` is core deps; `requirements-dev.txt` adds test tooling; `requirements-gpu.txt` adds local-GPU generation deps. API keys are all optional and go in `.env` (copied from `.env.example` by `make setup`) — more keys unlock more tools, but the core pipeline runs with zero keys via local/offline paths (FFmpeg, Piper TTS, Remotion).

Tests are organized by concern under `tests/`: `contracts/` (schema/tool-contract validation, no network), `qa/` (tool-by-tool output inspection), `eval/`, and mirrors of the source tree (`tools/`, `lib/`, `pipelines/`, `styles/`, `backlot/`). Run a subset the normal pytest way, e.g. `pytest tests/tools/ -v`.

## Key Conventions

- **No hardcoded tool/provider lists.** Discover everything through `tools.tool_registry.registry` (`.discover()`, then `.capability_catalog()`, `.provider_catalog()`, `.provider_menu_summary()`, `.support_envelope()`). Skills and docs describe *how* to use tools; the registry is the only source of truth for *what's available right now*.
- **Every tool inherits `tools/base_tool.py:BaseTool`**, sets its full contract (name, version, tier, capability, provider, supports, fallback_tools, agent_skills, install_instructions, ...), lives in the matching capability package (`tools/audio/`, `tools/video/`, ...), and is called via `.execute(params_dict)` → `ToolResult` — never `.run()`.
- **Tool class naming: PascalCase, no "Tool" suffix** (`MusicGen`, not `MusicGenTool`; `VideoCompose`, not `VideoComposeTool`). Check with `grep "^class " tools/<path>.py` when unsure.
- **Selector + provider pattern** for multi-provider capabilities: a capability router (`tts_selector`, `image_selector`, `video_selector`) plus one concrete tool per real provider. Adding a provider tool makes it available through the selector automatically — no selector code changes.
- **Pipelines are declarative YAML** in `pipeline_defs/`, validated against `schemas/pipelines/pipeline_manifest.schema.json`. Each stage names its director skill, `tools_available`, `review_focus`, `success_criteria`, and `human_approval_default`. A new pipeline needs a manifest plus stage director skills in `skills/pipelines/<name>/` (idea → publish), not new Python control flow.
- **Canonical artifacts are schema-validated**, in `schemas/artifacts/` (`brief`, `script`, `scene_plan`, `asset_manifest`, `edit_decisions`, `render_report`, `publish_log`, ...). Checkpoints (`lib/checkpoint.py`, schema in `schemas/checkpoints/`) persist stage state to `projects/<id>/checkpoint_<stage>.json`; a gated stage cannot be written `completed` without `human_approved=True`.
- **No orchestration logic in Python.** Creative decisions, review policy, and checkpoint/approval policy belong in skills (Markdown) and manifests (YAML), not in `lib/` or `tools/` code. If you find yourself adding a decision tree to a `.py` file that a skill should own, move it.
- **Style playbooks** (`styles/*.yaml`, schema `schemas/styles/playbook.schema.json`) define visual language (typography, motion, color, audio constraints) per aesthetic, loaded via `styles/playbook_loader.py`.
- **License:** AGPLv3. New/changed behavior should come with tests (`CONTRIBUTING.md`); no CLA/DCO required.

## Adding to the Codebase

- **New tool:** subclass `BaseTool` in the right `tools/<family>/` package, set the full contract, implement `execute()`, add a `schemas/tools/` entry if I/O is complex. Registry discovery is automatic.
- **New pipeline:** manifest in `pipeline_defs/` + stage director skills in `skills/pipelines/<name>/` + contract tests in `tests/contracts/`.
- **New Layer 3 (vendor) knowledge:** `.agents/skills/`, referenced from a tool's `agent_skills` field.
- **New Layer 2 (project convention) knowledge:** `skills/core/`, `skills/creative/`, or `skills/meta/`, registered in `skills/INDEX.md`.

When a task is ambiguous about whether it's "build/fix the software" vs. "produce a video," default to this file for the former and `AGENT_GUIDE.md` for the latter — most real requests from a repository maintainer or contributor are the former.
