# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Spec-Driven Development (SDD) workflow definition project** for the OpenCode (Claude Code) agentic platform. It contains no application code — only agent definitions, skill specs, artifact templates, and documentation. All documentation is in Chinese.

## How to Use

Entry point: `/sdd-workflow [requirement description]` — launches the Orchestrator agent to drive the full pipeline.

Other user-invocable skill: `/feature-init [feature-id] [feature-name]` — initializes a feature working directory under `features/`.

## Architecture

**Multi-agent orchestration with stage gating.** The Orchestrator (primary agent) delegates work to 7 SubAgents across pipeline stages:

1. **Triage** → `00-intake.md` (requirement capture + scale routing)
2. **Spec Writer** → `01-spec.md` (functional specification)
3. **Planner** → `02-plan.md` (technical solution design)
4. **Task Decomposer** → `03-tasks.md` (INVEST-based task breakdown)
5. **Implementer** → `04-implementation-log.md` + code changes
6. **Reviewer** → review verdict (calls `diff-review` skill)
7. **Validator** → `05-validation.md` (acceptance verification)

**Scale-based routing** determines which stages execute:
- **Small**: intake → implement → review → validate (skips spec/plan/tasks)
- **Medium**: all stages
- **Large**: all stages + forced checkpoint + final `06-report.md`

**Skills** (non-agent logic invoked at specific points):
- `stage-gate` — enforces preconditions before stage transitions (Orchestrator calls)
- `spec-check` — QA for spec documents (Orchestrator calls)
- `risk-assess` — risk evaluation before implementation (Orchestrator calls)
- `diff-review` — validates code changes against spec/tasks (Reviewer calls)

## Key Constraints

- **Orchestrator never edits artifact files** (00–06) — only SubAgents produce artifacts
- **SubAgents never interact with the user directly** — Orchestrator is the sole proxy
- **Reviewer and Validator are read-only** — they never modify code
- **Iterative loop model** for intake/spec/plan/tasks: SubAgent generates → Orchestrator shows user → user gives feedback → loop until approved
- **Execution model** for implement/review/validate: one-way flow, failures loop back to implement only
- **Step limits**: SubAgents have bounded steps (50 for most, 200 for Implementer). If exceeded, Orchestrator launches a new same-type agent to continue — never substitutes itself

## Directory Layout

```
agents/          # 8 SubAgent definitions (markdown with YAML front matter)
skills/          # 6 Skill implementations (each in its own directory with SKILL.md)
templates/       # 7 artifact templates (00-06, use {{feature-name}} placeholder)
docs/            # Research & proposal documents (Chinese)
```

When a feature is initialized, artifacts are created under `features/{id}-{name}/` by copying templates.

## Extending the Project

**New agent**: Create `agents/{name}.md` with YAML front matter defining `description`, `mode`, `steps`, `permissions`, then add the prompt body.

**New skill**: Create `skills/{name}/SKILL.md` with YAML front matter defining `name`, `description`, `user-invocable`, `allowed-tools`, then add the skill logic.

**New template**: Create `templates/{number}-{name}.md` with `{{feature-name}}` placeholders, then update `feature-init` skill to copy it.
