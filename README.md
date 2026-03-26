# OpenCode SDD Pipeline

A **Spec-Driven Development (SDD)** agentic workflow for [OpenCode](https://opencode.ai).

> Spec is the hub. Artifacts are the source of truth. Process is the constraint.

The pipeline orchestrates 7 specialized AI agents through a stage-gated workflow — from requirements intake to code validation — ensuring every feature is spec-anchored, artifact-driven, and implementation-bounded.

---

## Features

- **Stage-gated workflow** — phases progress in order; each transition is guarded by automated precondition checks
- **Multi-agent collaboration** — 7 SubAgents (Triage, Spec Writer, Planner, Task Decomposer, Implementer, Reviewer, Validator) coordinated by an Orchestrator
- **Iterative refinement** — requirement, spec, plan, and task phases loop with user feedback until explicitly finalized
- **Risk-aware execution** — automated risk assessment before implementation; medium/high risk requires human approval
- **Size-based routing** — small tasks skip spec/plan/task phases; large tasks enforce checkpoints and reports
- **Artifact-first** — every phase produces a versioned markdown artifact, never relying on session memory

## Quick Start

1. Copy the `agents/`, `skills/`, and `templates/` directories into your project's `.opencode/` folder:

```
your-project/
└── .opencode/
    ├── agents/
    ├── skills/
    └── templates/
```

2. Launch OpenCode:

```bash
opencode
```

3. Start the SDD workflow:

```
/sdd-workflow Implement a user search feature with keyword matching and result sorting
```

Or initialize a feature directory manually:

```
/feature-init 001 user-search
```

That's it. The Orchestrator agent will guide you through each phase.

---

## How It Works

```
intake → spec → plan → tasks → implement → review → validate → (report)
```

| Size | Path |
|------|------|
| **small** | intake → implement → review → validate |
| **medium** | intake → spec → plan → tasks → implement → review → validate |
| **large** | full path + forced checkpoint + manual review + report |

Each iterative phase (intake, spec, plan, tasks) follows a loop: the SubAgent writes an artifact and raises questions, the Orchestrator relays to the user, and the cycle repeats until finalized. Execution phases (implement, review, validate) run to completion and return results.

For the full architecture, workflow diagrams, and detailed agent/skill reference, see **[ARCHITECTURE.md](ARCHITECTURE.md)**.

---

## Project Structure

```
agents/                        # Agent definitions (copy to .opencode/)
├── orchestrator.md            # Primary Agent — orchestrates the workflow
├── triage.md                  # Requirements intake & size routing
├── spec-writer.md             # Functional specification authoring
├── planner.md                 # Technical design & trade-offs
├── task-decomposer.md         # Task breakdown (INVEST principles)
├── implementer.md             # Code implementation (200 steps)
├── reviewer.md                # Code review against spec/plan
└── validator.md               # Acceptance criteria verification

skills/                        # Skill definitions (copy to .opencode/)
├── sdd-workflow/SKILL.md      # Main entry point
├── feature-init/SKILL.md      # Feature directory scaffolding
├── stage-gate/SKILL.md        # Phase transition guard
├── spec-check/SKILL.md        # Spec quality audit
├── risk-assess/SKILL.md       # Pre-implementation risk assessment
└── diff-review/SKILL.md       # Change-spec consistency audit

templates/                     # Artifact templates (copy to .opencode/)
├── 00-intake.md
├── 01-spec.md
├── 02-plan.md
├── 03-tasks.md
├── 04-implementation-log.md
├── 05-validation.md
└── 06-report.md
```

At runtime, each feature produces its artifacts under `features/{id}-{name}/`.

---

## Core Principles

1. **Artifact-first** — all phase outputs are written to files; nothing depends on session memory
2. **Stage-gated** — phases advance in order; `stage-gate` skill checks preconditions at every transition
3. **User-finalized** — every artifact requires explicit user approval before the next phase begins
4. **Orchestrator-mediated** — SubAgents never interact with the user directly
5. **Implementation-bounded** — code changes are constrained to the task list; no scope creep
6. **Review + Validate** — dual-layer quality assurance before completion

---

## Requirements

- [OpenCode](https://opencode.ai) installed and configured

---

## Contributing

Contributions are welcome! Feel free to open issues or submit pull requests.

---

## License

MIT
