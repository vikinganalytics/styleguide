---
permalink: /pyguide
---
# Python Style Guide for Tooling

This guide describes how we build Python tooling: command-line tools,
developer utilities, and the libraries behind them. Its goal is a design
language that prevents common classes of errors by construction rather
than by defensive handling.

## Principles

Two principles are the point of this guide; everything else is in service
of them:

1. **A robust type system, so that bugs cannot be represented.** The best
   error handling is a state that cannot be constructed. Invariants are
   pushed into types — frozen, validated, precise — so that the type
   checker rejects whole classes of bugs before the program ever runs.
2. **Strong boundaries and contracts between moving parts.** Parts must not
   be able to interfere with each other in unintended or counterintuitive
   ways — and when something messy sneaks in anyway, despite our best
   intentions, the blast radius stays contained on one side of a boundary.

Every rule below is either a consequence of these principles or a
convention adopted for consistency (line length, jsonlines, import
grouping). The distinction matters in review: a deviation that weakens a
principle calls for a redesign, while a deviation from a convention needs
only a good reason. When rules seem to conflict, resolve toward the
principles.

## Precedence

When making a design or style decision, consult in this order:

1. **This guide.** Internal rules always win.
2. **The [Google Python Style Guide](https://google.github.io/styleguide/pyguide.html)**,
   for anything this guide does not cover.
3. **The [Zen of Python](https://peps.python.org/pep-0020/)** (`import this`),
   for anything still unclear.

Not every rule applies to every tool. A single-command CLI does not need
subcommand dispatch; a library needs no argument parsing at all. Skip the
parts that do not apply — but when a part _does_ apply, follow it as written.

---

## 1. Tooling baseline

Every project uses the same enforced toolchain. Style that is not enforced
by a tool decays; put rules in configuration, not in code review.

- **uv** for all package management and script execution
  (`uv run`, `uv lock --locked` in CI). Never pip.
- **ruff** for formatting _and_ linting, with a 100-character line length and
  a broad rule set. Start from this selection and remove only with a written
  justification:

  ```toml
  [tool.ruff]
  line-length = 100

  [tool.ruff.lint]
  extend-select = [
      'ANN', 'ARG', 'B', 'C4', 'E', 'EXE', 'F', 'FBT', 'FURB', 'I', 'ICN',
      'LOG', 'N', 'PERF', 'PGH', 'PIE', 'PLC0415', 'PLE', 'PT', 'PTH', 'Q',
      'RET', 'RUF', 'S', 'SIM', 'T10', 'TC', 'TD', 'TRY', 'UP', 'W', 'YTT',
  ]
  ```

  Every `ignore` entry gets a comment explaining _why_ it is ignored
  (e.g. `"T20"  # Print is intentional in a CLI tool`). Inline suppressions
  use the specific rule code, never a bare `# noqa`.

- **mypy in strict mode** for type checking. All code is fully annotated:
  the `ANN` ruff rules enforce that annotations are _present_; mypy
  enforces that they _hold_. The choice of checker is deliberate, not
  incidental: section 3's guarantees are only as strong as the checker's
  understanding of `attrs` — mypy's attrs plugin understands `converter=`
  and `validator=` semantics, and today no other checker verifiably does.
  Revisit faster checkers (e.g. ty) once their attrs support is proven
  against a test file of deliberate violations, not against release notes.
- **import-linter** for module-boundary contracts (section 2). An
  architecture that is enforced only by convention decays exactly the way
  unenforced style does.
- **vulture** for dead-code detection, with a committed allowlist file.
- **pytest + coverage** with a high enforced floor (`--fail-under=99`).
  Coverage that is not enforced is decorative.
- **One runner for all checks.** Every check above is wired into a single
  job-runner configuration as a named job with a fix counterpart, so the
  full suite runs identically on a developer machine, in a git hook, and
  in CI — one command to check, one command to fix.

## 2. Program structure

### The pipeline: argv → dispatch → parse → Context → business logic

A multi-command CLI has exactly one flow for arguments, and data crosses each
boundary in exactly one shape:

```
sys.argv[1:]
  → match/case dispatch on the first token          (__main__.py)
  → per-command module: parse_args(args)            (argparse)
  → frozen attrs Context                            (converters + validators)
  → module.main(ctx) -> int                         (business logic)
```

Rules:

- **Dispatch with structural pattern matching.** `case ["run", *args]:` reads
  as the grammar it implements, and keeps the full command grammar visible
  in one place.
- **Each command is a module satisfying a `Protocol`**, not a registration in
  a framework:

  ```python
  class Module(Protocol):
      @staticmethod
      def parse_args(args: Sequence[str]) -> Context: ...
      @staticmethod
      def main(ctx: Context) -> int: ...
  ```

  When a tool owns a _resource_ — a cache, a connection — it appears as an
  explicit parameter (`main(cache, ctx)`): opened at the entry point and
  injected (section 7), never reached for globally. Resources are the
  deliberate exception to the immutability rules of section 3 — stateful by
  nature, which is exactly why they are singular and passed visibly. But add
  such a parameter when the tool actually acquires the resource, not
  speculatively (section 10).

- **The `argparse.Namespace` never leaves `parse_args`.** It is converted
  field-by-field into a frozen `attrs` Context immediately. Business logic
  never sees argv, a Namespace, or a dict of options.
- **Shared flags are defined once**, in a central `Argument` enum plus an
  `add_argument(parser, argument)` dispatcher, so `--quiet` or `--repo-path`
  cannot drift between subcommands.
- **`main` returns an exit code**; only the top-level entry point calls
  `sys.exit`.

Simpler tools drop layers from the top, never from the bottom: a
single-command CLI skips the dispatch; a library skips argparse. But _any_
tool with configuration or input still builds a validated, frozen Context
(or equivalent typed object) at its boundary, and its internals consume only
that.

### Module boundaries are contracts

The pipeline above implies a layering, and a layering that lives only in
convention erodes one convenient import at a time. Declare it as an
[import-linter](https://import-linter.readthedocs.io/) contract and run it
with the rest of the checks:

```toml
[tool.importlinter]
root_package = "mytool"

[[tool.importlinter.contracts]]
name = "Layered architecture"
type = "layers"
layers = ["mytool.__main__", "mytool.commands", "mytool.core", "mytool.models"]

[[tool.importlinter.contracts]]
name = "Command modules are independent"
type = "independence"
modules = ["mytool.commands.*"]
```

Business logic importing the argparse layer, or one command module
reaching into a sibling's internals, is then a CI failure rather than a
review catch. This is the module-level expression of the boundary
principle: lower layers know nothing of what sits above them, and sibling
commands cannot couple to each other. Code two commands both need lives
one layer down, imported by both — never in one command, imported by the
other.

### Streams, not files

Follow the Unix convention for the three standard streams:

- **stdin** — data input.
- **stdout** — data output, and nothing else.
- **stderr** — logs, errors, progress, and all other user-facing
  information.

In practice this means a bare `print()` is reserved for data; everything
user-facing is `print(..., file=sys.stderr)` (this is also why the `T20`
lint rule is ignored for CLI tools).

For stdin and stdout, go a step further: **jsonlines is the highly
encouraged format**. One JSON object per line composes with the entire Unix
toolbox (`jq`, `grep`, `head`, pipes between tools) and parses
unambiguously. Records arriving on stdin are input like any other: they
cross the same validated boundary as argv — deserialized (cattrs) into
frozen, validated models before business logic sees them.

Do not force a filesystem dependency on the user. Paths on disk are shared
mutable state — they race, easily, in exactly the way section 3 teaches us
not to trust. A tool that reads stdin and writes stdout leaves the choice
to the user, who can still bring files when they want the statefulness:

```sh
uv run tool.py < input.jsonl > output.jsonl
```

Accepting a path argument is an occasionally-justified convenience;
_requiring_ one is a design smell.

## 3. Data and types

### The outside world is untrusted

Every class, function, and program — including `def main()` — treats
whatever it is given as untrusted user input. The distrust is symmetric:
just as we don't trust our input, we don't trust what consumers will do
with what we hand out. Two consequences follow on the consuming side:

- **Assume nothing beyond what the type system and the contract enforce.**
  Given a `Sequence`, you may rely on the caller's order reaching you
  intact — that is what `Sequence` communicates. Whether that order _means_
  anything is part of the function's contract: for argv it does. What you
  never assume is a property nobody promised — sortedness, uniqueness,
  non-emptiness. If you need a particular ordering of your own, produce it
  yourself: `sorted(arg, key=...)`.
- **Immutability is what makes the input usable at all.** If the thing you
  are given is immutable and its type is what you expect, you can simply use
  it — no worrying about whether the caller kept a reference and is mutating
  it, or whether you are racing with something else working on the same
  object. Without immutability, none of that is safe to assume.

The same frame governs producers, and it is what drives handing out
immutable structures and views in the first place: if the type you expose
makes mutation possible, assume some consumer will eventually mutate it —
and your internal state along with it. Since consumers do not trust
iteration order anyway, there is also rarely a reason to hand out a list:
prefer sets, annotate parameters as `Collection`, and — since we always
want the immutable version — that lands on `frozenset` all over the place.
(`Sequence` remains the right annotation when order _is part of the
meaning_ — argv, an ordered pipeline of stages.) The usual pushback is
"this element isn't hashable, so it can't go in a set"; ask _why_ it isn't
hashable, and the answer is almost always a mutable field — which violates
the immutability principle and should be fixed, not worked around.

Note what the concern actually is: **sharing mutable state with code you
don't control.** And "code you don't control" is broader than it sounds.
In a collaborative codebase it includes the small utility function you
wrote yourself — six months from now a colleague rewrites it with
absolutely none of your regard for the immutability principles you adhere
to. What you control is what is in front of you right now, nothing more.

Immutability is therefore a boundary discipline, not a ban on mutation.
While you exclusively own an object — locally, inside the function you are
writing — mutate it freely: comprehensions, a local `set` you `.update()`,
a list you `.extend()` in a loop are all fine, where appropriate. Freeze
at the boundary: the moment the object becomes visible to any other code,
it is a `frozenset`, a tuple, or a frozen attrs instance.

Two of the guarantees here are enforced statically, not at runtime:
returning a dict annotated as `Mapping` does not make the dict immutable —
it makes mutating it a type error, which has teeth only because the type
checker is mandatory (section 1). Runtime immutability (`frozenset`,
tuples, `attrs.frozen`) is used where it is cheap; the static contract
covers the rest.

### Rules

- **`attrs`, not dataclasses.** Dataclasses have neither converters nor
  validators, so they cannot implement the validated-boundary pattern this
  guide rests on. Boundary objects are `@attrs.frozen` with `converter=`
  and `validator=` on every field:

  ```python
  @attrs.frozen
  class Context:
      repo_path: Path = attrs.field(
          converter=[Path, Path.resolve],
          validator=attrs.validators.instance_of(Path),
      )
      stages_or_jobs: frozenset[str] = attrs.field(converter=frozenset, ...)
  ```

  Coercion and validation happen at construction, once. After that, the
  object is trusted everywhere — no re-checking downstream. This is the main
  mechanism that prevents whole classes of bugs: invalid states are
  unrepresentable rather than handled.

- **Converters are defensive copies, not just coercion.** A
  `converter=frozenset` on a field does two jobs: it normalizes the type,
  and it severs the object from whatever mutable structure the caller passed
  in. Without it, the caller may repurpose and mutate that list or dict for
  its own ends after handing it to you — and your "immutable" object now
  changes underneath you.
- **Immutable by default.** `frozenset`, tuples, and `Mapping` return types.
  Derive new state with `attrs.evolve(ctx, ...)`, never by mutation.
- **When full immutability is genuinely too expensive**, go as far as you
  can cheaply: hand consumers `dict.keys()` / `dict.values()` views rather
  than the dict itself, so they at least cannot mutate your state through
  what you gave them.
- **Non-determinism anxiety is healthy — keep it.** Sets have no stable
  iteration order, so determinism is _produced at the output point_, not
  stored in the structure: `sorted(...)` wherever order reaches a user —
  serialized output, error messages, display.
- **Annotate what you need — not necessarily what you know you'll be
  given.** Take argv: you know you will receive `sys.argv[1:]`, which is a
  `list[str]`. You need its order preserved — argv has a strong external
  convention, and nothing inside your function can reconstruct the "right"
  order — but you do not need to mutate it. So relax every part you don't
  need and annotate `Sequence[str]`: not `list[str]` (demands mutability
  you won't use), not `tuple[str, ...]` (must stay compatible with
  `sys.argv[1:]`, which isn't a tuple), not `Collection[str]` (unordered —
  discards the order you do need). Likewise a `Mapping` parameter: the
  caller may pass a `dict` or a `MappingProxyType`, both equally
  compatible, and the function promises to use nothing beyond what
  `Mapping` guarantees — if it does anyway, the type checker catches it.
  **Annotations are requirements and promises, and the type checker
  verifies that everything holds.** Accept abstractions from
  `collections.abc`; return concrete immutable types. Use `Protocol` for
  structural interfaces between layers (e.g. a `Runnable` with
  `run(cache) -> int`).
- **Modern syntax throughout**: builtin generics (`frozenset[str]`), `X | Y`
  unions, `pathlib` exclusively for paths.
- **cattrs** for (de)serialization of config files into these models —
  parsing and validation stay declarative.

### Precision: beyond containers

Frozen, validated objects are the foundation, but immutability only
prevents _corruption_ of state — it does not prevent _confusion_ of state.
The type checker can only reject what the types distinguish. Four
instruments close the gap:

- **Keyword-only args when types collide.** A function taking two `str`
  positionals — say a job name and a stage name — will eventually be
  called with the arguments swapped, silently. Force the caller to name
  them:

  ```python
  def assign(*, job: str, stage: str) -> None: ...
  ```

  The `FBT` rules already require this for booleans (section 5); apply
  the same discipline whenever two or more parameters share a type.
  Mixups become visible at the call site and caught by the checker.

- **Enums for closed sets of values.** When a string parameter has a
  discrete, known set of valid values, model it as an `enum.Enum`. The
  type checker then rejects typos and invalid values statically, and
  `cattrs` structures them natively — no custom hooks needed. Reserve
  bare `str` for values that are genuinely open-ended.

- **Tagged unions, not optional-field combinations.** A class with several
  `X | None` fields where only certain combinations are valid is the most
  common way invalid states stay representable. Model the alternatives as
  a union of frozen classes and let `match` take them apart:

  ```python
  @attrs.frozen
  class RunJob:
      job: str

  @attrs.frozen
  class RunStage:
      stage: str
      jobs: frozenset[str]

  Target = RunJob | RunStage
  ```

  Each variant carries exactly the fields valid for it. No
  `assert ctx.stage is not None` appears downstream, because there is
  nothing to assert.

- **Exhaustive `match` with `assert_never`.** Every `match` over a union
  or enum ends with a default arm calling `typing.assert_never`:

  ```python
  match target:
      case RunJob(job=job): ...
      case RunStage(stage=stage, jobs=jobs): ...
      case unreachable:
          typing.assert_never(unreachable)
  ```

  Adding a variant then fails the type check at every `match` that does
  not yet handle it — the checker finds the sites a new feature must
  touch, instead of a runtime surprise finding them later.

- **`Any` is a hole in the hull.** One `Any` silently disables checking
  for everything it flows into, downstream of the annotation. It may
  appear only at deserialization edges — the raw object a parser hands
  back — and must be converted to a precise type in the same function.
  mypy's strict mode enforces most of this; an untyped third-party
  library is wrapped behind a small typed facade rather than allowed to
  leak `Any` into the codebase.

### The cost of immutability, honestly

Most of the feared overhead is not real: `attrs.evolve` copies _by
reference_, not by value, so evolving a large Context is cheap. The real
cost is that **converters and validators rerun on every construction** —
including every `evolve` — and deep validators (`deep_iterable`,
`deep_mapping`) walk their whole collection each time. That can become a
legitimate performance problem.

The escalation path is empirical, not speculative. Tooling should feel
instant. When it becomes noticeably laggy: profile (typically `cProfile`).
Only if the profile shows converters or validators dominating, move or
remove _those specific_ copies or validations — never immutability
wholesale. Until the profile says otherwise, keep the discipline; it is
what lets everything downstream skip re-checking and locking.

## 4. Imports

- Absolute, module-level, fully qualified: `import mytool.models` and
  `mytool.models.JobConfig` at the use site. Call sites are
  self-documenting; `from x import y` is reserved for a handful of
  highly-recognizable names.
- Grouped stdlib / third-party / local (ruff `I` enforces this).
- No imports inside functions (`PLC0415`) except behind an explicit
  per-file ignore with a reason (e.g. optional dependencies).

## 5. Control flow and idiom

- `match`/`case` for any dispatch on shape or value — argv, enums,
  `(suffix, mode)` tuples. Prefer it over if/elif chains when there are
  three or more arms or the arms destructure. When the subject is a union
  or enum, the final arm is `typing.assert_never` (section 3) — except in
  argv dispatch, where the final arm is the unknown-command error.
- Walrus operator for check-and-bind (`if path := os.getenv(...)`).
- `contextlib.suppress` over `try/except: pass`.
- Boolean parameters are keyword-only (`*, exist_ok: bool`) — the `FBT`
  rules enforce this. The same applies when two or more parameters share a
  type (section 3): `assign(*, job: str, stage: str)` prevents silent
  mixups that positional arguments allow.
- Generators for streams of work (events, matches); the caller decides
  about collection.

## 6. Errors

- **A small exception hierarchy per failure domain**, each class with a
  one-line docstring and message formatting in `__init__`:

  ```python
  class UnknownJobError(ConfigError):
      """Job name is unknown."""
      def __init__(self, referenced_by: str, unknown_jobs: Collection[str], available_jobs: Collection[str]) -> None:
          super().__init__(f"Unknown jobs found in {referenced_by!r}: {sorted(unknown_jobs)}; ...")
  ```

  Callers raise with structured arguments; only the exception knows its
  prose. Error messages include what was found _and_ what was expected or
  available.

- **One translation point.** The top-level `main()` is the only place that
  catches domain exceptions and turns them into stderr messages and exit
  codes. Internals raise; they do not print and continue.
- Re-raise across layers with context: catch `ValueError` from attrs
  validation and raise the domain error `from exc`.
- Long messages inline are fine (`TRY003` is ignored) — one clear message
  beats a proliferation of single-use exception classes.

## 7. Resources and external state

- **Open once at the top, inject downward, close in one place.** The process
  entry point owns the resource lifetime (`with diskcache.Cache(...) as cache:`);
  everything below receives it as a parameter.
- **Inject as a protocol, not a concrete type.** A checksum cache is passed
  as `MutableMapping[bytes, bytes]`, so tests substitute a plain dict and no
  module depends on the storage backend.
- **Cleanup lives in `try/finally`**, including inside generators, and the
  entry point rebinds SIGINT/SIGTERM/SIGHUP to raise `KeyboardInterrupt` so
  that `finally` blocks actually run on external termination.
- **Treat shared filesystem state as crash-tolerant, not locked:**
  - Idempotent initialization (`mkdir(exist_ok=True)`, `touch`) at every
    startup — no "first run" special case.
  - Atomic replacement via write-to-temp-name + `rename()`, never in-place
    rewriting. The temp name must live in the _target directory_ —
    `rename()` is only atomic within one filesystem. This is the filesystem
    equivalent of `attrs.evolve` on a frozen class: we replace the inode
    rather than mutating the contents of the old one, so a process still
    working with the original inode is never disrupted — it keeps its
    consistent snapshot, exactly as a consumer holding a frozen object
    does.
  - Ownership tokens (e.g. pid files named by checksum) checked before
    deletion, so concurrent processes cannot clobber each other.
- **Respect platform conventions for cache and config locations.** Code
  must be well behaved on developer laptops, and many of those are macOS.
  Resolve the cache directory with an explicit precedence chain:

  1. A tool-specific env var (`MYTOOL_CACHE_DIR`), if set.
  2. `$XDG_CACHE_HOME/mytool`, if set.
  3. `~/Library/Caches/mytool` on macOS (`sys.platform == "darwin"`).
  4. `~/.cache/mytool` on other POSIX systems.
  5. A directory under `tempfile.gettempdir()` as a last resort, with a
     warning to stderr.

  Check writability (`os.access(..., os.W_OK)`) before settling on a
  candidate — a heuristic, so still handle `OSError` at use time — and fail
  loudly, rather than falling through, when an _explicitly configured_
  location (steps 1–2) is unusable: the user asked for that location, so
  silently caching elsewhere is a bug.

  Resolve the chain once, in the entry point, and pass the result down —
  through the Context or as a parameter — rather than into a module-level
  constant. Resolved-at-import globals are hidden shared state like any
  other, and they force tests to patch what they should be able to inject.

- Partition on-disk formats by anything that breaks compatibility (e.g.
  pickle caches keyed by Python minor version).

## 8. Comments and docstrings

- **Comments explain _why_, never _what_.** Platform quirks, performance
  constraints ("very hot loop, attribute lookup gets expensive"),
  ordering requirements, chicken-and-egg gotchas. If a comment restates the
  code, delete it.
- **Docstrings are sparse in source code.** Types, names, and structure
  carry the meaning; a one-line docstring on exception classes and genuinely
  non-obvious functions is enough. (Tests are the exception — see below.)
- Suppressions are surgical: `# noqa: S311` with the specific code, ideally
  with a trailing reason.

## 9. Testing

- **pytest**, with `tmp_path` for filesystem work. Because resources and
  locations are injected (section 7), tests rarely need patching at all:
  hand in `{}` for the cache, `tmp_path` for the cache directory. Reaching
  for `unittest.mock.patch` is a signal that something should have been
  injected — and never monkeypatch internals of the unit under test.
- **Parametrize aggressively** (`@pytest.mark.parametrize` with tuple-style
  argnames) instead of copy-pasted test bodies.
- **Every test gets a one-line docstring** stating the behavior under test.
- **Property-based/fuzz tests with hypothesis** for parsers and anything
  that consumes arbitrary user input.
- Coverage is enforced in CI at a high floor; test against the public
  behavior (files written, exit codes, output), not private call sequences.

## 10. What _not_ to copy

Some tools — job runners, daemons, anything supervising concurrent
processes — have domains that force a density of edge-case handling:
crash recovery, orphaned-process adoption, signal escalation, liveness
polling. That handling is **intrinsic to those problems, not a virtue to
emulate**. Do not cargo-cult defensive branches, recovery paths, or retry
loops into tools whose domain does not demand them.

The order of preference is:

1. **Make the edge case unrepresentable** (frozen validated types, immutable
   data, idempotent operations).
2. **Make it irrelevant** (atomic renames instead of lock-and-repair;
   ownership tokens instead of coordination).
3. Only then, **handle it explicitly** — with a comment explaining why it
   can occur.

If you find yourself writing many special cases, first ask whether the
design can be changed so they cannot happen. That principle — not the edge
cases themselves — is what this guide asks you to reproduce.
