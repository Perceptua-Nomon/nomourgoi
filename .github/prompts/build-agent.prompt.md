---
agent: agent
description: "Implement a feature or phase for the nomon project. Provide the plan document path or a description of what to build. The build agent will delegate to specialists (@rustsmith, @pythoneer, @uicraft, @schematist) as appropriate."
---

# Build: Implement a Feature

Implement the following for the nomon robot project:

**Target:** ${input:target:Describe what to implement, or provide the path to a plan document (e.g., "Phase 14 per docs/phase14.md")}

## Your Task

1. **Read the plan.** If a plan document path was given, read it in full. Otherwise, explore the codebase to understand the current state before writing any code.

2. **Delegate to specialists first:**
   - Rust / nomopractic work → invoke `@rustsmith`
   - Python / nomothetic work → invoke `@pythoneer`
   - TypeScript / nomotactic work → invoke `@uicraft`
   - ArcadeDB / nomographic work → invoke `@schematist`
   - For cross-cutting tasks or when delegation overhead is not justified, implement directly.

3. **Create a todo list** with concrete implementation steps. Structure as:
   - Rust type definitions / trait additions (if any) → @rustsmith
   - Core logic implementation
   - IPC handler integration (if adding new methods) → @rustsmith + @pythoneer
   - Python `hat.py` additions (if adding new IPC methods) → @pythoneer
   - Unit tests (Rust `#[cfg(test)]` blocks) → @rustsmith
   - Integration tests
   - Documentation updates (roadmap, ADR if needed)

4. **Implement incrementally.** Write one module at a time. After each module, run the relevant tests and fix any failures before continuing.

5. **Run full validation before declaring done** — unless you are about to invoke the review agent as the final step. If the review agent will be called next, skip this validation entirely; the review agent always starts its own full test and lint pass, so running it twice is redundant.

   When validation is needed:
   ```bash
   # nomopractic
   cd nomopractic && source "$HOME/.cargo/env" && cargo test && cargo clippy -- -D warnings && cargo fmt --check

   # nomothetic
   cd nomothetic && source .venv/bin/activate && pytest && black --check . && ruff check .
   ```

6. **Update documentation:**
   - Add `///` doc comments to all new public items
   - Update the roadmap to mark the phase or feature as complete
   - Create an ADR in `docs/adr/` if a significant architectural decision was made
   - If IPC schema changed, confirm `nomothetic/docs/hat_ipc_schema.md` is updated

7. **Confirm completion.** State: how many tests pass, which files were modified, and that all lints are clean.
