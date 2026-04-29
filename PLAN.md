# Python TUI Calculator - Implementation Plan

**Project Location:** `C:\2_WorkSpace\Training_QA\Session_3\fleet-tutorial`  
**Date:** 2026-04-29

## Context

Create a Python terminal calculator with an interactive TUI (Terminal User Interface) using the `textual` library. No GUI — runs entirely in the terminal with mouse and keyboard support, a button grid, expression display, and calculation history.

---

## Tech Stack

| Component | Choice | Why |
|-----------|--------|-----|
| TUI Framework | `textual` (>=0.80) | CSS-like layout, built-in widgets, keyboard/mouse support, testable with `pilot` |
| Expression Eval | `ast` module (whitelist) | Safe — no `eval()` of arbitrary code, handles parentheses natively |
| Testing | `pytest` + `pytest-asyncio` | Standard; `textual` pilot integration for headless UI tests |
| Build | `pyproject.toml` + `hatchling` | Modern Python packaging |
| Python | >= 3.10 | For `match` statements and modern `ast` features |

---

## Project Structure

```
fleet-tutorial/
|-- pyproject.toml
|-- README.md
|-- .gitignore
|-- src/
|   |-- tui_calc/
|       |-- __init__.py
|       |-- app.py              # Textual App subclass, main entry point
|       |-- calculator.py       # Safe AST-based expression evaluator
|       |-- widgets.py          # Display, ButtonGrid, HistoryPanel widgets
|       |-- history.py          # Calculation history store (deque-backed)
|       |-- styles.tcss         # Textual CSS for layout and colors
|-- tests/
|   |-- __init__.py
|   |-- test_calculator.py
|   |-- test_history.py
|   |-- test_app.py
```

---

## Dependency Graph

```
Phase 1 (Scaffolding)
  |
  +---> Phase 2 (Calculator) --+
  |                             |
  +---> Phase 3 (History) ------+---> Phase 4 (Widgets) ---> Phase 5 (App) ---> Phase 6 (Tests) ---> Phase 7 (Polish)
```

Phases 2 and 3 are independent and can run in parallel. Phase 4 depends on Phase 3. Phase 5 depends on 2+3+4. Phase 6 depends on 5. Phase 7 depends on 6.

---

## Phase 1: Project Scaffolding

**Goal:** Establish directory structure, dependency management, and dev tooling so `pip install -e .` works and `pytest` runs with zero tests.

**Files to create:**

| File | Purpose |
|------|---------|
| `pyproject.toml` | Metadata, `textual>=0.80` dep, `[project.scripts] tui-calc = "tui_calc.app:main"`, optional dev deps (`pytest`, `pytest-asyncio`, `textual-dev`) |
| `.gitignore` | `__pycache__/`, `*.pyc`, `.venv/`, `dist/`, `*.egg-info/`, `.pytest_cache/`, `.mypy_cache/` |
| `README.md` | Placeholder with project name and one-liner |
| `src/tui_calc/__init__.py` | `__version__ = "0.1.0"` |
| `src/tui_calc/app.py` | Stub: `class CalculatorApp(App): pass` + `def main(): CalculatorApp().run()` |
| `tests/__init__.py` | Empty |

**Acceptance criteria:**
- `pip install -e ".[dev]"` succeeds
- `pytest` exits with "0 tests collected"
- `tui-calc` launches a blank Textual app that quits with Ctrl+C

---

## Phase 2: Safe Expression Evaluator (`calculator.py`)

**Goal:** Implement the core math engine — `safe_eval()` parses an expression into an AST, validates every node against a whitelist, and computes the result without ever calling `eval()`.

**Files to create:**

### `src/tui_calc/calculator.py`

**Custom exceptions:**
- `CalculatorError(Exception)` — base
- `InvalidExpression(CalculatorError)` — disallowed AST nodes or syntax errors
- `DivisionByZeroError(CalculatorError)` — division by zero
- `EvaluationError(CalculatorError)` — overflow, non-finite results

**Allowed AST node whitelist:**
```python
ALLOWED_NODES = (
    ast.Expression, ast.Constant, ast.BinOp, ast.UnaryOp,
    ast.Add, ast.Sub, ast.Mult, ast.Div, ast.Pow,
    ast.USub, ast.UAdd,
)
```

**Functions:**

| Function | Responsibility |
|----------|---------------|
| `_validate_node(node)` | Walk AST with `ast.walk()`, check each node against `ALLOWED_NODES`. For `Constant` nodes, verify value is `int` or `float` (reject strings, booleans, None). Raise `InvalidExpression` on violation. |
| `_eval_node(node)` | Recursive evaluator: handle `Expression` -> `Constant` -> `BinOp` -> `UnaryOp`. For `Div`: check zero denominator. For `Pow`: cap exponent at 1000. Check results for `inf`/`nan`. |
| `safe_eval(expression: str)` | Strip whitespace, parse with `ast.parse(mode="eval")`, validate, evaluate, return result. |
| `format_result(value)` | Display `4.0` as `"4"`, strip trailing zeros, use scientific notation for very large/small values. Round to 10 significant digits to mitigate floating-point noise. |

### `tests/test_calculator.py`

**Test categories:**

| Category | Tests |
|----------|-------|
| Basic arithmetic | `2+3`=5, `10-4`=6, `3*7`=21, `15/4`=3.75 |
| Precedence & parens | `2+3*4`=14, `(2+3)*4`=20, `((1+2)*(3+4))`=21 |
| Exponents | `2**10`=1024, `2**10000` raises `EvaluationError` |
| Unary operators | `-5`=-5, `+3`=3, `--5`=5 |
| Floating point | `3.14*2`=6.28, `0.5+0.5`=1.0 |
| Division by zero | `1/0` raises `DivisionByZeroError` |
| Invalid input | `""` raises `InvalidExpression`, `2++` raises `InvalidExpression` |
| Security | `__import__('os')`, `x+1`, `(1).__class__`, `'hello'`, `True+1` all raise `InvalidExpression` |
| format_result | `4.0`->"4", `3.14`->"3.14", `1e20`->reasonable string |

**Acceptance criteria:**
- `pytest tests/test_calculator.py -v` all green
- Any non-whitelisted AST node raises `InvalidExpression`
- `2**10000` raises error, not a hang

---

## Phase 3: History Store (`history.py`)

**Goal:** Implement `HistoryStore` — records calculation entries with retrieval, clearing, and max-size limit. Fully decoupled from TUI.

**Files to create:**

### `src/tui_calc/history.py`

```python
@dataclasses.dataclass(frozen=True)
class HistoryEntry:
    expression: str     # e.g., "2+3*4"
    result: str         # e.g., "14" (already formatted)
    timestamp: float    # time.time()
```

**`HistoryStore` class:**

| Method | Behavior |
|--------|----------|
| `__init__(maxlen=50)` | Initialize `collections.deque(maxlen=maxlen)` |
| `add(expression, result)` | Create `HistoryEntry`, append, return it |
| `entries()` | Return list newest-first (`reversed(deque)`) |
| `clear()` | Clear the deque |
| `__len__` / `__bool__` | Length and emptiness checks |

### `tests/test_history.py`

| Test | Verifies |
|------|----------|
| `test_add_entry` | `len(store) == 1`, fields correct |
| `test_entries_newest_first` | Add A then B, `entries()[0]` is B |
| `test_max_length` | `maxlen=3`, add 5, only 3 remain (newest) |
| `test_clear` | After `clear()`, `len == 0` |
| `test_empty_store` | `len == 0`, `bool == False`, `entries() == []` |
| `test_timestamp_set` | Timestamp is positive float near `time.time()` |

**Acceptance criteria:**
- `pytest tests/test_history.py -v` all green
- `HistoryStore(maxlen=3)` with 100 additions never exceeds 3 entries

---

## Phase 4: Textual Widgets (`widgets.py` + `styles.tcss`)

**Goal:** Build the three visual components as reusable Textual widgets plus the CSS stylesheet.

**Files to create:**

### `src/tui_calc/widgets.py`

**`Display(Widget)`:**
- `compose()`: yields `Static` for expression + `Static` for result inside a `Vertical` container
- `update_expression(text)` / `update_result(text)` / `clear()` methods

**`ButtonGrid(Widget)`:**
- Button layout constant:
  ```
  ("(", "paren"), (")", "paren"), ("CE", "clear"), ("C", "clear"),
  ("7", "digit"), ("8", "digit"), ("9", "digit"), ("/", "operator"),
  ("4", "digit"), ("5", "digit"), ("6", "digit"), ("*", "operator"),
  ("1", "digit"), ("2", "digit"), ("3", "digit"), ("-", "operator"),
  ("0", "digit"), (".", "digit"), ("=", "equals"), ("+", "operator"),
  ```
- `compose()`: yield `Button` widgets in grid container with `id=btn-{sanitized}` and `classes=btn btn-{category}`
- Posts custom `ButtonPressed(label: str)` message that bubbles up to App

**`HistoryPanel(Widget)`:**
- Contains `VerticalScroll` with `Static` entries
- `refresh_entries(entries: list[HistoryEntry])`: re-render entries
- `toggle()`: add/remove `visible` CSS class
- Initially hidden

### `src/tui_calc/styles.tcss`

| Selector | Styling |
|----------|---------|
| `Screen` | Dark background, vertical layout |
| `#display-container` | Dock top, fixed height 3-4 lines, right-aligned |
| `#expression-line` | Dimmer color for expression |
| `#result-line` | Bold for result |
| `#button-grid` | CSS grid, 4 columns, gap 1 |
| `.btn-digit` | Light gray on dark |
| `.btn-operator` | Orange/amber |
| `.btn-equals` | Green/teal |
| `.btn-clear` | Red-tinted |
| `.btn-paren` | Blue-tinted |
| `#history-panel` | Dock right, scrollable, hidden by default |
| `#history-panel.visible` | Shown state |

**Acceptance criteria:**
- Importing `widgets.py` doesn't error
- Buttons arrange in 5-row x 4-column grid visually
- CSS loads without Textual syntax errors

---

## Phase 5: App Wiring (`app.py`)

**Goal:** Wire all components together in `CalculatorApp` — compose widget tree, handle button presses and keyboard input, evaluate expressions, manage history.

**File to modify:** `src/tui_calc/app.py`

**`CalculatorApp(App)` class:**

| Attribute/Config | Value |
|-----------------|-------|
| `CSS_PATH` | `"styles.tcss"` |
| `TITLE` | `"TUI Calculator"` |
| `BINDINGS` | Escape=clear_all, h=toggle_history, q=quit |
| `_expression` | `str = ""` (current expression buffer) |
| `_history` | `HistoryStore()` |
| `_last_result` | `str | None` (for chaining) |

**`compose()`:** Yield `Header`, `Display`, `Horizontal(ButtonGrid, HistoryPanel)`, `Footer`

**Event routing (`on_button_pressed`):**

| Button | Action |
|--------|--------|
| `0-9`, `.` | Append to expression. If `_last_result` set, start new expression |
| `+`, `-`, `*`, `/` | Append. If `_last_result` set, chain from last result |
| `(`, `)` | Append |
| `=` | Call `_evaluate()` |
| `CE` | Remove last char (backspace) |
| `C` | Reset expression + result |

**Keyboard handling (`on_key`):**

| Key | Maps to |
|-----|---------|
| `0-9` | Digit button |
| `+`, `-`, `*`, `/` | Operator button |
| `(`, `)` | Paren button |
| `Enter` | `=` button |
| `Backspace` | `CE` button |
| `Delete` | `C` button |

**`_evaluate()`:** Call `safe_eval`, `format_result`, update display, add to history, catch `CalculatorError` and show user-friendly error.

**Acceptance criteria:**
- `tui-calc` launches full calculator
- Clicking buttons builds expression and evaluates on `=`
- Keyboard input works identically to buttons
- `C` clears, `CE` backspaces
- `h` toggles history panel
- Division by zero shows error, not crash
- `q` quits, `Escape` clears

---

## Phase 6: Integration Tests (`test_app.py`)

**Goal:** Async integration tests using Textual's `pilot` to verify end-to-end behavior.

**File to create:** `tests/test_app.py`

All tests use: `async with CalculatorApp().run_test() as pilot:`

| Test | Scenario |
|------|----------|
| `test_button_click_addition` | Click 2, +, 3, =. Verify display shows "5" |
| `test_button_click_multiplication` | Click 4, *, 5, =. Verify "20" |
| `test_clear_button` | Type 123, click C. Verify empty |
| `test_clear_entry_button` | Type 123, click CE. Verify "12" |
| `test_keyboard_addition` | Press 1, +, 2, Enter. Verify "3" |
| `test_keyboard_parentheses` | Press (, 2, +, 3, ), *, 4, Enter. Verify "20" |
| `test_backspace` | Type 12, Backspace. Verify "1" |
| `test_escape_clears` | Type 123, Escape. Verify cleared |
| `test_division_by_zero` | Type 1/0, Enter. Verify error message shown |
| `test_invalid_expression` | Type +*, Enter. Verify error message |
| `test_history_toggle` | Evaluate 2+3, press h. Verify panel visible with "2+3 = 5" |
| `test_history_multiple` | Evaluate 3 expressions, toggle history. Verify all 3 newest-first |
| `test_result_chaining` | Evaluate 2+3 (=5), type *2, Enter. Verify "10" |

**Also modify:** `pyproject.toml` — add `[tool.pytest.ini_options]` with `asyncio_mode = "auto"`

**Acceptance criteria:**
- `pytest tests/ -v` passes all unit + integration tests
- No test relies on sleep hacks; uses `pilot.pause()` for Textual message processing

---

## Phase 7: Polish and Documentation

**Goal:** Complete README, final review pass, verify clean install.

**Tasks:**
1. Complete `README.md`: install instructions, usage, keyboard shortcuts table, button layout diagram, dev/test commands
2. Update `__init__.py`: export public API (`safe_eval`, `format_result`, `HistoryStore`, `HistoryEntry`)
3. Review pass: no function >50 lines, user-friendly error messages, no hardcoded magic values
4. Verify no `eval()` or `exec()` anywhere in codebase (grep check)

**Acceptance criteria:**
- `pip install -e ".[dev]" && pytest tests/ -v` passes from clean venv
- `tui-calc` runs, all buttons and keyboard shortcuts work
- README is complete and accurate

---

## Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| AST whitelist incomplete (new Python node types) | Allowlist approach — new node types blocked by default. Test with malicious inputs. |
| Textual CSS errors only caught at runtime | Integration tests in Phase 6 load the app, catching CSS errors immediately |
| Keyboard `h` captured as input vs history toggle | Textual binding system consumes bound keys before `on_key`. Test in Phase 6. |
| `2**999999999` hangs | Exponent cap at 1000 in `_eval_node`. Explicit test. |
| `0.1+0.2` floating point noise | `format_result` rounds to 10 significant digits |

---

## Success Checklist

- [ ] `pip install -e ".[dev]"` works cleanly
- [ ] `tui-calc` launches functional calculator TUI
- [ ] 5-row x 4-column button grid renders correctly
- [ ] Mouse clicks build and evaluate expressions
- [ ] Keyboard input works identically to buttons
- [ ] `safe_eval` rejects all non-whitelisted AST nodes
- [ ] Division by zero and invalid expressions show friendly errors
- [ ] History panel toggles with `h`, shows entries newest-first
- [ ] All unit + integration tests pass
- [ ] No `eval()` or `exec()` in codebase
