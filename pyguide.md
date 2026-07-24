---
permalink: /pyguide
---

# Python Style Guide for Tooling

This guide describes how we build Python tooling: command-line tools,
developer utilities, and the libraries behind them. Its goal is a design
language that prevents common classes of errors by construction rather
than by defensive handling.

A recurring strategy throughout the guide is to **left-shift errors**:
move detection from production to tests, from tests to the type checker,
from the type checker to a state that cannot be constructed at all. A
type checker that runs in seconds over the entire codebase finds
problems you would otherwise discover in a slow test suite at best, or
in production — or not at all — at worst.

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
   _append_ (one complete record per write, as with jsonlines). Append
   is atomic in practice when each record is small enough to complete in
   a single `write()` syscall (well under the OS page size — a typical
   jsonlines record qualifies easily) and the file is opened with
   `O_APPEND`; for larger writes the kernel may split the operation, and
   concurrent appenders can interleave. Both patterns are valid; neither
   is universally correct — choose based on the concurrency model, not
   by default. Append carries a life-cycle
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

### Properties that emerge — not code to write

The principles above, followed consistently, produce several properties
that are critical in scheduler-managed environments where tools are
retried, rerun, and executed in parallel. These properties are _named
here so you can recognize when a design choice would break them_ — not
so you add code to achieve them. The moment a developer writes
`if is_rerun(): skip_this_step()`, the system is worse than if they
had not tried: the branching is usually wrong about what "already done"
means, it is hard to test, and it makes the tool harder to reason about.

- **Idempotency.** A tool that reads stdin, produces deterministic
  output, and writes via atomic replace (`tempfile` + `os.rename`) is
  already idempotent: running it twice with the same input produces the
  same file. No conditional logic, no "skip if exists" — the second run
  overwrites identically. Pure stdin/stdout tools are inherently
  rerunnable because there is no state to get confused by. If a design
  choice would make a tool non-idempotent — appending without
  deduplication, generating non-deterministic identifiers, depending on
  wall-clock time — that is the signal to reconsider the design, not to
  add rerun-detection logic.
- **Schema evolution.** Tools that produce and consume jsonlines records
  will, over time, add fields. The combination of `cattrs` structuring
  with `attrs` defaults handles this naturally: a new optional field
  with a default is backwards-compatible (old records missing the field
  structure successfully), and unknown fields in the input are ignored
  by default (old readers handle new records). This is not automatic
  robustness — it requires that new fields have sensible defaults, and
  that removed fields are deprecated through a period where they are
  optional before being dropped. But it requires _no special code_ — the
  existing `attrs` + `cattrs` patterns deliver it.
- **Pipeline-level correctness.** A stdin/stdout pipeline
  (`ingest | transform | load`) gets two properties from the model,
  not from extra code. First, `pipefail` (or its Python equivalent —
  checking subprocess exit codes) makes a mid-pipeline failure
  _detectable_: the caller knows something went wrong. Second, when
  each stage writes complete records (jsonlines), any partial output
  from a failed pipeline is a valid prefix — complete records of work
  that was done — not corrupt garbage. Together these mean the caller
  can safely discard and retry. Note that this is _not_ atomicity:
  a bare redirect (`> output`) creates and truncates the destination
  file before any stage runs, so partial output is observable on
  disk even if the pipeline fails. True all-or-nothing requires
  atomic replace at the end (`> "$tmpfile" && mv "$tmpfile" output`)
  — the same pattern used everywhere else. The partial-state problem
  across _separate invocations_ is an orchestrator-level concern, not
  the tool's. When a workflow requires intermediate state (stages run
  as separate jobs, not a single pipeline), the tool's responsibility
  is to be safe to rerun: idempotent, deterministic, atomic in its
  own writes. A workflow manager that tracks which stages completed
  can rerun from the failed stage precisely because each tool upholds
  these properties. Trying to solve multi-stage recovery inside the
  tool is scope creep that produces the `is_rerun()` branching this
  section warns against.

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

  Every `ignore` entry gets a comment explaining _why_ it is ignored.
  Inline suppressions use the specific rule code, never a bare `# noqa`.

  For example, CLI tools that use `print()` for data output (section 3)
  will typically ignore the `T20` family:

  ```toml
  [tool.ruff.lint]
  ignore = [
      "T20",  # print() is intentional — stdout is the data channel (section 3)
  ]
  ```

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

**Target a recent, stable Python version** — typically one minor behind
the latest release, giving the ecosystem time to catch up, but not
rigidly so. We adopted 3.14 roughly eight months after launch; the
"one behind" heuristic is approximate, not a rule. What matters is that
the version is well supported by the type checker, attrs, and the
libraries we depend on.

### Packaging and distribution

Most tools are distributed as source and run where they land — on a
developer laptop, a shared worker, a compute node, or occasionally inside
a container. Wheels and PyPI-style distribution are uncommon but not
unheard of. The common delivery paths are:

1. **Source checkout + `uv run`.** The repository is cloned (or mounted
   as a git submodule) and invoked directly: `uv run mytool ...`. This
   is the simplest model and the one the rest of the guide assumes.
2. **Containers.** Some infrastructure (GCP/GKE, CI runners) uses
   container images. A Dockerfile may exist at the repo root, but its
   presence does not mean all tooling in the repo will be run through
   it — it is one deployment path among several. The tool must not
   assume it is running inside a container, and the container must not
   assume it is the only way the tool will be invoked.
3. **`uv tool install`** for developer-local utilities — formatters,
   linters, one-off scripts. This works well when the tool is
   self-contained and used interactively; it is not appropriate for
   production or cluster deployment.

The tool must be safe to run on shared infrastructure regardless of
the deployment mechanism. A tool that is only safe when it has a machine
to itself has failed the design goal from the principles section — the
isolation guarantees come from the tool's design (immutability, atomicity,
stdin/stdout composability), not from the environment it happens to land
in. Containers, VMs, and dedicated nodes are operational conveniences;
they are not correctness mechanisms and must not be relied on as such.

Every project has a `pyproject.toml` at its root. `setup.py` is
forbidden — it is executable code masquerading as configuration,
impossible to analyze statically, and unnecessary since PEP 517.
**Use `pyproject.toml` for all Python tool configuration** — ruff,
pytest, type checker, coverage, import-linter. Do not use standalone
config files (`ruff.toml`, `pytest.ini`, `.flake8`, `mypy.ini`) when
the tool supports `pyproject.toml`. One file, one place to look; a
sprawl of dotfiles at the repo root is a sign that configuration is
being accumulated rather than managed. When a
monorepo contains multiple tools, each has its own `pyproject.toml` in a
subdirectory, and cross-tool dependencies are expressed as path
dependencies or git submodules — not as published packages with version
constraints, unless the tool is genuinely distributed independently.
Each submodule or subdirectory project owns its own configuration; a
parent repo does not override a child's formatting or linting decisions.

### Module API surface

**`__init__.py` is not encouraged.** Implicit namespace packages
(directories without `__init__.py`) are the default. An `__init__.py`
that re-exports symbols creates a second name for everything it touches,
complicates refactoring, and makes import-linter contracts harder to
reason about. If a module needs initialization code, that code belongs in
the entry point, not in import-time side effects.

**Prefer CLI boundaries over library imports.** When one tool needs
another tool's functionality, the default is to invoke it as a
subprocess — stdin/stdout, exit codes, the same interface a human uses.
This keeps the tools independently deployable, independently testable,
and free of version-coupling. A shared library is justified when the
overhead of serialization is genuinely prohibitive or when the shared
code is pure computation with no IO — but it should be extracted into its
own package with its own `pyproject.toml`, not imported across tool
boundaries within a monorepo.

When code _is_ imported as a library, define `__all__` in every public
module to make the API surface explicit. Without `__all__`, every
top-level name is public by convention, and consumers will eventually
depend on names that were never meant to be stable. `__all__` is the
module-level equivalent of a frozen attrs class: it declares what
crosses the boundary, so that everything else can change freely.

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
  injected (section 9), never reached for globally. Resources are the
  deliberate exception to the immutability rules of section 5 — stateful by
  nature, which is exactly why they are singular and passed visibly. But add
  such a parameter when the tool actually acquires the resource, not
  speculatively (section 11).

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
          case unreachable:
              typing.assert_never(unreachable)
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

What deserves to live one layer down is _knowledge_, not lookalike
code. A fact of the system — a flag's definition, a path convention, a
record schema — must have exactly one home, because two homes will
eventually disagree; that is what the shared `Argument` enum is
(section 2). Code that merely looks similar is a different matter
entirely. An abstraction must be justified by its own design — a
boundary that would be genuinely good even if it had only one caller —
never by the number of duplicate sites it would absorb. Ten sites of
honest boilerplate are cheaper than one abstraction that fits none of
them well: the boilerplate stays local, independent, and free to
diverge as the commands evolve, while a bad shared abstraction taxes
every caller forever and punishes the next person who needs it to bend.
The world is full of boilerplate, and sometimes that is fine. If the
genuinely good abstraction exists, extract it; if it doesn't, the
duplication is not a debt — it is the honest representation of code
that happens to look alike today and has no deeper unity.

## 3. Streams and IO

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
mutable state — they race, easily, in exactly the way section 5 teaches us
not to trust. A tool that reads stdin and writes stdout leaves the choice
to the user, who can still bring files when they want the statefulness:

```sh
uv run tool.py < input.jsonl > output.jsonl
```

Accepting a path argument is an occasionally-justified convenience;
_requiring_ one is a design smell.

## 4. Logging

All log output goes to stderr — stdout is for data (section 3). Use the
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

## 5. Data and types

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
- **Every parameter, return type, and assignment the checker cannot infer
  carries an explicit annotation.** There are no exceptions — not "it's
  obvious," not "it's just a helper," not "it's a script." An unannotated
  function is an unchecked function: the type checker cannot verify
  callers, cannot catch broken refactors, and cannot propagate constraints
  into downstream code. One missing annotation is one hole, and holes
  compound. The `ANN` ruff rules (section 1) reject unannotated code
  mechanically; this rule explains _why_ the linter is right to reject it.

  **Annotate what you need — not necessarily what you know you'll be
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

### Inheritance

**Implementation inheritance is the strongest form of coupling this
guide has occasion to warn against, and it is strongly discouraged.**
A subclass does not consume its parent through a contract; it reaches
into the parent's implementation — overriding methods the parent calls
on itself, depending on attribute initialization order, breaking when
a detail the parent considered private changes. Full annotation
(section 5) does not save you here, because the problem is not
unchecked code: every line of both classes is checked, but the
dependency a subclass takes on its parent's self-call structure and
initialization order has no type-level representation, so there is
nothing for the checker to verify. This is the failure mode this
section exists to prevent — implementation shared with code you don't
control — made structural and permanent, and placed beyond the reach
of the tools that guard every other boundary. The colleague who
rewrites the parent six months from now cannot see your subclass's
assumptions, and nothing will tell either of you what broke.

Hierarchies are therefore capped at two levels: an abstract interface
— a `Protocol`, or an ABC when runtime `isinstance` checks are
genuinely needed — and its implementations. Never deeper. The abstract
base is a type-checkable template declaring what implementations
expose to the outside world, and never anything more: no logic, no
state, no helper methods for implementations to inherit. A base class
that accumulates shared logic is never needed, because everything it
would hold already has a better home — shared behavior is a plain
function in a utils module that both implementations call, shared
state is a frozen attrs object that both compose as a field, and
variants of a concept are a tagged union (section 6). Subclassing
`Enum` and `Exception` is fine — those are the stdlib's designated
extension points and carry no implementation to couple to.

## 6. Type precision

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

  The `FBT` rules already require this for booleans (section 7); apply
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

## 7. Control flow and idiom

Every idiom in this section exists because of a specific failure mode it
closes — not because the syntax is newer or more fashionable. The
connecting thread is the same left-shift strategy as everywhere else:
prefer the construct that lets the type checker or the language itself
reject the mistake, over one that relies on the developer noticing it in
review.

- **`match`/`case` for dispatch on shape or value** — argv, enums,
  `(suffix, mode)` tuples. Prefer it over if/elif chains when there are
  three or more arms or the arms destructure. The reason is not cosmetic:
  `match` interacts with the type checker in a way that `if/elif` does
  not. When the subject is a union or enum, a final `assert_never` arm
  (section 6) turns an incomplete `match` into a type error — the checker
  finds every site a new variant must touch, instead of an if/elif chain
  silently falling through to a default that was never meant to handle it.
  The exception is argv dispatch, where the final arm is the
  unknown-command error — an open set by nature.

- **Walrus operator for check-and-bind** (`if (path := os.getenv(...)):`).
  The alternative is two statements: assign, then check. That separation
  creates a window where the unchecked value is in scope and usable — a
  gap between "obtained" and "validated" that the boundary philosophy
  (section 5) exists to eliminate. The walrus closes the gap: the binding
  only enters scope inside the branch where the check has passed. This is
  a small instance of the same principle that drives converter-at-construction:
  never let unvalidated data exist in a reachable name.

- **`contextlib.suppress(SpecificError)` over `try/except SpecificError: pass`.**
  A `try` block has a _spatial_ catch surface — everything between `try:`
  and `except` is covered, including lines added later by a maintainer who
  did not notice the exception handler. `suppress` wraps a single
  statement, making the scope of the silencing lexically obvious and
  resistant to scope creep. The principle is the same one behind narrow
  type annotations: say exactly what you mean, so that accidental
  expansion is visible.

- **Generators for streams of work** (events, matches, records). A
  function that builds a list internally and returns it has committed to
  a representation (list), a lifecycle (everything in memory at once),
  and a policy (all-or-nothing) on behalf of a caller who may want none
  of those things. A generator defers all three decisions: the caller
  collects into a list if it needs one, iterates lazily if it doesn't,
  and can stop early without the producer finishing. This is the
  producer-side application of the boundary principle: hand out the
  narrowest commitment — a single-pass iterable — and let the consumer
  choose. It also connects directly to the streaming model in section 3:
  a tool that yields jsonlines records one at a time can flush each to
  stdout immediately, keeping memory bounded and ensuring that work
  completed before an interruption is never lost.

- **Return a value or perform an effect — never both.** Every function
  picks a side. A _query_ computes and returns a result, touching
  nothing; a _command_ performs its effect and returns `None`.
  Mutation is already the discouraged option (section 5) — most
  functions should be queries — but where an effect is genuinely the
  point, keep it unmixed. A function that mutates _and_ returns a
  result has a signature that no longer tells the whole truth: the
  type checker sees and verifies the return type while the effect
  travels invisibly beside it, so callers come to rely on the return
  value while overlooking the effect, or the reverse. The split also
  serves principle 3 directly: a pure query is trivially safe to call
  again, and a command that does exactly one thing is the unit that
  the atomic patterns of section 9 can actually make atomic — an
  operation smeared across a computation has no single point at which
  to commit. The entry-point convention `main(ctx) -> int` is not an
  exception: the exit code is the command's status report, not a
  computed result.

## 8. Errors

- **One translation point.** The top-level `main()` is the only place that
  catches domain exceptions and turns them into stderr messages and exit
  codes. Internals raise; they do not print and continue.
- Re-raise across layers with context: catch `ValueError` from attrs
  validation and raise the domain error `from exc`.

### Retries

**A tool that fails exits with a nonzero code. The caller decides whether
to retry.** This is the retry equivalent of the scaling model (section 11):
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

## 9. Resources and external state

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
- **Restore `SIGPIPE` to its default behavior** at the entry point:

  ```python
  import signal
  signal.signal(signal.SIGPIPE, signal.SIG_DFL)
  ```

  Python sets `SIGPIPE` to `SIG_IGN` at startup, which means writing to
  a closed pipe raises `BrokenPipeError` instead of killing the process
  quietly. Without this, `tool | head -5` produces a traceback after
  `head` closes the pipe. Restoring the default makes the process die
  silently on `SIGPIPE`, like every other Unix tool. Stdout is data and
  pipes are the composition model — a tool that breaks when piped
  through `head` is not composable.

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
    does. On network and distributed filesystems (NFS, Lustre, GPFS)
    the atomicity guarantee is weaker — a `rename()` may be visible to
    some clients before others, or both names may be briefly visible.
    This does not change the advice: atomic replace is still the
    best available primitive. Be aware of the limitation when
    targeting shared cluster storage.
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
  [hardened example](https://github.com/olle-lindgren/cixstack/blob/5282850f5057c52e73e078c9ff59ac491320c94f/cixstack/utils.py)
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

## 10. Verification and testing

The left-shift strategy that governs production code applies with equal
force to the code that verifies it. Just as the production application
is code that validates and transforms state, the developer environment
is code that transforms code — the details differ, but a mindset that
succeeds in one will succeed in the other. A verification pipeline that
takes thirty minutes to tell you about a typo has the same design
failure as a production pipeline that takes thirty minutes to tell you
about an invalid input: the feedback is correct but arrives too late to
be useful, so people route around it.

### The verification pipeline

Verification is a pipeline of stages, ordered by speed and by cost of
the infrastructure they require. Each stage catches a class of errors
that the stages before it cannot, and each stage is slower than the one
before it. The developer's job is to push as much assurance as possible
into the earliest, fastest stages — and the codebase's job is to make
that possible.

```
format  →  lint  →  type check  →  unit test  →  integration test  →  postmerge
```

1. **Format** (ruff format). Subsecond. Rewrites style violations
   mechanically. No human time should ever be spent on formatting
   disagreements — the formatter is the authority, and it is fast enough
   to run on every save.
2. **Lint** (ruff check). Subsecond. Catches mechanical errors —
   unused imports, unreachable code, banned patterns — that are beneath
   the type checker's concern. A lint failure is a local fix, never a
   design discussion.
3. **Type check** (ty). Seconds over the entire codebase. Verifies that
   contracts hold across every call site, every branch, every module
   boundary. This is the stage where the investment in precise types
   (sections 5 and 6) pays off: the type checker finds the bug that a
   test would need a specific input to trigger, and it finds it in every
   code path simultaneously, not just the ones the tests happen to
   exercise. A type checker that runs in seconds over the full codebase
   is fast enough to run on every save; one that takes minutes is not,
   and the assurance it provides arrives too late.
4. **Unit tests** (pytest). Seconds — under one second for the files you
   changed, under a minute for the full suite. No external
   dependencies: no database, no network, no container, no
   `docker compose up`. A developer runs a single test file with
   `pytest path/to/test.py` and nothing else running. This is the
   fastest stage that exercises runtime behavior, and its speed is what
   makes incremental development possible — change a function, run its
   tests, see the result, iterate. Everything that can be verified here
   _must_ be verified here.
5. **Integration tests**. Minutes to tens of minutes. Requires local
   services — a database, a message queue, a container stack. Tests
   contracts across real infrastructure boundaries: does the query
   actually work against the real database engine, does the serialized
   message parse on the other side. This is the first stage where
   feedback is slow enough to interrupt flow, and the boundary between
   "I run this on every change" and "I run this before I push."
6. **Postmerge / system tests**. Minutes to hours. The full production
   stack, end-to-end scenarios, environments that cannot be reproduced
   locally. Runs in CI after merge, or in a staging environment.
   Failures here are expensive — they block the pipeline and require
   context-switching back to code the developer has mentally left
   behind.

The time budget in each tier is minimized ruthlessly. ruff makes
formatting and linting fast. ty makes type checking fast. But unit test
speed is not a property of the test framework — it is a property of the
code's design. A codebase where business logic is entangled with
database clients and HTTP sessions cannot have fast unit tests no matter
what test runner it uses. The investment is in the production code:
inject dependencies (section 9), type interfaces as protocols, keep
logic pure. The fast tests follow.

### What makes a unit test a unit test

**A test's infrastructure requirements must not exceed what the code
under test actually touches.** A function that transforms a dict does
not need MongoDB. A service method that dispatches work based on
configuration does not need a message broker. If the test requires
infrastructure the code does not, one of two things is true: the test is
testing the wrong thing, or the code has a dependency it should not.

A "unit" test that requires `docker compose up` is an integration test
that has been misfiled. The label matters because it poisons the tier:
one test file that needs a running database means _no_ unit test can run
without the full stack, the tier's speed guarantee is broken, and
developers stop running it locally. The damage is not to that one test —
it is to every test in the directory.

**Design for injection, not for mocking infrastructure.** The reason a
test needs a database is almost never that the logic under test
genuinely requires one — it is that the logic reaches for a database
client it was handed at module level or constructs internally. Section
9's injection discipline solves this at the source: when the database is
a parameter typed as `MutableMapping` or a `Protocol`, the test passes a
dict. No patching, no test containers, no startup cost. If you cannot
substitute a plain in-memory object for a dependency in a unit test,
that dependency's interface is too wide — narrow it. For those of you
wearing an architect hat, now is the time to pause a moment and think
about how your tech stack forms its verification infrastructure - code
that expects a redis client can get a dictionary, but for something
expecting a psycopg.Connection, faking it is not so easy now, is it?
Make your architecture calls accordingly.

Guard the tier boundary in CI: the unit tier should run in a stripped
environment where no external service is reachable, so that a test that
silently depends on infrastructure fails immediately and visibly — not
six months later when someone tries to run `pytest tests/unit/` on a
train.

### The cost equation

The cost of a test is not what it costs to write — it is what it costs
to _run_, multiplied by every developer, every branch, every CI
invocation, for the lifetime of the project. A 90-second integration
test is justified when it guards a real integration boundary. The same
90 seconds spent testing a pure function — because the fixture happened
to be convenient, or because "we already have the database up anyway" —
is waste that compounds silently. Nobody notices the moment unit tests
cross from one second to one minute; everybody notices when the full
suite takes forty-five minutes and developers stop running it.

The strategy is the same as in production code: when something is too
slow, do not optimize — restructure so the expensive operation is not
needed. Do not make integration tests faster; make them fewer, by moving
the assurance they provide into the unit tier. Every function you
extract from a database-dependent method into a pure function is a
function whose tests drop from the integration tier to the unit tier —
minutes become milliseconds, and the coverage _improves_, because a unit
test can exercise edge cases that an integration test is too slow to
bother with.

### Testing rules

- **pytest**, with `tmp_path` for filesystem work. Because resources and
  locations are injected (section 9), tests rarely need patching at all:
  hand in `{}` for the cache, `tmp_path` for the cache directory. Reaching
  for `unittest.mock.patch` is a signal that something should have been
  injected — and never monkeypatch internals of the unit under test.
  The substitute must honor the protocol's _semantics_, not just its
  shape: `{}` works as the cache because a dict is a real
  `MutableMapping` with real behavior, so the test exercises the same
  contract production code runs against. A `Mock` satisfies the type
  checker and nothing else — it returns a `Mock` from `.get()`,
  accepts calls no real backend would accept, and lets a test pass
  against behavior that exists nowhere in production.
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
- **Separate test tiers by infrastructure cost.** A typical layout:
  - `tests/unit/` — no external dependencies, runs in a sandbox, fast
    enough to run on every save.
  - `tests/integration/` — requires local services (a database, a
    message queue), tests contracts across boundaries.
  - `tests/system/` — requires the full stack, tests end-to-end
    behavior.

  CI runs all tiers. The tiers are independently runnable — `pytest
tests/unit/` succeeds with nothing else running, always.

## 11. Concurrency

### The scaling model: more instances, not more threads

When a tool needs to go faster or handle more work, the answer is
running more instances of the program in parallel — not making a single
instance internally concurrent. Each instance reads its own input, writes
its own output, and exits. The orchestrator (a shell loop, a job
scheduler, `xargs -P`) decides the parallelism.

This has a direct consequence for how you write your tool: **design for
single-process correctness first.** Stdin/stdout composability,
idempotent operations, and atomic filesystem writes (section 9) are what
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
  resource (section 9).
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
  immutability discipline from section 5 already ensures this), not
  structured to exploit them for internal parallelism.
- **Immutability makes concurrency safe.** The discipline from section 5
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

## 12. What _not_ to copy

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

The same warning applies to design ideas imported from object-oriented
tradition. "Open for extension, closed for modification" — plugin
registries, hook systems, subclass-to-extend frameworks — optimizes
for adding a variant without touching existing code. This guide wants
the opposite. Exhaustive `match` with `assert_never` (section 6) makes
a new variant _deliberately_ break every site that must handle it, at
type-check time: the checker hands you the complete list of places the
change must touch. Extension without modification sounds like a virtue
and is here the failure mode — the new variant flows silently into
default arms and generic handlers that were never designed, tested, or
reasoned about with it in mind.

If you find yourself writing many special cases, first ask whether the
design can be changed so they cannot happen. That principle — not the edge
cases themselves — is what this guide asks you to reproduce.
