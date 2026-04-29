# TUI Calculator — Requirements

## Objective

Build a Python terminal calculator with an interactive TUI (Terminal User Interface) using the `textual` library. No GUI — runs entirely in the terminal with mouse and keyboard support, a button grid, expression display, and calculation history.

## Functional Requirements

1. **Safe Expression Evaluation** — AST-based whitelist parser. No `eval()` or `exec()`. Supports `+`, `-`, `*`, `/`, `**`, parentheses, unary operators. Rejects all non-whitelisted AST nodes.
2. **Button Grid** — 5-row x 4-column layout: digits 0-9, operators, parentheses, decimal point, equals, CE (backspace), C (clear).
3. **Keyboard Input** — Full keyboard support mirroring button grid: digits, operators, Enter=evaluate, Backspace=CE, Delete=C, Escape=clear all.
4. **Expression Display** — Shows current expression being built and the result after evaluation. Right-aligned.
5. **Calculation History** — Toggleable panel (press `h`) showing past calculations newest-first. Capped at 50 entries.
6. **Result Chaining** — After evaluating, typing an operator chains from the last result; typing a digit starts fresh.
7. **Error Handling** — Division by zero and invalid expressions show user-friendly error messages, not crashes.

## Non-Functional Requirements

- Python >= 3.10
- `textual >= 0.80` as TUI framework
- `pytest` + `pytest-asyncio` for testing, including headless TUI tests via Textual's `pilot`
- Modern packaging: `pyproject.toml` + `hatchling`
- No security vulnerabilities: exponent cap at 1000, no `eval()`/`exec()`, AST whitelist approach

## Success Criteria

- `pip install -e ".[dev]"` works cleanly
- `tui-calc` launches a functional calculator TUI
- All unit + integration tests pass
- No `eval()` or `exec()` in codebase
