---
permalink: /pyguide
---
# Python Style Guide for Tooling

This guide describes how we build Python tooling: command-line tools,
developer utilities, and the libraries behind them. Its goal is a design
language that prevents common classes of errors by construction rather
than by defensive handling.

## Principles

Three principles are the point of this guide; everything else is in
service of them:

1. **A robust type system, so that bugs cannot be represented.** The best
   error handling is a state that cannot be constructed. Invariants are
   pushed into types — frozen, validated, precise — so that the type
   checker rejects whole classes of bugs before the program ever runs.
2. **Strong boundaries and contracts between moving parts.** Parts must not
   be able to interfere with each other in unintended or counterintuitive
   ways — and when something messy sneaks in anyway, despite our best
   intentions, the blast radius stays contained on one side of a boundary.
3. **Atomic side effects, so that incomplete work is never observable.**
   Every operation that changes state visible to another part of the
   system — a file, a cache entry, an external record — either completes
   fully or leaves no trace. No consumer should ever be able to observe,
   interact with, or build on the half-finished work of another system.
   This is what makes boundaries real at runtime: immutability protects
   data in flight, atomicity protects state at rest. In practice, prefer
   not to touch the filesystem at all — read stdin, write stdout, let the
   caller decide where state lives. When filesystem state is unavoidable,
   the two atomic patterns are _replace_ (`tempfile` + `os.rename`) and
   _append_ (one complete record per write, as with jsonlines). Both are
   valid; neither is universally correct — choose based on the
   concurrency model, not by default. Append carries a life-cycle
   obligation: a file that grows without bound will at best be silently
   reclaimed by OS cache management and at worst cause storage pressure
   that takes down unrelated applications sharing the same system. If
   you choose a location and append to it, you own the cleanup strategy.

### Design goal: one binary, every environment

These tools must behave identically in three settings:

1. **A developer laptop** (macOS, Ubuntu, or WSL) — interactive use,
   quick iteration, a single invocation at a time.
2. **A VM or HPC node** — manual or scripted runs on shared
   infrastructure, often with a shared filesystem.
3. **Scaled-out execution** — hundreds or thousands of concurrent
   invocations managed by an HPC scheduler (Slurm, PBS), an OS-level
   orchestrator (`xargs -P`, GNU parallel), or a job runner.

The same program, unchanged, must be correct in all three — and
correctness in any one setting must be sufficient evidence of
correctness in the others. If a tool passes its tests on a laptop, that
must be enough to release it with confidence; no one should need to
provision a cluster-scale environment to answer "is this safe to
deploy." Conversely, a Saturday-evening run on shared infrastructure
should never surface a bug that a developer could have caught locally.
That mutual guarantee is what makes the principles above non-negotiable:
immutability and atomicity prevent the corruption that surfaces only
under concurrency; strong boundaries keep a tool's footprint predictable
so a scheduler can manage it; stdin/stdout composability lets an
orchestrator wire tools together without the tools needing to know they
are being orchestrated. A tool that works on a laptop but requires a
special "cluster mode," or one that is only safe under light load, has
failed the design goal.

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
- **ruff** for formatting _and_ linting, with a broad rule set. Start from
  this selection and remove only with a written justification:

  ```toml
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

- **An attrs-compatible type checker in strict mode** for type checking.
  All code is fully annotated: the `ANN` ruff rules enforce that
  annotations are _present_; the type checker enforces that they _hold_.
  Section 3's guarantees are only as strong as the checker's understanding
  of `attrs` — it must understand `converter=` and `validator=` semantics.
  Default to **ty** for its speed — fast feedback loops matter more than
  marginal precision. In practice the gap is small — roughly one
  `type: ignore` per 500 lines — and well worth the faster iteration
  cycle. Verify against a test file of deliberate violations committed
  to the repo, not against release notes.
- **import-linter** for module-boundary contracts. An architecture that is
  enforced only by convention decays exactly the way unenforced style does.
- **pytest + coverage** with a high enforced floor in CI. Coverage that
  is not enforced is decorative.
- **One runner for all checks.** Every check above is wired into a single
  job-runner configuration as a named job with a fix counterpart, so the
  full suite runs identically on a developer machine, in a git hook, and
  in CI — one command to check, one command to fix.

### Dependencies and Python version

**Version pinning through uv is mandatory.** Every dependency is locked to
a specific version; `latest` is banned everywhere, for everything. An
unpinned dependency is a build that works today and breaks tomorrow.

Good hygiene is expected and required of all developers: keep dependencies
as updated as they can be, resolve deprecation warnings promptly, and
track upstream changes rather than letting them pile up. Stale
dependencies accumulate security and compatibility debt that compounds
faster than it seems.

**Target the Python version one minor behind the latest release** (e.g.
3.14 while 3.15 gets its first year to stabilize). This gives the
ecosystem — type checkers, attrs, key libraries — time to catch up while
keeping us close to the leading edge.

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
- **Each command is a module exporting two functions** — not a class, not a
  registration in a framework:

  ```python
  # mytool/commands/run.py
  def parse_args(args: Sequence[str]) -> Context: ...
  def main(ctx: Context) -> int: ...
  ```

  The dispatch in `__main__.py` imports the module and calls its functions
  directly. The shape is a convention enforced by usage, not by a `Protocol`
  — mypy cannot check that a module satisfies a structural type, so a
  `Protocol` here would be decorative.

  When a tool owns a _resource_ — a cache, a connection — it appears as an
  explicit parameter (`main(cache, ctx)`): opened at the entry point and
  injected (section 6), never reached for globally. Resources are the
  deliberate exception to the immutability rules of section 3 — stateful by
  nature, which is exactly why they are singular and passed visibly. But add
  such a parameter when the tool actually acquires the resource, not
  speculatively (section 8).

- **The `argparse.Namespace` never leaves `parse_args`.** It is converted
  field-by-field into a frozen `attrs` Context immediately. Business logic
  never sees argv, a Namespace, or a dict of options.
- **Shared flags are defined once**, in a central `Argument` enum plus an
  `add_argument(parser, argument)` dispatcher, so `--quiet` or `--repo-path`
  cannot drift between subcommands:

  ```python
  # mytool/args.py
  class Argument(Enum):
      REPO_PATH = "--repo-path"
      QUIET = "--quiet"
      MAX_JOBS = "--max-jobs"

  def add_argument(parser: argparse.ArgumentParser, argument: Argument) -> argparse.Action:
      match argument:
          case Argument.REPO_PATH:
              return parser.add_argument(
                  "--repo-path", "-r",
                  type=lambda path: Path(path).resolve(),
                  default=Path.cwd(),
              )
          case Argument.QUIET:
              return parser.add_argument(
                  "--quiet", "-q",
                  action="append_const", const=1,
              )
          case Argument.MAX_JOBS:
              return parser.add_argument(
                  "-n", "--max-jobs",
                  type=int, default=0,
              )
      raise ValueError(f"Unknown argument: {argument}")  # unreachable; satisfies the type checker
  ```

  Each command's `parse_args` calls `add_argument` for the flags it needs;
  the definition stays in one place.
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
[import-linter](https://import-linter.readthedocs.io/) contract and run
it with the rest of the checks — business logic importing the argparse
layer, or one command module reaching into a sibling's internals, is then
a CI failure rather than a review catch. Lower layers know nothing of
what sits above them, and sibling commands cannot couple to each other.
Code two commands both need lives one layer down, imported by both —
never in one command, imported by the other.

### Streams, not files

Follow the Unix convention for the three standard streams:

- **stdin** — data input.
- **stdout** — data output, and nothing else.
- **stderr** — logs, errors, progress, and all other user-facing
  information.

In practice this means a bare `print()` is reserved for data; everything
user-facing is `print(..., file=sys.stderr)` (this is also why the `T20`
lint rule is ignored for CLI tools).

For stdout, **jsonlines is the encouraged machine-readable format**. One
JSON object per line composes with the entire Unix toolbox (`jq`, `grep`,
`head`, pipes between tools) and parses unambiguously. Critically,
jsonlines is naturally streamable: each record is written the moment it is
ready. This steers program design toward "do work a little bit at a time,
yield bits of output when you've made progress" rather than "build one
massive data structure and dump it at the end." The streaming model keeps
the program's memory footprint low, avoids holding a file open for a long
time, and means that incomplete work is never silently lost when a process
is interrupted — everything flushed so far is already on disk. A plain
JSON array would actively fight this pattern.

Records arriving on stdin are input like any other: they cross the same
validated boundary as argv — deserialized (cattrs) into frozen, validated
models before business logic sees them.

**Human-readable output** is sometimes the right choice — but keep it
parseable. Avoid emojis, Unicode box-drawing, and characters that render
differently across terminals. Stick to ASCII that `grep` and `awk` can
make sense of and that prints cleanly everywhere. When a tool supports
both modes, provide a flag (e.g. `--format=json|text`) and an output
abstraction: a small function or class that takes the attrs model
representing a result and formats it in the selected mode. Business logic
produces the model; the formatter decides the presentation.

Do not force a filesystem dependency on the user. Paths on disk are shared
mutable state — they race, easily, in exactly the way section 3 teaches us
not to trust. A tool that reads stdin and writes stdout leaves the choice
to the user, who can still bring files when they want the statefulness:

```sh
uv run tool.py < input.jsonl > output.jsonl
```

Accepting a path argument is an occasionally-justified convenience;
_requiring_ one is a design smell.

### Logging

All log output goes to stderr — stdout is for data (see above). Use the
stdlib `logging` module, one logger per module:

```python
import logging

logger = logging.getLogger(__name__)
```

**Logging must be configurable without code changes.** Load configuration
from a file (`logging.config.dictConfig` from a JSON/TOML file, or
`logging.config.fileConfig`) at the entry point, resolved once alongside
the rest of the program's configuration. When an overly verbose library
or subsystem is drowning useful output, an operator must be able to set
`mypackage.noisy_module` to `WARNING` in a config file and restart —
without a code change, a review, or a deploy.

Rules:

- **Never configure logging at import time.** No module-level
  `logging.basicConfig()`, no `handler.setLevel()` scattered across
  files. The entry point owns the configuration; everything else just
  calls `logger.debug/info/warning/error`.
- **Use log levels consistently.** `DEBUG` for internal state useful when
  diagnosing; `INFO` for operational milestones (started, finished,
  skipped); `WARNING` for recoverable surprises; `ERROR` for failures the
  tool will report via exit code. Do not log at `INFO` what should be
  `DEBUG` — when thousands of instances run in parallel, `INFO` that is
  really `DEBUG` becomes noise that obscures real problems.
- **Structured fields over f-strings in log messages.** Prefer
  `logger.info("processed %d records from %s", count, source)` over
  `logger.info(f"processed {count} records from {source}")` — the former
  lets structured formatters (JSON log handlers) extract fields without
  parsing, and defers string formatting to when the message is actually
  emitted (which matters at `DEBUG` level when debug logging is off).
- **A default configuration ships with the tool** so that it works
  without a config file — typically `WARNING` to stderr. The config file
  _overrides_ the default; its absence is not an error.

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
and your internal state along with it. Choose the immutable collection
that fits the data: `frozenset` when the collection is unordered and
elements should be unique (a set of job names, a set of file paths to
watch); `tuple` when order matters or duplicates are meaningful;
`Mapping` (backed by a dict) when you need key-value lookup. Don't reach
for `frozenset` by default when a tuple or Mapping would be a more
natural fit — the goal is immutability, not a specific container. Annotate
parameters with the abstract type that communicates what you need:
`Collection` when you only iterate, `Sequence` when order is part of the
meaning (argv, an ordered pipeline of stages). The usual pushback on sets
is "this element isn't hashable, so it can't go in a set"; ask _why_ it
isn't hashable, and the answer is almost always a mutable field — which
violates the immutability principle and should be fixed, not worked around.

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
covers the rest. Often the best defense is not exposing a mapping at
all — restructure so the data stays in the caller's local scope or is
absorbed into a frozen attrs field.

### Rules

- **`attrs`, not dataclasses.** Dataclasses have neither converters nor
  validators, so they cannot implement the validated-boundary pattern this
  guide rests on. Two exceptions exist: (1) the tool must run on system
  Python across a wide range of machines with truly zero third-party
  dependencies — a single-file stdlib-only script, not a normal
  distribution; (2) profiling has proven that attrs construction overhead
  is the bottleneck, and dataclasses measurably resolve it. Both cases are
  rare; neither should be assumed up front. Boundary objects are
  `@attrs.frozen` with `converter=` and `validator=` on every field:

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
  object is trusted everywhere — no re-checking downstream. Everything
  the rest of the program does — every branch, every output format,
  every edge case it handles or deliberately doesn't — rests on
  assumptions about the data that passed through this boundary. The
  validator's job is to enforce those assumptions exhaustively, so that
  data the program was never designed for cannot silently enter and
  travel through code that was never tested against it.

  This is where developer instinct most commonly fails the design.
  Rejecting data feels harsh — "the caller might have a reason for
  passing this." That instinct is not wrong: the caller's use case may
  be perfectly valid. But if the data does not match what the program
  was built, tested, and reasoned about, then the program does not
  actually serve that use case — it only appears to, until it doesn't.
  The validator makes the mismatch visible at the earliest possible
  moment, so the user can act on it: pass different data, file a
  feature request, use a previous version, or choose another tool.
  Without the validator, the same mismatch still exists — it just
  surfaces three layers deeper as a confusing `KeyError` or a silently
  wrong result, pointing nowhere near the actual problem and giving no
  one — neither the user nor the maintainer — the information they need
  to act. When assumptions about the data an application handles stop
  holding, it is critical for everyone involved that the gap is
  visible. A validator at construction makes it visible; permissiveness
  buries it.

- **Converters make strict boundaries ergonomic.** Validators enforce
  that the class's assumptions hold — but without converters, every
  caller must construct exactly the type the field demands. That burden
  pushes back against strictness: "why does this need a
  `tuple[Path, ...]`, can't I just pass my list of strings?" The caller
  is not wrong to ask. Converters let you have it both ways: the caller
  presents something easily constructed, and the converter narrows it
  into the exact representation the class needs. A converter chain like
  `converter=[partial(map, Path), dict.fromkeys, tuple]` accepts a
  bare list of strings — the kind a call to `git ls-files` produces —
  and delivers a deduplicated tuple of resolved paths. The field is
  strict; the interface is forgiving.

  Converters also serve as **defensive copies**. A `converter=frozenset`
  severs the object from whatever mutable structure the caller passed
  in. Without it, the caller may repurpose and mutate that list or dict
  for its own ends after handing it to you — and your "immutable" object
  now changes underneath you.
- **Immutable by default.** Return concrete immutable types — `frozenset`,
  `tuple`, `Mapping` — chosen to fit the data, not one container for
  everything. Derive new state with `attrs.evolve(ctx, ...)`, never by
  mutation.
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
  `collections.abc`; return `Sequence`, `Mapping`, or a shaped `tuple`
  when the structure matters (e.g. `tuple[str, int]`). Use `Protocol` for
  structural interfaces between layers (e.g. a `Runnable` with
  `run(cache) -> int`). Always parameterize generics fully —
  `Sequence[tuple[str, int, float | None]]`, never bare `Sequence` or
  `Sequence[Any]`. A type that the checker cannot see through is a type
  it cannot protect. When a precise annotation would require an import
  that is not needed at runtime, guard it with `if TYPE_CHECKING:` — a
  type-only dependency is not a real dependency, and is no reason to
  leave an annotation vague.
- **Modern syntax throughout**: builtin generics (`frozenset[str]`), `X | Y`
  unions, `pathlib` exclusively for paths.
- **cattrs** for (de)serialization of config files and jsonlines records
  into these models — parsing and validation stay declarative:

  ```python
  c = cattrs.Converter()
  c.register_structure_hook(re.Pattern, lambda obj, _: re.compile(obj))
  config = c.structure(raw_dict, MyConfig)  # MyConfig is @attrs.frozen
  ```

  Custom hooks handle types cattrs cannot infer (compiled regexes, Path
  objects); the converter + frozen attrs model replaces hand-written
  parsing and field-by-field validation. This is the extent of what cattrs
  requires in practice — see the
  [cattrs documentation](https://catt.rs/en/stable/) for the full hook
  API when the defaults are not enough.

### Precision: beyond containers

Frozen, validated objects are the foundation, but immutability only
prevents _corruption_ of state — it does not prevent _confusion_ of state.
The type checker can only reject what the types distinguish. Five
instruments close the gap:

- **Keyword-only args when types collide.** A function taking two `str`
  positionals — say a job name and a stage name — will eventually be
  called with the arguments swapped, silently. Force the caller to name
  them:

  ```python
  def assign(*, job: str, stage: str) -> None: ...
  ```

  The `FBT` rules already require this for booleans (section 4); apply
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
  Strict mode enforces most of this; an untyped third-party
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

## 4. Control flow and idiom

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

## 5. Errors

- **One translation point.** The top-level `main()` is the only place that
  catches domain exceptions and turns them into stderr messages and exit
  codes. Internals raise; they do not print and continue.
- Re-raise across layers with context: catch `ValueError` from attrs
  validation and raise the domain error `from exc`.

### Retries

**A tool that fails exits with a nonzero code. The caller decides whether
to retry.** This is the retry equivalent of the scaling model (section 8):
be a good managed process. A Slurm job that fails gets resubmitted by
the scheduler; a shell pipeline reruns the failed step; a human reads
the error and acts. Internal retry loops hide failures from the
orchestrator, make resource usage unpredictable from the outside, and
turn a clear "this failed" into an opaque hang.

The exception is **transient failure within a single logical
operation** — an HTTP request to a flaky endpoint, a connection that
drops mid-handshake. The caller cannot meaningfully retry a sub-step it
does not know about, so bounded retries with backoff are justified here.
Use **[tenacity](https://tenacity.readthedocs.io/)** — not hand-rolled
loops. The rules:

- **Bounded.** A small, fixed limit (`stop=stop_after_attempt(5)`).
  Never indefinite, never configurable without a hard ceiling.
- **Backoff.** Exponential or jittered delay (`wait=wait_exponential()`).
  Immediate retries against a struggling service make the problem worse.
- **Surfaced.** If retries are exhausted, the failure propagates as a
  normal exception (`reraise=True`) — the caller sees the same clear
  error it would have seen without retries, just delayed. Never
  swallowed, never logged and continued.
- **Logged.** Each retry attempt is logged at `WARNING`
  (`before_sleep=before_sleep_log(logger, logging.WARNING)`) so that
  operators can distinguish "succeeded after two retries" from
  "succeeded on the first try" — the former is often an early signal of
  a degrading dependency.

A typical site looks like this:

```python
import tenacity

@tenacity.retry(
    stop=tenacity.stop_after_attempt(5),
    wait=tenacity.wait_exponential(multiplier=1, max=30),
    reraise=True,
    before_sleep=tenacity.before_sleep_log(logger, logging.WARNING),
)
def _fetch(url: str) -> bytes: ...
```

If you find yourself decorating many functions with retry logic, the
problem is not that retries are too verbose — it is that the tool is
doing too much. A tool that makes one HTTP call and processes the result
needs one retry site. A tool that orchestrates a dozen network calls is
a workflow engine and should be designed as one, not papered over with
`@retry` decorators.

## 6. Resources and external state

Resources are the primary vector for shared mutable state — and the place
where the atomicity principle (principle 3) does its heaviest work. Every
rule below exists to ensure that no consumer ever sees an operation's
intermediate state.

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
  Getting this right is harder than it looks: the resolution must respect
  tool-specific env vars, `XDG_CACHE_HOME`, macOS `~/Library/Caches`,
  POSIX `~/.cache`, and a tempdir fallback — fail loudly when an
  explicitly configured location is unusable, fall through silently when a
  platform default is not. This is also why platform-conventional
  locations matter: we still manage our own cache directory, but cache
  management is hard to test robustly — and when our cleanup is buggy,
  placing the cache where the OS or a system cache manager already
  handles eviction gives it a way to contain the situation before it
  becomes a critical problem for other applications on the same system. A
  [hardened example](https://github.com/ollelindgren/cixstack/blob/main/cixstack/utils.py)
  is available as a reference (`_get_cache_dir()`).

  Resolve the cache directory once, in the entry point, and pass the
  result down — through the Context or as a parameter — rather than into
  a module-level constant. Resolved-at-import globals are hidden shared
  state like any other, and they force tests to patch what they should be
  able to inject.

- **Least privilege toward external resources.** The immutability
  principle applies beyond data structures: do not acquire capabilities
  you will not use. Use read-only credentials when the tool only reads.
  Hit a read replica when the tool does not need consistency guarantees
  from the primary. Call the narrowest API scope that covers the
  operation. The reasoning is the same as returning a `Mapping` instead
  of a `dict`: if the tool cannot write, it cannot corrupt — and the
  failure mode where a read-only tool accidentally mutates shared state
  becomes unrepresentable.
- Partition on-disk formats by anything that breaks compatibility (e.g.
  pickle caches keyed by Python minor version).

## 7. Testing

- **pytest**, with `tmp_path` for filesystem work. Because resources and
  locations are injected (section 6), tests rarely need patching at all:
  hand in `{}` for the cache, `tmp_path` for the cache directory. Reaching
  for `unittest.mock.patch` is a signal that something should have been
  injected — and never monkeypatch internals of the unit under test.
- **Parametrize aggressively** (`@pytest.mark.parametrize` with tuple-style
  argnames) instead of copy-pasted test bodies.
- **Every test gets a docstring** explaining _what_ is being tested and
  _why_ that justifies a test — the bug it guards against, the invariant
  it enforces, or the assumption it validates. One line is often not
  enough; a sentence or two of context is better than a vague summary.
  The docstring exists to prevent a future maintainer from rewriting the
  test to pass when it fails: if the purpose isn't clear, the path of
  least resistance is to make the failure go away — potentially
  re-introducing the exact bug the original author thought to check for.
- **Property-based/fuzz tests with hypothesis** for parsers and anything
  that consumes arbitrary user input.
- Coverage is enforced in CI at a high floor; test against the public
  behavior (files written, exit codes, output), not private call sequences.

## 8. Concurrency

### The scaling model: more instances, not more threads

When a tool needs to go faster or handle more work, the answer is
running more instances of the program in parallel — not making a single
instance internally concurrent. Each instance reads its own input, writes
its own output, and exits. The orchestrator (a shell loop, a job
scheduler, `xargs -P`) decides the parallelism.

This has a direct consequence for how you write your tool: **design for
single-process correctness first.** Stdin/stdout composability,
idempotent operations, and atomic filesystem writes (section 6) are what
make a program safe to run as one of a thousand copies. Internal
threading does not help if the program cannot be safely started twice in
the same directory.

### When internal concurrency is justified

Sometimes a single invocation genuinely needs concurrency — a tool that
fans out HTTP requests, or one that must overlap IO with CPU-bound work.
The rules:

- **`concurrent.futures` for IO-bound fan-out.** `ThreadPoolExecutor`
  with an explicit `max_workers` is the default. The executor is created
  at the entry point and passed down or used in a `with` block, like any
  resource (section 6).
- **Threads for IO, C extensions for CPU.** The GIL makes pure-Python
  CPU parallelism via threads ineffective. When CPU is the bottleneck and
  profiling proves it, push the hot loop into a C extension or call out
  to a compiled tool — do not reach for `multiprocessing`. The reason is
  not only technical (serialization, fork-safety, shared-state
  complexity) but architectural: in the environments these tools run in,
  your process is managed — by `xargs -P`, GNU parallel, Slurm, PBS, or
  a similar scheduler. That scheduler decides how many copies of you run,
  on which cores, with what resource limits. A tool that internally
  spawns its own worker pool fights the scheduler for resources it
  doesn't own: it makes CPU and memory usage unpredictable from the
  outside, which is exactly what the scheduler needs to be predictable.
  Be a good managed process — single-threaded for CPU, predictable in
  your resource footprint — and let the orchestrator scale you.
  Free-threading (available from Python 3.14) removes the GIL
  constraint that was the last technical argument for
  `multiprocessing` — but it does not change the scaling model.
  Code should be compatible with free-threaded builds (the
  immutability discipline from section 3 already ensures this), not
  structured to exploit them for internal parallelism.
- **Immutability makes concurrency safe.** The discipline from section 3
  — frozen attrs objects, frozensets, tuples — means threads sharing data
  cannot corrupt each other. This is not a coincidence; it is the
  payoff. Mutable shared state between threads requires a lock, and a
  lock you forgot is a bug you find in production under load.

### Async

`asyncio` is permitted but not preferred. It is the right tool when the
program is fundamentally an event loop — many concurrent connections, a
long-lived server — and the wrong tool when `concurrent.futures` would
do. Do not introduce `async`/`await` for a handful of parallel HTTP
calls that a thread pool handles cleanly.

When async is used:

- **Lint for unawaited coroutines.** Enable ruff's `RUF006` rule (or
  equivalent) — a coroutine that is called but not awaited is silently
  discarded, producing a bug that is invisible at the call site and
  difficult to diagnose from symptoms alone.
- **Do not mix sync and async IO in the same call chain.** A blocking
  call inside an async function stalls the event loop. Use
  `loop.run_in_executor` to push blocking work to a thread, or keep the
  code synchronous throughout.
- **The entry point owns the event loop.** `asyncio.run()` is called
  once, at the top. No library code calls `asyncio.run()` or
  `loop.run_until_complete()`.

## 9. What _not_ to copy

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
