---
name: modern-python-syntax
description: Apply modern Python syntax features from versions 3.11 through 3.14 when writing, reviewing, or refactoring Python code. Use when the user asks to write Python, generate a Python snippet, refactor or modernize Python code, review a Python file, or explicitly mentions Python 3.11, 3.12, 3.13, or 3.14. Prefer PEP 695 generic syntax over `typing.TypeVar`, PEP 654 exception groups with `except*`, PEP 701 expressive f-strings, `Self`, `TaskGroup`, unparenthesized `except`, and t-strings where appropriate. Skip when working on codebases targeting Python 3.10 or older.
metadata:
  version: 1.0.0
  python-versions: "3.11, 3.12, 3.13, 3.14"
  category: code-style
---

# Modern Python Syntax (3.11 – 3.14)

## Purpose

Write Python that takes advantage of syntax added in the four most recent minor releases. Many Python codebases still read like 3.9 — verbose `TypeVar` imports, parenthesized `except` tuples, string-quoted forward references. When the target runtime is 3.11+, prefer the newer constructs below.

## When to apply

Before writing or editing Python, determine the target version:

1. Check `pyproject.toml` for `requires-python` or `python_requires`
2. Check `.python-version`, `Pipfile`, or CI config if present
3. If no signal, **ask the user** which version they target before applying 3.12+ syntax
4. If the user is clearly on an older version (e.g., a Django LTS on 3.10), fall back to the legacy form and do not force the new syntax

Gate each feature on the minimum version listed below. Never introduce 3.14 syntax into a 3.11 project.

## Feature reference

### Python 3.11

**PEP 654 — Exception groups and `except*`** (for concurrent / aggregate errors)

```python
# Raising several unrelated errors at once
raise ExceptionGroup("validation failed", [
    ValueError("bad email"),
    KeyError("missing id"),
])

# Handling them — each except* branch runs independently
try:
    run_all_tasks()
except* ValueError as eg:
    log_validation_errors(eg.exceptions)
except* ConnectionError as eg:
    retry(eg.exceptions)
```

Use this whenever multiple independent errors can surface together — especially inside `asyncio.TaskGroup`, parallel job runners, or batch validators. Do **not** convert ordinary sequential `try/except` to `except*` — it adds noise when there is no group.

**`asyncio.TaskGroup`** — preferred over manual `gather()` / `create_task()` bookkeeping

```python
async with asyncio.TaskGroup() as tg:
    t1 = tg.create_task(fetch(url1))
    t2 = tg.create_task(fetch(url2))
# On exit: all tasks awaited; any failures surface as an ExceptionGroup
```

Use `TaskGroup` for new async code. Keep `asyncio.gather` only when you specifically need `return_exceptions=True` semantics or partial-success behaviour.

**`typing.Self`** — for fluent builders, `__enter__`, and factory methods

```python
from typing import Self

class QueryBuilder:
    def where(self, **kw) -> Self:
        self._filters.update(kw)
        return self
```

Replaces `-> "QueryBuilder"` strings and TypeVar-bound-to-class tricks. Apply to any method returning `self` or an instance of the current class.

**`BaseException.add_note()`** — attach context to an in-flight exception

```python
try:
    parse(payload)
except ValueError as e:
    e.add_note(f"while parsing payload id={payload_id}")
    raise
```

Better than wrapping-and-re-raising when the original exception type should be preserved.

**`typing.LiteralString`, `Required`, `NotRequired`** — tighten type hints where they matter (SQL/shell safety, partial TypedDicts).

### Python 3.12

**PEP 695 — inline generic syntax and `type` statement** (the single biggest readability win)

```python
# Old (3.11 and earlier)
from typing import TypeVar, Generic
T = TypeVar("T")
class Stack(Generic[T]):
    def push(self, item: T) -> None: ...

# New (3.12+)
class Stack[T]:
    def push(self, item: T) -> None: ...

def first[T](items: list[T]) -> T:
    return items[0]

# Bounds and constraints
def longest[T: str](a: T, b: T) -> T: ...
def process[T: (int, str)](item: T) -> None: ...

# ParamSpec / TypeVarTuple
type IntFunc[**P] = Callable[P, int]
type LabeledTuple[*Ts] = tuple[str, *Ts]
```

**`type` statement for aliases** — unambiguous, lazy, and generic-friendly:

```python
type UserId = int
type Result[T] = tuple[T, Exception | None]
type JSON = dict[str, "JSON"] | list["JSON"] | str | int | float | bool | None
```

Drop the `from typing import TypeAlias` + `: TypeAlias =` dance. Drop module-level `TypeVar` declarations whenever the type parameter is used in exactly one place.

**PEP 701 — unrestricted f-strings**

```python
# Reuse the same quotes inside an f-string
print(f"user: {data["name"]}")

# Multi-line expressions, comments, backslashes are all legal now
result = f"""Total: {
    sum(
        line.amount  # skip voided
        for line in invoice.lines
        if not line.voided
    ):.2f
}"""
```

Useful when embedding dict lookups or non-trivial expressions — no more juggling single/double quotes.

### Python 3.13

**PEP 696 — defaults for type parameters**

```python
# 3.13+
class Container[T = int]:
    def __init__(self, value: T) -> None:
        self.value = value

Container()       # T defaults to int
Container("hi")   # T inferred as str

type Response[T = dict[str, Any]] = tuple[int, T]
```

Great for generic container classes and response types where a sensible default exists.

**`typing.TypeIs`** — precise type narrowing (stricter than `TypeGuard`)

```python
from typing import TypeIs

def is_str_list(val: list[object]) -> TypeIs[list[str]]:
    return all(isinstance(x, str) for x in val)

def handle(items: list[object]) -> None:
    if is_str_list(items):
        # items narrowed to list[str] here
        # AND in the else branch, narrowed to "not list[str]"
        ...
```

Prefer `TypeIs` over `TypeGuard` for new narrowing helpers — it narrows both branches, not just the positive one.

**`typing.ReadOnly` in TypedDict** — mark immutable keys

```python
from typing import ReadOnly, TypedDict

class Song(TypedDict):
    name: ReadOnly[str]
    band: ReadOnly[str]
    plays: int   # still mutable
```

**`warnings.deprecated()` decorator** — machine-readable deprecation for runtime + type checker

```python
from warnings import deprecated

@deprecated("Use fetch_user_v2 instead; removed in 2.0")
def fetch_user(id: int) -> User: ...
```

**PEP 742** / pathlib walks: `Path.walk()` is the native replacement for `os.walk()` on pathlib objects — use it for tree traversal in new code.

### Python 3.14

**PEP 758 — unparenthesized `except`**

```python
# Still valid
try:
    do_work()
except (ValueError, TypeError):
    ...

# New concise form (no `as` clause)
try:
    do_work()
except ValueError, TypeError:
    ...

# Parentheses still REQUIRED when binding with `as`
try:
    do_work()
except (ValueError, TypeError) as e:
    log(e)
```

Same rule applies to `except*`. Drop parentheses only when there is no `as` clause.

**PEP 649 — deferred evaluation of annotations**

Forward references no longer need string quotes at definition time:

```python
# 3.14+
class Node:
    next: Node | None = None    # no quotes needed
    children: list[Node] = []

@dataclass
class LinkedList:
    head: Node                  # Node can be defined below this class
```

When writing new code for 3.14, drop the string quotes around self-referential and forward-referenced types. Do **not** do this in libraries that still support 3.13 or earlier — it will break at runtime.

**PEP 750 — t-strings (template string literals)**

Unlike `f"..."`, a `t"..."` produces a `Template` object with static parts and interpolations separated — let a consumer function handle escaping:

```python
from string.templatelib import Template

def sql(template: Template) -> tuple[str, list]:
    query_parts, params = [], []
    for part in template:
        if isinstance(part, str):
            query_parts.append(part)
        else:
            query_parts.append("?")
            params.append(part.value)
    return "".join(query_parts), params

user_id = 42
stmt, params = sql(t"SELECT * FROM users WHERE id = {user_id}")
# stmt   == "SELECT * FROM users WHERE id = ?"
# params == [42]
```

Reach for t-strings when building SQL, shell commands, HTML, or logging templates where user values must be escaped or separated from structure. Reach for f-strings for everything else. **Do not** use t-strings just because they exist — plain f-strings remain correct for ordinary formatting.

**PEP 765 — `return`/`break`/`continue` in `finally` is now a `SyntaxWarning`**

```python
# This now warns — and usually indicates a real bug
def foo():
    try:
        raise ValueError("oops")
    finally:
        return "swallowed"   # SyntaxWarning: 'return' in a 'finally' block
```

When reviewing code, flag these — they silently swallow exceptions.

**PEP 734 — subinterpreters in the stdlib**

For CPU-bound work on 3.14, `concurrent.interpreters` and `concurrent.futures.InterpreterPoolExecutor` are now first-class. Consider them as an alternative to `multiprocessing` when the overhead of process creation hurts and data-passing is simple.

## Decision flow for a write/edit task

1. **Determine target version** (pyproject.toml → user → assume 3.11 minimum for new code)
2. **Type hints**: if ≥ 3.12, use `class Foo[T]:` and `type Alias = ...`. If ≥ 3.13, add parameter defaults where useful.
3. **Forward references**: if ≥ 3.14, drop the string quotes. Otherwise keep them.
4. **Exception handling**:
   - Multiple exception types, no `as` clause, ≥ 3.14 → drop the parens
   - Concurrent/aggregate errors → raise `ExceptionGroup`, catch with `except*`
   - Need to enrich an exception → `add_note()`, don't wrap-and-reraise
5. **Async**: new code uses `TaskGroup`, not `gather`
6. **Fluent / self-returning methods**: annotate with `Self`
7. **Templates for SQL/shell/HTML on ≥ 3.14**: consider t-strings; otherwise f-strings

## Anti-patterns to flag in review

- `TypeVar("T")` at module scope when T is used only by one class/function (≥ 3.12: inline it)
- `StrOrInt: TypeAlias = str | int` (≥ 3.12: use the `type` statement)
- `-> "ClassName"` return annotation inside `ClassName` (≥ 3.11: use `Self`; ≥ 3.14: drop the quotes entirely)
- Manual `asyncio.gather(*tasks)` followed by exception juggling (≥ 3.11: use `TaskGroup`)
- `try: ... except (E,): ...` with a one-element tuple — the parens are unnecessary even on older versions
- `return` or `break` inside a `finally:` block (always a bug; 3.14 now warns)

## What this skill does NOT do

- Does not rewrite working code just for stylistic modernization unless the user asks
- Does not use any 3.14-only syntax on pre-3.14 targets
- Does not touch free-threaded build / JIT / subinterpreter runtime concerns — those are runtime/deployment topics, not syntax
- Does not replace f-strings with t-strings by default; t-strings are for structured templates, not everyday formatting

## Sources

Official "What's New" documents:
- Python 3.11: https://docs.python.org/3/whatsnew/3.11.html
- Python 3.12: https://docs.python.org/3/whatsnew/3.12.html
- Python 3.13: https://docs.python.org/3/whatsnew/3.13.html
- Python 3.14: https://docs.python.org/3/whatsnew/3.14.html
