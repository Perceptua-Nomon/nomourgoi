---
agent: agent
description: "End-to-end phase execution: design → build → review loop. Handles both new phases (design from scratch) and planned phases (consistency check). Iterates until review passes."
---

# Phase Build: Design → Build → Review

Execute a complete phase for the nomon robot project.

**Phase:** ${input:phase:Phase name or description (e.g., "Phase 10: Calibration & Configuration" or "add OTA firmware updates")}

---

## Step 1 — Design

Invoke the **design agent** with the following instruction:

> "For **${input:phase}**:
>
> First, read both roadmaps (`nomopractic/docs/roadmap.md` and `nomothetic/docs/roadmap.md`) to determine whether this phase is already planned.
>
> **If the phase is NOT on the roadmap** (new phase): Research the codebase and produce a complete implementation plan. Add the plan to both roadmaps in the appropriate location before handing off.
>
> **If the phase IS already on the roadmap** (existing phase): Perform a consistency check — verify that the planned features align with:
> - The current IPC schema (`nomothetic/docs/hat_ipc_schema.md`)
> - The architecture docs (`nomopractic/docs/architecture.md`, `nomothetic/docs/architecture.md`)
> - Project coding standards and philosophy (`docs/coding-standards.md`)
> - Anything already implemented that the plan depends on
>
> Make any corrections needed to the roadmap or supporting docs so the plan is internally consistent and ready to build."

Wait for the design agent to complete and confirm the plan is ready before proceeding.

---

## Step 2 — Build

Invoke the **build agent** with the following instruction:

> "Implement **${input:phase}** based on the roadmap and supporting docs. Follow the plan exactly.
>
> - Run tests during the build as needed to catch failures early.
> - **Skip the final full validation run** — the review agent will handle that.
> - Once implementation is complete, hand off directly to the review agent."

Wait for the build agent to confirm implementation is complete before proceeding.

---

## Step 3 — Review

Invoke the **review agent** with the following instruction:

> "Review all work completed for **${input:phase}**. Run the full test suites and linters, inspect all new and changed code, verify IPC schema consistency, and apply the security checklist. Produce a complete findings report with severity ratings and recommendations."

After the review agent returns its report:

**Present the full findings report to the user** and ask:

> "The review is complete. Should I pass these findings back to the build agent for implementation?"

---

## Step 4 — Iterate (if user approves)

If the user says **yes**: invoke the **build agent** with the review findings:

> "Address all findings from the latest review of **${input:phase}**. The findings are listed below. Fix each one, skipping the final validation run since the review agent will re-validate.
>
> [paste the findings table here]"

Then return to **Step 3** — invoke the review agent again.

Repeat **Steps 3–4** until either:
- The review agent returns an **overall verdict of PASS** with no remaining required actions, **or**
- The user declines to iterate further.

## Step 5 — Final documentation update

When the phase is fully implemented and reviewed, and all iteration is complete, invoke the **design agent** one final time with this instruction:

> "For **${input:phase}**, update all documentation to reflect the completed work:
>
> - Mark the phase as complete on both roadmaps
> - Add `///` doc comments to all new public items
> - If a significant architectural decision was made, create an ADR in `docs/adr/`
> - If the IPC schema changed, confirm `nomothetic/docs/hat_ipc_schema.md` is updated accordingly"

---

## Completion

When the phase is fully reviewed and passing, confirm to the user:
- Final test counts for both repos
- Lint status
- List of files modified
- That both roadmaps are updated to mark the phase complete
