---
name: pep8-style
description: >
  Apply PEP 8 – the official Python style guide – when writing, reviewing, or
  refactoring Python code. Use this skill whenever the user asks to:
  write new Python code, review Python for style issues, fix or clean up
  existing Python, lint Python files, check Python formatting, or mentions
  PEP 8 / Python style conventions explicitly. Even if the user just says
  "write me a Python function" or "can you clean this up?", use this skill
  to ensure the output is PEP 8 compliant. Always use this skill when Python
  code quality or style is involved in any way.
---
 
# PEP 8 Style Skill
 
Guide Claude to write, review, and fix Python code according to
[PEP 8 – Style Guide for Python Code](https://peps.python.org/pep-0008/).
 
---
 
## Workflow
 
### 1. Identify the task
 
| Task | Action |
|------|--------|
| **Write new code** | Follow all rules in the Quick Reference below from the start |
| **Review existing code** | Audit against each rule category; produce a structured violation report |
| **Fix/refactor code** | Apply automated tools if available, then verify manually |
 
### 2. Use linting tools when bash is available
 
If you have access to `bash_tool`, always try to run automated tools first:
 
```bash
# Install if needed
pip install flake8 autopep8 --break-system-packages -q
 
# Check violations
flake8 --max-line-length=79 <file>
 
# Auto-fix (safe fixes only)
autopep8 --in-place --aggressive <file>
 
# Show diff before applying
autopep8 --diff <file>
```
 
When tools are unavailable, apply the Quick Reference rules manually.
 
### 3. Report violations clearly — exhaustively
 
When reviewing or fixing code, structure your output as:
 
```
## PEP 8 Review
 
### Violations Found  (ALL of them — nothing omitted)
- [E501] Line 14 exceeds 79 characters (92 chars)
- [E302] Line 20: expected 2 blank lines before `class Foo`, found 1
- [N802] Line 20: function name `MyFunc` should be lowercase (`my_func`)
- [W291] Line 7: trailing whitespace
... (continue for every violation)
 
### Fixed Code
<corrected code here — must be fully PEP 8 clean>
 
### Summary
X violations fixed. The fixed code above passes flake8 with zero warnings.
(If any violation was intentionally left: explain why and cite the PEP 8 exception.)
```
 
**If zero violations found**: explicitly state "No PEP 8 violations found. Code is fully compliant."
 
Do not collapse or group violations to save space — list every instance individually (e.g. if line length is exceeded on 5 lines, list all 5).
 
---
 
## Quick Reference — PEP 8 Rules
 
### Indentation
- **4 spaces** per level. Never tabs.
- Continuation lines: align with opening delimiter, OR use hanging indent (+4 spaces) with no args on first line.
- Closing bracket on its own line, aligned with first non-whitespace of last line, or first char of opening line.
 
### Line Length
- **Max 79 characters** for code.
- **Max 72 characters** for docstrings and comments.
- Wrap using implicit continuation inside `()`, `[]`, `{}`. Prefer this over `\`.
- Break **before** binary operators (Knuth style):
  ```python
  # Correct
  income = (gross_wages
            + taxable_interest
            - ira_deduction)
  ```
 
### Blank Lines
- **2 blank lines** around top-level function and class definitions.
- **1 blank line** around method definitions inside a class.
- Use blank lines sparingly inside functions to separate logical sections.
 
### Imports
- One import per line: `import os` / `import sys` (not `import os, sys`).
- `from x import a, b` on one line is fine.
- Order: (1) stdlib → (2) third-party → (3) local. Blank line between groups.
- Avoid wildcard imports (`from x import *`).
- Absolute imports preferred; explicit relative imports acceptable.
- Module-level dunders (`__all__`, `__version__`) go after module docstring, before imports (except `from __future__`).
 
### String Quotes
- Pick single or double quotes and be consistent. Don't mix without reason.
- Use the other quote style to avoid escaping: `"it's fine"` vs `'it\'s fine'`.
- Triple-quoted strings always use `"""` (for docstrings, per PEP 257).
 
### Whitespace
**Avoid** extra whitespace:
- Inside brackets: `spam(ham[1], {eggs: 2})` ✓ — not `spam( ham[ 1 ], { eggs: 2 } )`
- Before comma/colon/semicolon: `if x == 4: print(x, y)` ✓
- Before function call parens: `spam(1)` ✓ — not `spam (1)`
- Before index/slice brackets: `dct['key']` ✓ — not `dct ['key']`
- Aligning assignments with extra spaces: just use one space each side.
 
**Always** surround with single space:
- Binary operators: `=`, `+=`, `==`, `<`, `>`, `!=`, `in`, `not in`, `is`, `is not`, `and`, `or`, `not`
- Exception: no spaces around `=` for keyword args or unannotated defaults:
  ```python
  def func(x, y=0):       # correct
  func(x, y=0)            # correct
  def func(x: int = 0):   # correct (annotated — space required)
  ```
 
**Slice rule**: colon acts like a binary operator with equal space each side:
```python
ham[1:9]              # correct (simple)
ham[lower+offset : upper+offset]   # correct (complex)
ham[lower + offset:upper + offset]  # wrong
```
 
### Trailing Commas
- Mandatory for single-element tuples: `FILES = ('setup.cfg',)`
- Recommended for multi-line collections (helps version control diffs):
  ```python
  FILES = [
      'setup.cfg',
      'tox.ini',
  ]
  ```
 
### Comments
- Keep comments current — stale comments are worse than none.
- Block comments: full sentences, indented to match code, start with `# `.
- Inline comments: use sparingly, at least 2 spaces from statement, start with `# `.
- Don't state the obvious: `x = x + 1  # Increment x` is noise.
- Write in English.
 
### Docstrings (PEP 257)
- Write docstrings for all public modules, classes, functions, methods.
- One-liner: `"""Return an ex-parrot."""` (closing `"""` same line)
- Multi-liner: closing `"""` on its own line.
- Non-public methods: use a regular `#` comment after the `def` line instead.
 
### Naming Conventions
 
| Entity | Style | Example |
|--------|-------|---------|
| Package / Module | `lowercase` or `lower_with_underscores` | `mypackage`, `my_module` |
| Class | `CapWords` | `MyClass` |
| Exception | `CapWords` + `Error` suffix | `ValidationError` |
| Function / method | `lower_case_with_underscores` | `compute_value()` |
| Variable | `lower_case_with_underscores` | `total_count` |
| Constant | `UPPER_CASE_WITH_UNDERSCORES` | `MAX_RETRIES` |
| Type variable | `CapWords`, short | `T`, `KT`, `VT` |
| "Internal" names | `_leading_underscore` | `_helper()` |
| Name mangling | `__double_leading` | `__private_attr` |
| Avoid | `l`, `O`, `I` as single-char names (ambiguous) | — |
 
### Programming Recommendations
 
- Use `is` / `is not` to compare with singletons (`None`, `True`, `False`): `if x is None`
- Use `is not` not `not ... is`: `if x is not None` ✓
- Don't compare booleans with `==`: `if flag:` not `if flag == True:`
- Use `isinstance()` not `type()` for type checks: `isinstance(x, int)`
- Catch specific exceptions, not bare `except:`
- For sequences, empty = falsy: `if not seq:` rather than `if len(seq) == 0:`
- Use `def` not `lambda` assigned to a name: `def f(x): return x*2` not `f = lambda x: x*2`
- Derive exceptions from `Exception` not `BaseException`
- Use `str.startswith()` / `str.endswith()` rather than slicing for prefix/suffix checks
 
### Function and Variable Annotations (Python 3)
- Follow normal colon rules, always space around `->`:
  ```python
  def greet(name: str) -> str:
  ```
- Annotated defaults need spaces around `=`:
  ```python
  def func(x: int = 0): ...   # correct
  ```
 
---
 
## Strictness Policy — Flag Every Violation
 
This skill is **strict**: report and fix *every* PEP 8 violation, no matter how minor.
 
- Do not skip or silently ignore any rule category.
- Do not say "this is minor" or "this is optional" without flagging it.
- Do not leave any violation unfixed in the output code unless the user explicitly asks to keep it.
- If a rule has exceptions defined in PEP 8 itself (e.g. a project-specific line length agreement), note the exception but still flag the default violation.
- When writing new code, produce output that passes `flake8` with zero warnings out of the box.
- When reviewing, produce an exhaustive list — no item is too small to mention.
 
**Every rule in the Quick Reference above is mandatory, not advisory.**
 
---
 
## Common Flake8 Error Codes Reference
 
| Code | Rule |
|------|------|
| E1xx | Indentation |
| E2xx | Whitespace |
| E3xx | Blank lines |
| E4xx | Imports |
| E5xx | Line length |
| E7xx | Statement style |
| W6xx | Deprecated features |
| N8xx | Naming (requires pep8-naming plugin) |
 
---
 
## Full PEP 8 Reference
 
For edge cases or detailed rules, read the canonical source:
👉 https://peps.python.org/pep-0008/
 
For docstring conventions:
👉 https://peps.python.org/pep-0257/
 
