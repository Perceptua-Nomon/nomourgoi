---
agent: agent
description: "Review nomon project code for quality, security, and cross-repo consistency. Runs tests and lints, checks IPC schema sync, invokes @sentinel for security audit, produces a structured findings report."
---

# Review: Validate Implementation

Review the following for the nomon robot project:

**Review Target:** ${input:target:What to review (e.g., "Phase 14 implementation", "motor control changes", "full" for a complete sweep)}

## Your Task

1. **Run the test suites** for affected repos and report pass/fail counts:
   ```bash
   cd nomopractic && source "$HOME/.cargo/env" && cargo test 2>&1 | tail -20
   cd nomothetic && source .venv/bin/activate && pytest -v --tb=short 2>&1 | tail -30
   ```

2. **Run all linters and formatters:**
   ```bash
   cd nomopractic && cargo clippy -- -D warnings
   cd nomopractic && cargo fmt --check
   cd nomothetic && source .venv/bin/activate && ruff check . && black --check .
   ```

3. **Read the changed/new code files.** Examine for:
   - Idiomatic patterns and adherence to project conventions
   - Missing or incomplete error handling
   - `unwrap()` / `expect()` outside of tests
   - Missing type hints (Python) or doc comments (Rust)
   - Logic correctness against the original plan

4. **Check IPC schema synchronization:**
   - Read `nomopractic/src/ipc/schema.rs` for method names, param types, result fields
   - Read `nomothetic/docs/hat_ipc_schema.md` and `nomothetic/src/nomothetic/hat.py`
   - Confirm they all agree. Report any discrepancy as a HIGH finding.

5. **Invoke `@sentinel` for a structured security deep-dive:**
   ```
   @sentinel Review the changes to [list affected files] for all threat categories
   ```
   Embed the sentinel report verbatim in your findings.

6. **Produce a structured Review Report:**
   - Test results table (suite, total, passed, failed)
   - Lint results (clean or violations)
   - Findings table (Severity, File, Line, Description, Recommendation)
   - IPC consistency status
   - Security section (from @sentinel — CRITICAL/HIGH reproduced verbatim)
   - Overall verdict: **PASS** / **PASS WITH MINOR ISSUES** / **FAIL**
   - Key actions required (bulleted list, or "None")
