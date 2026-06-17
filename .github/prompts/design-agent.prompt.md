---
agent: agent
description: "Propose a detailed implementation plan for a new nomon feature or phase. Provide the feature description and target repos. The design agent will consult specialists (@rustsmith, @pythoneer, @uicraft, @schematist, @sentinel) for feasibility validation."
---

# Design: Propose a Phase Plan

Analyze the nomon project and produce a complete implementation plan for:

**Feature / Phase:** ${input:feature:Describe the feature or phase to design (e.g., "Phase 14: encoder velocity feedback")}

## Your Task

1. **Explore the codebase first.** Use the Explore subagent to read:
   - Roadmaps: `nomopractic/docs/roadmap.md` and `nomothetic/docs/roadmap.md`
   - Any related existing modules (grep for relevant terms)
   - The IPC schema if the feature touches the HAT interface: `nomothetic/docs/hat_ipc_schema.md`
   - Architecture docs if the feature has broad structural implications

2. **Consult specialists for feasibility** before committing to the plan:
   - `@rustsmith` — hardware constraints, IPC protocol feasibility
   - `@pythoneer` — API patterns, Python service limitations
   - `@uicraft` — UI complexity, bundle impact
   - `@schematist` — migration feasibility, graph schema design
   - `@sentinel` — security implications of new API surfaces

3. **Identify all cross-repo impacts.** Determine whether this feature requires:
   - New IPC methods (both repos)
   - New hardware constants or GPIO pins
   - New Python API endpoints
   - New database schema elements (nomographic)
   - New UI screens or components (nomotactic)
   - Documentation or ADR additions

4. **Produce the Phase Plan** following your standard format:
   - Objective (one sentence)
   - Scope (repos, key files)
   - Numbered implementation steps (repo → file → action → verification)
   - Cross-repo impacts
   - Risks and mitigations
   - Verification commands

5. **Flag blockers.** If anything needs design decisions before implementation can start, say so explicitly.
