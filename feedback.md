# TUI Calculator — Code Review (Phase 1)

**Reviewer:** lf-reviewer  
**Date:** 2026-04-29 18:45:00+05:30  
**Verdict:** APPROVED

> See the recent git history of this file to understand the context of this review.

---

## 1. Build & Install — PASS

`pip install -e ".[dev]"` succeeds. The editable install produces `tui-calc==0.1.0` with all dependencies resolved:

- `textual==8.2.4` (satisfies `>=0.80`)
- `pytest==8.4.2` (satisfies `>=7.0`)
- `pytest-asyncio==1.3.0` (satisfies `>=0.21`)
- `textual-dev==1.8.0` (satisfies `>=0.80` — see note in section 5)

The hatchling build backend correctly finds `src/tui_calc` via the `[tool.hatch.build.targets.wheel]` packages directive.

---

## 2. Test Suite — PASS

`python -m pytest tests/ -v` exits with 0 items collected and exit code 5 (no tests found). This matches the Phase 1 acceptance criterion of "pytest exits with 0 tests collected." The test infrastructure is correctly wired — pytest discovers the `tests/` directory and `tests/__init__.py` exists.

---

## 3. Entry Point & Imports — PASS

- `tui-calc` entry point is registered correctly at `tui_calc.app:main`.
- `from tui_calc.app import CalculatorApp, main` succeeds.
- `CalculatorApp` correctly subclasses `textual.app.App`.
- `from tui_calc import __version__` returns `"0.1.0"`, consistent with `pyproject.toml`.
- `app.py` is a clean stub: `CalculatorApp(App)` with `pass` body and `main()` that calls `run()`.
- No `eval()` or `exec()` anywhere in the codebase.

NOTE: The entry point script `tui-calc.exe` installs to a directory not on PATH. This is an environment issue, not a code issue — the entry point registration itself is correct.

---

## 4. .gitignore Encoding — PASS (fixed)

**Previously FAIL.** The `.gitignore` was UTF-16 LE without BOM, causing git to treat it as binary and silently breaking `*.pyc` and `CLAUDE.md` patterns.

**Doer:** fixed in commit `ea54926` — re-encoded `.gitignore` as UTF-8.

**Re-review verification:**

- File is now 90 bytes of clean UTF-8 (was 164 bytes of UTF-16 LE)
- `git check-ignore -v "*.pyc"` → `.gitignore:2:*.pyc` — **matches**
- `git check-ignore -v "CLAUDE.md"` → `.gitignore:8:CLAUDE.md` — **matches**
- `git check-ignore -v "__pycache__/"` → `.gitignore:1:__pycache__/` — **matches**
- `git check-ignore -v ".venv/"` → `.gitignore:3:.venv/` — **matches**
- `git check-ignore -v "foo.egg-info/"` → `.gitignore:5:*.egg-info/` — **matches**

All 8 patterns are present and all tested patterns resolve correctly. Issue is resolved.

---

## 5. Factual References — PASS with NOTE

**pyproject.toml** — all package names, build backend, and entry point syntax are correct.

**NOTE:** The dev dependency `textual-dev>=0.80` uses a version floor borrowed from `textual`. The `textual-dev` package has never published a version 0.80 — its versions are in the 1.x range. This constraint resolves correctly today (since `1.8.0 >= 0.80` is true) but is semantically misleading. A more accurate constraint would be `textual-dev>=1.0` or simply `textual-dev`. This is not blocking — pip resolves it fine — but it should be corrected in a future phase.

**Doer (advisory acknowledged):** `textual-dev>=0.80` version floor is noted. No action taken now — resolves correctly. Will correct to `textual-dev>=1.0` in a future phase cleanup.

**README.md** — links to `https://github.com/Textualize/textual`, which is the correct repository. Content is a placeholder as specified by the plan.

---

## 6. Project Structure — PASS

All files specified in PLAN.md Phase 1 are present:

| File | Status |
|------|--------|
| `pyproject.toml` | ✓ Present, correct |
| `.gitignore` | ✓ Present, UTF-8 encoded, all patterns working |
| `README.md` | ✓ Present, placeholder |
| `src/tui_calc/__init__.py` | ✓ `__version__ = "0.1.0"` |
| `src/tui_calc/app.py` | ✓ `CalculatorApp(App)` stub + `main()` |
| `tests/__init__.py` | ✓ Empty |

No extra files beyond what was planned.

---

## 7. Alignment with PLAN.md Acceptance Criteria

| Criterion | Result |
|-----------|--------|
| `pip install -e ".[dev]"` succeeds | **PASS** |
| `pytest` exits with "0 tests collected" | **PASS** (exit code 5, 0 items) |
| `tui-calc` launches blank Textual app | **PASS** (import chain verified; headless launch not testable but entry point registered correctly) |

---

## 8. Prior Review Context

The plan review (commit `4767c03`) was APPROVED with notes. The initial Phase 1 code review (commit `a72772a`) found one blocking issue: `.gitignore` UTF-16 encoding. The doer fixed this in commit `ea54926` and annotated feedback.md with the fix reference. This re-review confirms the fix is correct and all previously passing checks remain green.

---

## Summary

**Verdict: APPROVED**

Phase 1 scaffolding is complete. All acceptance criteria pass:

- `pip install -e ".[dev]"` installs cleanly with all dependencies resolved
- `pytest tests/ -v` collects 0 items (expected — no tests yet)
- Entry point registered, imports work, `CalculatorApp` subclasses `App`
- `.gitignore` is now UTF-8 with all patterns working correctly (fixed in `ea54926`)
- No `eval()` or `exec()` in codebase
- Project structure matches PLAN.md exactly

**Deferred (non-blocking):** `textual-dev>=0.80` version floor should be corrected to `>=1.0` in a future phase.

Phase 2 (Safe Expression Evaluator) and Phase 3 (History Store) can proceed in parallel.
