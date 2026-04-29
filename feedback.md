# TUI Calculator — Code Review (Phase 1)

**Reviewer:** lf-reviewer  
**Date:** 2026-04-29 18:15:00+05:30  
**Verdict:** CHANGES NEEDED

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

## 4. .gitignore Encoding — FAIL

**The `.gitignore` file is UTF-16 LE without BOM.** Git treats it as a binary file (`git diff` reports "Binary files differ"). This causes pattern-matching failures:

- `git check-ignore -v "*.pyc"` → **no match** (exit code 1)
- `git check-ignore -v "CLAUDE.md"` → **no match** (exit code 1)
- `CLAUDE.md` appears as untracked in `git status`, confirming the ignore pattern is broken

Some patterns appear to match by coincidence due to null-byte alignment (`__pycache__/`, `.venv/`, `dist/`), but this is unreliable.

**Raw bytes confirm the issue:** every ASCII character is followed by a `\x00` byte (e.g., `b'_\x00_\x00p\x00y\x00c\x00...'`), with no BOM prefix.

**Doer:** fixed in commit below — re-encoded `.gitignore` as UTF-8. `git check-ignore -v "CLAUDE.md"` and `git check-ignore -v "__pycache__/"` both match correctly after the fix.

**Required fix:** Re-encode `.gitignore` as UTF-8. The content is correct when decoded — the 8 patterns match what PLAN.md specifies:

```
__pycache__/
*.pyc
.venv/
dist/
*.egg-info/
.pytest_cache/
.mypy_cache/
CLAUDE.md
```

The commit message for task 1.2 says "fixed encoding" but the file is still UTF-16 LE.

---

## 5. Factual References — PASS with NOTE

**pyproject.toml** — all package names, build backend, and entry point syntax are correct.

**NOTE:** The dev dependency `textual-dev>=0.80` uses a version floor borrowed from `textual`. The `textual-dev` package has never published a version 0.80 — its versions are in the 1.x range. This constraint resolves correctly today (since `1.8.0 >= 0.80` is true) but is semantically misleading. A more accurate constraint would be `textual-dev>=1.0` or simply `textual-dev`. This is not blocking — pip resolves it fine — but it should be corrected in a future phase.

**README.md** — links to `https://github.com/Textualize/textual`, which is the correct repository. Content is a placeholder as specified by the plan.

---

## 6. Project Structure — PASS

All files specified in PLAN.md Phase 1 are present:

| File | Status |
|------|--------|
| `pyproject.toml` | ✓ Present, correct |
| `.gitignore` | ✓ Present, **encoding broken** (see section 4) |
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

The plan review (commit `4767c03`) was APPROVED with notes. The Phase 1 plan review noted that Phase 1 could be strengthened by verifying widget rendering, not just a blank app — the implementer kept the stub as `pass` which is exactly what the plan specified. The plan review also flagged Windows terminal compatibility as a risk — this is not yet tested but is appropriate for later phases.

---

## Summary

**Verdict: CHANGES NEEDED**

Phase 1 scaffolding is nearly complete. The build system, entry point, imports, test infrastructure, and project structure all work correctly. One blocking issue must be fixed:

1. **BLOCKING — `.gitignore` encoding:** File is UTF-16 LE without BOM. Git cannot parse it, causing `*.pyc` and `CLAUDE.md` ignore patterns to silently fail. Must be re-encoded as UTF-8.

Non-blocking note for future phases:

- `textual-dev>=0.80` version floor is semantically wrong (should be `>=1.0` or unconstrained) but resolves correctly today.

**Doer (advisory acknowledged):** `textual-dev>=0.80` version floor is noted. No action taken now — resolves correctly. Will correct to `textual-dev>=1.0` in a future phase cleanup.
