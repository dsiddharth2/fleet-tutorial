# TUI Calculator — Plan Review

**Reviewer:** lf-reviewer  
**Date:** 2026-04-29  
**Verdict:** APPROVED

> See the recent git history of this file to understand the context of this review.

---

## 1. Done Criteria — PASS

Every phase has explicit acceptance criteria that are testable and objective. Phase 1 specifies `pip install` and `pytest` exit conditions. Phase 2 lists specific test cases and a security invariant (non-whitelisted nodes raise `InvalidExpression`). Phase 3 defines a deque overflow invariant. Phase 5 enumerates specific user-facing behaviors (button clicks, keyboard input, error display). Phase 6 lists 13 named integration test scenarios. No phase leaves the implementer guessing when "done" is.

---

## 2. Cohesion and Coupling — PASS

Each phase has a single clear responsibility. Phase 2 (calculator) and Phase 3 (history) are pure logic with zero TUI dependencies — they can be developed and tested in isolation. Phase 4 (widgets) is purely presentational. Phase 5 is the integration point, which is the right place for coupling. The only cross-phase data contract is `HistoryEntry`, which is a frozen dataclass — minimal surface area. The plan correctly identifies Phases 2 and 3 as parallelizable.

---

## 3. Shared Interfaces in Earliest Tasks — PASS

Phase 2 establishes the core abstractions: `safe_eval()`, `format_result()`, and the `CalculatorError` exception hierarchy. Phase 3 establishes `HistoryEntry` and `HistoryStore`. These are the interfaces consumed by every later phase. Phases 4, 5, and 6 build on top of them without introducing new foundational types.

---

## 4. Riskiest Assumption Validated Early — PASS with NOTE

The riskiest technical assumption is that an AST-whitelist approach can safely evaluate arbitrary user expressions without `eval()`. This is validated in Phase 2 with explicit security test cases (`__import__`, `(1).__class__`, `True+1`). Phase 1 validates the next riskiest assumption — that `textual` installs and launches on the target platform.

**NOTE:** Phase 1 could be strengthened by verifying that Textual renders a basic widget (e.g., a `Static` with text), not just a blank app. This would catch display/terminal compatibility issues one phase earlier. This is a minor suggestion, not a blocker.

---

## 5. Later Tasks Reuse Early Abstractions (DRY) — PASS

Phase 5 consumes `safe_eval`, `format_result`, `CalculatorError` (from Phase 2) and `HistoryStore` (from Phase 3). Phase 4's `HistoryPanel` consumes `HistoryEntry` from Phase 3. Phase 6 reuses the full `CalculatorApp` via Textual's `run_test()` pilot. No logic is duplicated across phases.

---

## 6. Verify Checkpoints — PASS with NOTE

Each phase ends with acceptance criteria that function as verify gates. Phases 2 and 3 include their own unit tests, which is strong. Phase 6 is a dedicated integration-test phase covering the assembled application.

**NOTE:** The plan does not include explicit VERIFY checkpoint phases (e.g., "VERIFY: run all tests, confirm green"). The per-phase acceptance criteria serve this purpose implicitly, but an implementer could skip verification until Phase 6. Consider adding a brief "run `pytest tests/ -v` — must be green" checkpoint after Phase 3 completes (before starting widget work) to catch regressions early. Phase 4's acceptance criteria are the weakest of all phases — "importing doesn't error" and visual inspection — consider adding a minimal smoke test (e.g., instantiate each widget in a test to verify `compose()` doesn't raise).

---

## 7. Each Task Completable in One Session — PASS

Phase 1 (scaffolding) is ~30 minutes. Phase 2 (calculator + tests) is the largest at roughly 1-2 hours but is well-scoped with clear test cases. Phase 3 is small (~30 min). Phase 4 (widgets + CSS) is moderate. Phases 5-7 are each one-session tasks. No phase requires context that would be lost across sessions.

---

## 8. Dependencies Satisfied in Order — PASS

The dependency graph is correct and explicitly documented:
- Phase 1 is a prerequisite for all others (project structure)
- Phases 2 and 3 are independent and parallelizable
- Phase 4 depends on Phase 3 (for `HistoryEntry`) — correct
- Phase 5 depends on 2 + 3 + 4 — correct
- Phases 6 and 7 are sequential after 5

No phase references artifacts from a later phase.

---

## 9. Vague Tasks — PASS with NOTE

The plan is unusually specific — function signatures, AST node lists, button layouts, CSS selectors, and test case tables are all spelled out.

**NOTE:** Two minor areas of ambiguity:
1. Phase 5's "user-friendly error messages" — the plan doesn't specify the exact error text. Two developers might show "Error" vs. "Division by zero" vs. "Cannot divide by zero." Consider specifying the error message format (e.g., display the exception's string representation).
2. Phase 4's visual verification ("Buttons arrange in 5-row x 4-column grid visually") — this is inherently subjective. The integration tests in Phase 6 partially address this by testing button functionality, but grid layout correctness remains a visual-only check.

---

## 10. Hidden Dependencies — PASS

Phase 6 modifies `pyproject.toml` to add `asyncio_mode = "auto"` — this is a cross-cutting change but is documented in the plan and has no impact on earlier phases. No other hidden dependencies detected. The only inter-phase data contracts are `HistoryEntry` (Phase 3 -> 4) and the calculator API (Phase 2 -> 5), both explicitly documented.

---

## 11. Risk Register — PASS with NOTE

The risk register covers 5 risks with concrete mitigations: AST whitelist completeness, runtime CSS errors, keyboard binding conflicts, exponent hang, and floating-point noise. All are real risks with actionable mitigations.

**NOTE — missing risks to consider:**
- **Windows terminal compatibility:** The project runs on Windows 11 (per the working environment). Textual's Windows terminal support has historically lagged behind Unix. Consider adding a risk: "Textual rendering issues on Windows Terminal / ConPTY" with mitigation "test on Windows Terminal early in Phase 1."
- **Long input strings:** No cap on expression length. A user pasting a 100KB expression could cause performance issues in AST parsing. Mitigation: add a max expression length (e.g., 1000 chars) in `safe_eval`.
- **Textual version pinning:** The plan pins `textual >= 0.80` but Textual's API changes frequently across minor versions. Consider pinning to a specific version (e.g., `textual ~= 0.80`) to avoid surprise breakage.

None of these are blockers, but documenting them would strengthen the plan.

---

## 12. Alignment with Requirements — PASS

All 7 functional requirements from `requirements.md` map directly to plan phases:

| Requirement | Covered In |
|-------------|-----------|
| Safe Expression Evaluation | Phase 2 — AST whitelist, no eval(), security tests |
| Button Grid (5x4) | Phase 4 — explicit layout constant |
| Keyboard Input | Phase 5 — key-to-button mapping table |
| Expression Display | Phase 4 — Display widget with expression + result |
| Calculation History (h toggle, 50-cap) | Phases 3 + 4 — HistoryStore(maxlen=50) + HistoryPanel |
| Result Chaining | Phase 5 — `_last_result` logic documented |
| Error Handling | Phase 2 (exceptions) + Phase 5 (display) |

All 4 non-functional requirements (Python >= 3.10, textual >= 0.80, pytest + pilot, pyproject.toml + hatchling) are addressed. The 4 success criteria from requirements.md are a subset of the plan's success checklist.

The plan solves the right problem and doesn't gold-plate beyond what the requirements ask for.

---

## Summary

**Verdict: APPROVED**

The plan is well-structured, specific, and aligned with the requirements. It establishes clean abstractions early (Phases 2-3), builds on them without duplication (Phases 4-5), and validates the result with comprehensive tests (Phase 6). Dependencies are correctly ordered and documented.

**What passed cleanly (8 of 12):** Done criteria, cohesion/coupling, shared interfaces early, DRY reuse, one-session tasks, dependency order, no hidden dependencies, requirements alignment.

**What passed with notes (4 of 12):**
- *Risk validation in Phase 1* — could verify widget rendering, not just blank app launch.
- *Verify checkpoints* — acceptance criteria exist per-phase, but no explicit mid-plan verification gate. Phase 4's acceptance criteria are weak.
- *Vague tasks* — error message format and visual grid verification are slightly ambiguous.
- *Risk register* — exists and is solid, but should add Windows terminal compatibility, input length caps, and version pinning risks.

**No items require changes before implementation can begin.** The notes above are suggestions for strengthening the plan that can be addressed during implementation.
