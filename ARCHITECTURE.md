# Architecture

This document describes the internal architecture of the OpenCode SDD Pipeline.

---

## System Overview

```mermaid
graph TB
    User((User))
    Orch[Orchestrator<br/>Primary Agent]

    subgraph SubAgents[SubAgents — no direct user interaction]
        Triage[Triage Agent<br/>Requirements Intake]
        Spec[Spec Writer Agent<br/>Specification]
        Planner[Planner Agent<br/>Technical Design]
        TaskD[Task Decomposer Agent<br/>Task Breakdown]
        Impl[Implementer Agent<br/>Code Implementation]
        Rev[Reviewer Agent<br/>Code Review]
        Valid[Validator Agent<br/>Validation]
    end

    subgraph Skills[Skills — explicit invocation]
        SDDSkill[sdd-workflow<br/>Main Entry Point]
        FeatInit[feature-init<br/>Directory Init]
        RiskA[risk-assess<br/>Risk Assessment]
        StageG[stage-gate<br/>Stage Check]
        SpecC[spec-check<br/>Spec Quality Audit]
        DiffR[diff-review<br/>Change Consistency Audit]
    end

    User <-->|Conversation| Orch
    Orch -->|Invoke| Triage
    Orch -->|Invoke| Spec
    Orch -->|Invoke| Planner
    Orch -->|Invoke| TaskD
    Orch -->|Invoke| Impl
    Orch -->|Invoke| Rev
    Orch -->|Invoke| Valid
    Orch -.->|Explicit call| FeatInit
    Orch -.->|Explicit call| RiskA
    Orch -.->|Explicit call| StageG
    Orch -.->|Explicit call| SpecC
    Rev -.->|Explicit call| DiffR

    Triage -->|Return artifact + questions| Orch
    Spec -->|Return artifact + questions| Orch
    Planner -->|Return artifact + questions| Orch
    TaskD -->|Return artifact + questions| Orch
    Impl -->|Return results| Orch
    Rev -->|Return review verdict| Orch
    Valid -->|Return validation verdict| Orch
```

---

## Complete Workflow

```mermaid
flowchart TD
    Start([User submits requirement]) --> Init[/feature-init Skill/]
    Init --> Intake[Intake Phase<br/>Triage Agent iterative loop]
    Intake --> IntakeDone{User finalized?}
    IntakeDone -->|No, has feedback| Intake
    IntakeDone -->|Yes| SG1[/stage-gate Skill/]
    SG1 --> Route{Size Routing}

    Route -->|small| SG4[/stage-gate Skill/]
    Route -->|medium / large| SpecPhase[Spec Phase<br/>Spec Writer Agent iterative loop]

    SpecPhase --> SC[/spec-check Skill/]
    SC --> SpecDone{User finalized?}
    SpecDone -->|No| SpecPhase
    SpecDone -->|Yes| SG2[/stage-gate Skill/]
    SG2 --> PlanPhase[Plan Phase<br/>Planner Agent iterative loop]

    PlanPhase --> PlanDone{User finalized?}
    PlanDone -->|No| PlanPhase
    PlanDone -->|Yes| SG3[/stage-gate Skill/]
    SG3 --> TaskPhase[Tasks Phase<br/>Task Decomposer Agent iterative loop]

    TaskPhase --> TaskDone{User finalized?}
    TaskDone -->|No| TaskPhase
    TaskDone -->|Yes| SG4

    SG4[/stage-gate Skill/] --> Checkpoint[/risk-assess Skill/]
    Checkpoint --> RiskLevel{Risk Level}
    RiskLevel -->|Low| ImplPhase
    RiskLevel -->|Medium| UserConfirm{User confirms<br/>checkpoint?}
    RiskLevel -->|High| ForceReview[Force stop<br/>User reviews plan + tasks]
    UserConfirm -->|Confirmed| ImplPhase
    ForceReview -->|Approved| ImplPhase

    ImplPhase[Implement Phase<br/>Implementer Agent] --> ReviewPhase[Review Phase<br/>Reviewer Agent<br/>calls diff-review Skill]
    ReviewPhase --> ReviewResult{Review Verdict}
    ReviewResult -->|Pass| ValidPhase
    ReviewResult -->|Needs changes| ImplPhase

    ValidPhase[Validate Phase<br/>Validator Agent] --> ValidResult{Validation Verdict}
    ValidResult -->|Pass| IsLarge{Is large task?}
    ValidResult -->|Conditional pass| IsLarge
    ValidResult -->|Fail| ImplPhase

    IsLarge -->|Yes| Report[Generate 06-report.md]
    IsLarge -->|No| Done([Workflow Complete])
    Report --> Done
```

---

## Iterative Loop Pattern

The Intake, Spec, Plan, and Tasks phases all follow the same **iterative loop** pattern. The SubAgent writes/updates an artifact and returns clarification questions on each invocation. The Orchestrator relays between the SubAgent and the user until both sides have no remaining questions.

```mermaid
sequenceDiagram
    participant U as User
    participant O as Orchestrator<br/>(Primary Agent)
    participant S as SubAgent<br/>(e.g. Triage)

    Note over O: Phase begins
    O->>S: First call: pass requirements/context
    S->>S: Explore code + write artifact
    S-->>O: Return: artifact summary + clarification questions

    O->>U: Present artifact summary + questions
    U-->>O: Answers + modification requests

    loop Iterate until finalized
        O->>S: Iteration call: pass user answers + modifications
        S->>S: Update artifact
        S-->>O: Return: update summary + new questions (if any)
        O->>U: Present updated artifact + new questions (if any)
        U-->>O: Answers / modifications / approve
    end

    Note over O: User approves → call stage-gate skill → next phase
```

**Finalization condition**: The SubAgent has no more questions **AND** the user explicitly confirms. The loop continues as long as either party has questions or change requests.

---

## SubAgent Step-Limit Continuation

When a SubAgent exits due to exceeding its max step count, the Orchestrator **must not take over** — it launches a new SubAgent of the same type to continue.

```mermaid
sequenceDiagram
    participant O as Orchestrator
    participant S1 as SubAgent #1
    participant S2 as SubAgent #2 (continuation)

    O->>S1: Call: pass task
    S1->>S1: Execute task...
    Note over S1: Reached step limit
    S1->>S1: Update implementation-log with progress
    S1-->>O: Exit: return completed + remaining tasks

    O->>O: Read progress, prepare continuation params
    O->>S2: Continuation call: pass remaining tasks + current progress
    S2->>S2: Continue from remaining tasks
    S2-->>O: Return: completion summary
```

---

## Execution-Type SubAgent Interaction

Implementer, Reviewer, and Validator are **execution-type** SubAgents — they don't follow the iterative loop; they execute and return results in one pass.

```mermaid
sequenceDiagram
    participant U as User
    participant O as Orchestrator
    participant I as Implementer Agent
    participant R as Reviewer Agent
    participant V as Validator Agent

    Note over O: Implement Phase
    O->>I: Call: pass feature directory + tasks
    I->>I: Execute tasks sequentially
    I->>I: Continuously update 04-implementation-log.md
    I-->>O: Return: completion summary

    Note over O: Review Phase
    O->>R: Call: pass feature directory
    R->>R: Audit code changes against spec/plan
    R->>R: Call diff-review skill
    R-->>O: Return: review verdict + issue list
    O->>U: Present review results

    alt Review passed
        Note over O: Enter Validate Phase
    else Review failed
        O->>U: Present issues
        Note over O: Return to Implement for fixes
    end

    Note over O: Validate Phase
    O->>V: Call: pass feature directory
    V->>V: Verify each AC + run tests
    V->>V: Write 05-validation.md
    V-->>O: Return: pass/fail verdict
    O->>U: Present validation results
```

---

## SubAgent Reference

| Agent | Type | Responsibility | Output | Steps |
|-------|------|---------------|--------|-------|
| **Triage** | Iterative | Requirements intake, size assessment | `00-intake.md` | 50 |
| **Spec Writer** | Iterative | Functional specification | `01-spec.md` | 50 |
| **Planner** | Iterative | Technical design | `02-plan.md` | 50 |
| **Task Decomposer** | Iterative | Task breakdown (INVEST) | `03-tasks.md` | 50 |
| **Implementer** | Execution | Code implementation | `04-implementation-log.md` + code | 200 |
| **Reviewer** | Execution | Code review (calls `diff-review`) | Review verdict | 50 |
| **Validator** | Execution | AC verification + tests | `05-validation.md` | 50 |

---

## Skill Reference

| Skill | Caller | When | Condition | User-invokable |
|-------|--------|------|-----------|:-:|
| **sdd-workflow** | User | Start SDD workflow | User types `/sdd-workflow` | Yes |
| **feature-init** | Orchestrator / User | Workflow start | None | Yes |
| **stage-gate** | Orchestrator | Before each phase transition | Required at every transition | No |
| **spec-check** | Orchestrator | After Spec Agent output | Size >= medium | No |
| **risk-assess** | Orchestrator | Before Implement phase | After `stage-gate` passes | No |
| **diff-review** | Reviewer Agent | During Review phase | After Implement completes | No |

---

## Size Routing

The Triage Agent assesses task size during Intake, which determines the subsequent path. Users can override the agent's assessment.

```mermaid
flowchart LR
    Intake[Intake Finalized] --> Size{Size Assessment}
    Size -->|"small<br/>bug fix / single file / config"| PathS["intake → implement<br/>→ review → validate"]
    Size -->|"medium<br/>new feature / multi-file"| PathM["intake → spec → plan → tasks<br/>→ implement → review → validate"]
    Size -->|"large<br/>architecture change / cross-module"| PathL["Full path + forced checkpoint<br/>+ manual review + report"]
```

---

## Stage Gate Mechanism

Before each phase transition, the Orchestrator explicitly calls the `stage-gate` Skill to verify preconditions. Transitions are blocked if any condition is unmet.

| From | To | Preconditions |
|------|----|--------------|
| Intake | Spec | `00-intake.md` finalized, size >= medium |
| Intake | Implement | `00-intake.md` finalized, size = small |
| Spec | Plan | `01-spec.md` finalized |
| Plan | Tasks | `02-plan.md` finalized |
| Tasks | Implement | `03-tasks.md` finalized + checkpoint passed |
| Implement | Review | `04-implementation-log.md` exists |
| Review | Validate | Reviewer approved or user confirmed |
| Validate (fail) | Implement | Re-enter for fixes |
| Validate (pass) | Report | Large tasks only |

---

## Risk Assessment Checkpoint

Before entering the Implement phase, the Orchestrator calls the `risk-assess` Skill.

```mermaid
flowchart TD
    Tasks[Tasks Finalized] --> SG[/stage-gate Skill/]
    SG --> RA[/risk-assess Skill/]

    RA --> Low{Low Risk}
    RA --> Med{Medium Risk}
    RA --> High{High Risk}

    Low -->|Auto-continue| Impl[Enter Implement]
    Med -->|Generate checkpoint summary| UserMed[User confirms to continue]
    High -->|Force stop| UserHigh[User reviews plan + tasks<br/>manual approval]

    UserMed --> Impl
    UserHigh --> Impl
```

**Assessment dimensions**: number of changed files, core module impact, data mutations, external dependencies, irreversible operations.

---

## Key Constraints

- SubAgents **cannot** interact with the user directly — all communication goes through the Orchestrator
- The Orchestrator **cannot** modify SubAgent-generated artifacts directly — it must pass modifications back to the SubAgent
- The Orchestrator **cannot** execute SubAgent work itself — when a SubAgent exceeds its step limit, a new SubAgent of the same type must be launched
- All Skills are invoked **explicitly** within the workflow

---

## Directory Layout

```
.opencode/
├── agents/                         # Agent definitions
│   ├── orchestrator.md             # Primary Agent
│   ├── triage.md                   # Requirements intake SubAgent
│   ├── spec-writer.md              # Specification SubAgent
│   ├── planner.md                  # Technical design SubAgent
│   ├── task-decomposer.md          # Task breakdown SubAgent
│   ├── implementer.md              # Implementation SubAgent (steps: 200)
│   ├── reviewer.md                 # Code review SubAgent
│   └── validator.md                # Validation SubAgent
├── skills/                         # Skill definitions
│   ├── sdd-workflow/SKILL.md       # Main entry point (user-invokable)
│   ├── feature-init/SKILL.md       # Directory init (user-invokable)
│   ├── risk-assess/SKILL.md        # Risk assessment (Orchestrator)
│   ├── stage-gate/SKILL.md         # Stage check (Orchestrator)
│   ├── spec-check/SKILL.md         # Spec quality audit (Orchestrator)
│   └── diff-review/SKILL.md        # Change consistency audit (Reviewer)
├── templates/                      # Artifact templates (00–06)
├── docs/                           # Research & proposal docs
├── AGENTS.md                       # Project-level contract
└── README.md

features/
└── {id}-{name}/                    # Per-feature artifact directory
    ├── 00-intake.md
    ├── 01-spec.md
    ├── 02-plan.md
    ├── 03-tasks.md
    ├── 04-implementation-log.md
    ├── 05-validation.md
    └── 06-report.md                # Optional
```
