# nomourgoi

Development infrastructure for the nomon robot fleet.

Contains AI agents, workflow skills, prompt templates, shared standards, and project-wide documentation used across all nomon repositories.

## Agent Ecosystem

nomourgoi provides **8 specialized agents** and **1 workflow skill** for nomon development.

### Workflow Skill

| Skill | Purpose |
|-------|---------|
| `iterate` (`.github/skills/iterate/SKILL.md`) | Orchestrates the full design → build → review → fix loop end-to-end |

### Core Agents

| Agent | Invocation | Role |
|-------|-----------|------|
| design | `@design` | Strategic planner — produces phase plans, IPC contracts, cross-repo impact analysis. Consults specialists for feasibility. |
| build | `@build` | Meta-coordinator — delegates to domain specialists, falls back to direct implementation for multi-repo tasks. |
| review | `@review` | Quality gatekeeper — runs tests, lints, IPC consistency, invokes `@sentinel` for security. |

### Domain Specialists

| Agent | Invocation | Domain |
|-------|-----------|--------|
| rustsmith | `@rustsmith` | Rust / nomopractic — HAT drivers, IPC protocol, tokio async, clippy-clean code |
| pythoneer | `@pythoneer` | Python / nomothetic — FastAPI, pytest, HAT IPC client, black/ruff |
| uicraft | `@uicraft` | TypeScript/Expo / nomotactic — lightweight UI, expo-router, HTTPS transport |
| schematist | `@schematist` | ArcadeDB / nomographic — schema migrations, graph design, central/local split |
| sentinel | `@sentinel` | Security auditor (all repos) — read-only, structured threat analysis |

## Usage

### Fully Automated
```
invoke the iterate skill
```

### Step-by-Step
```bash
# Plan a feature
@design Propose a plan for Phase 14: Encoder velocity feedback

# Implement it (delegates to @rustsmith, @pythoneer, etc. internally)
@build Implement Phase 14 per the plan

# Review (invokes @sentinel automatically)
@review Validate Phase 14 against quality and security standards
```

### Direct Specialist Invocation
```bash
@rustsmith Implement the HC-SR04 temperature-compensated distance filter
@pythoneer Add /api/encoder endpoint and pytest coverage
@uicraft Add velocity gauge component to the main screen
@schematist Create V4 central migration to add EncoderReading vertex
@sentinel Audit the authentication and pairing endpoints
```

## Delegation Map

```
iterate skill ──► @design ──► @build ──► @review
                               │              │
                               ▼              ▼
                        @rustsmith        @sentinel
                        @pythoneer
                        @uicraft
                        @schematist
```

## Repository Layout

```
.github/
  agents/          Agent definition files (.agent.md)
  skills/iterate/  Iterate workflow skill (SKILL.md)
  prompts/         Structured prompt templates (.prompt.md)
  .copilot-instructions.md  Project-wide Copilot instructions
docs/
  project-context.md    Consolidated architecture reference
  coding-standards.md   Unified style guide (Rust, Python, TypeScript)
  security-checklist.md Security requirements and threat model
```

## Documentation Reference

| Path | Purpose |
|------|---------|
| `docs/project-context.md` | Architecture reference for all repos — hardware, IPC, module maps |
| `docs/coding-standards.md` | Unified style guide (Rust, Python, TypeScript) |
| `docs/security-checklist.md` | Security threat model and review checklist |
| `.github/.copilot-instructions.md` | Full agent ecosystem documentation and development workflow |
