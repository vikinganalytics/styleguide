---
permalink: /shellguide
---
# Shell Style Guide for Tooling

This guide describes how we write shell scripts: orchestration layers
that wire programs together, thin wrappers around external tools, and
the glue that connects heterogeneous components into pipelines. Its
goal is scripts that are reliable units of work — safe to run by hand,
under CI, or as one of a thousand parallel invocations on a cluster.

## Principles

Three principles are the point of this guide; everything else is in
service of them:

1. **Strict defaults, so that common bugs fail loudly.** Shell is
   permissive by default — undefined variables expand silently, failed
   commands are ignored, broken pipelines continue. Strict mode
   (`set -euo pipefail`) and disciplined quoting make these bugs
   immediate, noisy failures instead of silent corruption. This is the
   closest shell gets to static safety: whole classes of bugs — typos
   in variable names, unchecked errors, word-splitting surprises —
   become impossible under strict defaults rather than lurking until
   production.
2. **Strong boundaries between moving parts.** A script communicates
   with the programs it calls — and with whatever calls it — through
   stdin, stdout, stderr, and exit codes. This process boundary is what
   makes shell scripts composable and safe: each component runs in its
   own process with its own state, and no component can reach into
   another's internals. The boundary is what makes shell useful as an
   orchestration layer; code that weakens it (sourcing, globals,
   parsing internal state) weakens the reason for using shell in the
   first place.
3. **Atomic side effects, so that incomplete work is never observable.**
   Every operation that changes state visible to another part of the
   system — a file, a directory, a deployment artifact — either
   completes fully or leaves no trace. Write to a temporary location
   and `mv` to the destination — never redirect directly to the output
   file. A script that is interrupted, killed by a scheduler, or
   terminated by a signal must not leave half-written files for the
   next invocation to discover.

### Design goal: one script, every environment

Shell scripts must behave identically in three settings:

1. **A developer laptop** (macOS, Ubuntu, or WSL) — interactive use,
   quick iteration, a single invocation at a time.
2. **A VM or HPC node** — manual or scripted runs on shared
   infrastructure, often with a shared filesystem.
3. **Scaled-out execution** — hundreds or thousands of concurrent
   invocations managed by an HPC scheduler (Slurm, PBS), an OS-level
   orchestrator (`xargs -P`, GNU parallel), or a job runner.

The same script, unchanged, must be correct in all three. Strict mode
and quoting prevent the bugs that surface only under concurrency or
unusual input. Atomic side effects prevent the corruption that surfaces
only when scripts run in parallel on a shared filesystem. Clean exit
codes and predictable resource usage make the script a good managed
process — safe for a scheduler to run, kill, and rerun without special
handling. A script that works on a laptop but breaks when a scheduler
runs fifty copies of it has failed the design goal.

### Properties that emerge — not code to write

The principles above, followed consistently, produce several properties
that are critical in scheduler-managed environments where scripts are
retried, rerun, and executed in parallel. These properties are _named
here so you can recognize when a design choice would break them_ — not
so you add code to achieve them. The moment a developer writes
`if [[ -f "$marker" ]]; then echo "skipping..."; return 0; fi`, the
system is worse than if they had not tried: the check is usually wrong
about what "already done" means, it is hard to test, and it makes the
script harder to reason about.

- **Idempotency.** A script that reads input, produces deterministic
  output, and writes via atomic replace (`mktemp` + `mv`) is already
  idempotent: running it twice with the same input produces the same
  file. No conditional logic, no skip-if-exists marker files — the
  second run overwrites identically. Pure stdin/stdout scripts are
  inherently rerunnable because there is no state to get confused by.
  If a design choice would make a script non-idempotent — appending
  without deduplication, generating non-deterministic identifiers,
  depending on wall-clock time — that is the signal to reconsider the
  design, not to add rerun-detection logic.
- **Pipeline-level correctness.** A stdin/stdout pipeline under
  `set -o pipefail` (`ingest | transform | load > output`) is already
  atomic at the pipeline level: if any stage fails, the whole pipeline
  fails and no output is produced — the final redirect or atomic
  replace never completes. This is a feature of the model, not
  something extra to build. The partial-state problem only arises when
  stages write to intermediate files between separate invocations —
  which is an orchestrator-level concern, not the script's. When a
  workflow does require intermediate state (stages run as separate
  jobs, not a single pipeline), the script's responsibility is to be
  safe to rerun: idempotent, deterministic, atomic in its own writes.
  A workflow manager that tracks which stages completed can rerun from
  the failed stage precisely because each script upholds these
  properties. Trying to solve multi-stage recovery inside the script
  is scope creep that produces the marker-file branching this section
  warns against.

Every rule below is either a consequence of these principles or a
convention adopted for consistency. The distinction matters in review:
a deviation that weakens a principle calls for a redesign, while a
deviation from a convention needs only a good reason. When rules seem
to conflict, resolve toward the principles.

## Precedence

When making a style or design decision, consult in this order:

1. **This guide.** Internal rules always win.
2. **The [Google Shell Style Guide](https://google.github.io/styleguide/shellguide.html)**,
   for anything this guide does not cover.

Not every rule applies to every script. A three-line wrapper does not
need argument parsing; a data pipeline does not need interactive
prompts. Skip the parts that do not apply — but when a part _does_
apply, follow it as written.

---

## 0. When to use shell — and when to stop

**Shell orchestrates programs. Something orchestrates the shell
script.** A pipeline like

```sh
uv run ingest.py < raw.jsonl \
  | uv run transform.py \
  | uv run load.py --target db \
  && notify "pipeline complete"
```

is good shell. Each component communicates through stdin/stdout and
exit codes, enforcing a process boundary that prevents them from
coupling to each other's internals. The shell script's job is to
connect them, not to do their work.

But that pipeline is itself a unit of work. A scheduler submits it, a
CI system runs it, an operator invokes it — or a thousand copies of it
run in parallel on an HPC cluster. This means the shell script is
subject to the same discipline it imposes on the programs it calls:
deterministic behavior, atomic side effects, predictable resource
usage, and a meaningful exit code. A script that works when run once
by hand but breaks under parallel execution or leaves half-written
output when interrupted has failed as an orchestrator — not because the
tools inside it are wrong, but because the script itself wasn't written
as a reliable unit of work.

### When to stop

Shell has no type system, no data structures beyond flat arrays, and
error handling that amounts to "fail or continue." These are not
limitations to work around — they are signals that the work belongs
somewhere else. Move to a proper programming language when:

- **The script needs data structures.** Associative arrays, nested
  structures, anything beyond a flat list of strings.
- **The script parses structured data.** JSON, YAML, CSV, XML — if
  the script is extracting fields, filtering records, or
  transforming structured input, it is doing business logic in a
  language that cannot check it.
- **The script has complex control flow.** Nested conditionals that
  make decisions about data, loops with error recovery, state machines.
- **The script exceeds roughly 100 lines.** Length is not a hard rule,
  but it is a reliable signal. A script that long is almost certainly
  doing work that belongs in a language with types and tests.

The most valuable judgment in shell scripting is knowing when to stop.

## 1. Tooling baseline

Style that is not enforced by a tool decays; put rules in
configuration, not in code review.

- **[ShellCheck](https://www.shellcheck.net/)** for static analysis.
  ShellCheck is the closest shell gets to a type checker — it catches
  quoting errors, undefined variables, useless uses of `cat`, and
  hundreds of other pitfalls that are invisible to the eye and silent
  at runtime. Run it in CI on every shell file. Every inline
  suppression (`# shellcheck disable=SC2059`) gets a comment
  explaining _why_ the warning does not apply — same discipline as
  the linter rules: the default is that ShellCheck is right.
- **[shfmt](https://github.com/mvdan/sh)** for formatting. Consistent
  indentation and syntax normalization, enforced mechanically. Run it
  in CI alongside ShellCheck.
- **bash, not sh.** Declare `#!/usr/bin/env bash` and use bash
  features — arrays, `[[ ]]`, `local`, process substitution. Portable
  POSIX shell sacrifices safety features (no arrays, no `local`, no
  `[[ ]]`) for compatibility with shells the script will never run
  under. Target the shell you actually use.

## 2. Strict mode

Every script begins with:

```bash
#!/usr/bin/env bash
set -euo pipefail
```

This is non-negotiable. Each flag closes a class of bugs:

- **`-e`** — exit immediately on a failed command. Without it, a
  script silently continues past errors, processing garbage or
  operating on stale state.
- **`-u`** — treat undefined variables as errors. Without it, a typo
  in a variable name (`$PAHT` instead of `$PATH`) expands to the
  empty string silently.
- **`-o pipefail`** — a pipeline fails if _any_ command in it fails,
  not just the last. Without it, `bad_command | grep pattern` succeeds
  as long as `grep` succeeds, hiding the failure of `bad_command`
  entirely.

Strict mode changes how you write shell. Some idioms that work under
permissive defaults break under `set -e`:

- A command that is _expected_ to fail (e.g. `grep` returning 1 for
  no match) must be guarded: `if grep -q pattern file; then ...` or
  `grep pattern file || true`.
- `set -e` does not catch failures inside command substitutions
  in all contexts. Assign and use separately:
  `result=$(cmd)` on its own line, not `local result=$(cmd)` where
  the `local` masks the exit code.

These adjustments are the cost of strict mode. They are worth paying.

## 3. Program structure

### The `main` pattern

All logic lives in functions. The script's top-level code is a single
call:

```bash
main() {
    # ...
}

main "$@"
```

This prevents top-level variables from leaking into functions,
mirrors the structure of the programs the script orchestrates, and
makes the script's entry point explicit.

### Arguments

Validate arguments at the top of `main`, before any work begins —
the shell equivalent of parsing argv into a validated context:

```bash
main() {
    if [[ $# -lt 2 ]]; then
        echo "Usage: $(basename "$0") <input> <output>" >&2
        return 2
    fi
    local -r input_file=$1
    local -r output_file=$2

    # ... work starts here
}
```

For scripts with flags, use `getopts` or a manual `while`/`case`
loop. In either case, parsing is complete before the first side effect.

### Functions

Functions receive their inputs as arguments, not through globals:

```bash
# Good — explicit inputs
process_file() {
    local -r input=$1
    local -r output=$2
    # ...
}

# Bad — implicit coupling through globals
process_file() {
    # uses $INPUT_FILE and $OUTPUT_FILE from the caller's scope
    # ...
}
```

Declare local variables with `local`. Use `local -r` for values that
do not change after assignment — it is not enforced deeply, but it
communicates intent and catches accidental reassignment.

### Output

**Output is stdout, stderr, and exit codes.** A shell script
communicates with its caller the same way the programs it orchestrates
communicate with it: data on stdout, diagnostics on stderr, success or
failure as an exit code.

- **stdout** — data output, and nothing else. If the script produces
  results that another program consumes, they go here.
- **stderr** — logs, progress, errors, and all other human-facing
  information. `echo "processing $file..." >&2`.
- **Exit codes** — 0 for success, nonzero for failure. Use distinct
  codes when the caller needs to distinguish failure modes: 1 for
  general errors, 2 for usage errors. The script's exit code is the
  exit code of `main`.

**Do not write scripts that expect to be sourced.** A sourced script
runs in the caller's shell — it shares variables, options, and
execution context. This collapses the process boundary and interacts
badly with strict mode, cleanup traps, and every assumption this guide
rests on. Environment configuration files (`.env`, `.zshrc`) are a
recognized exception, but they are a different kind of artifact — not
a tool, not orchestration.

## 4. Quoting and word safety

**Quote every expansion.** `"$var"`, `"$@"`, `"$(command)"`. Unquoted
expansions are word-split and glob-expanded by the shell — a filename
with a space becomes two arguments, a string containing `*` expands
to the contents of the current directory. These bugs are silent,
intermittent, and often destructive.

The rare intentional unquoted expansion gets a comment explaining why
— same discipline as a ShellCheck suppression:

```bash
# Deliberately unquoted: flags is a space-separated list of
# single-word options that must expand to separate arguments.
# shellcheck disable=SC2086
cmd $flags
```

### Arrays for lists

Use bash arrays for lists of things. Never store multiple items in a
space-separated string:

```bash
# Good
files=("$input_dir"/*.csv)
for f in "${files[@]}"; do
    process "$f"
done

# Bad — breaks on spaces in filenames
files=$(ls "$input_dir"/*.csv)
for f in $files; do
    process "$f"
done
```

When passing lists between programs, use newline-delimited or
null-delimited (`\0`) streams and `xargs`:

```bash
find "$dir" -name '*.csv' -print0 | xargs -0 -I{} process {}
```

### Filenames and paths

Never parse `ls` output. Use globs (`*.csv`) or `find`. Assume
filenames can contain spaces, quotes, and any character except null —
because they can.

## 5. External tools

Shell's composability comes from calling external programs. The
programs in a pipeline are the shell equivalent of function calls —
each takes input, produces output, and communicates success or failure
through its exit code.

**External tools are for selection, not computation.** `grep` to
filter lines, `jq -r '.field'` to extract a value, `awk '{print $3}'`
to pick a column, `sort` and `uniq` to order and deduplicate, `head`
and `tail` to slice, `cut` to split on a delimiter, `wc` to count.
These are the vocabulary of shell glue — simple, composable transforms
that select, filter, or rearrange data.

When the work inside a single tool invocation requires conditionals,
aggregation, or multi-step transformation, it has become business logic:

```bash
# This is a program written in the wrong language
jq '[.[] | select(.age > 30) | {name, dept: .department}]
    | group_by(.dept)
    | map({dept: .[0].dept, count: length})
    | sort_by(.count) | reverse' < data.json
```

Move it to a proper programming language, where a type system can
check it. The test: if you need to _debug_ the `jq` or `awk`
expression, it is too complex for shell.

The same principle applies to `sed` — a substitution is fine, a sed
script is not — and to any tool that has grown its own programming
language inside a command-line argument.

## 6. Errors and exit codes

### Fail fast

Strict mode (`set -e`) means the script exits on the first failure.
This is the right default. Internals fail; the top level reports.

When a command is expected to fail — `grep` finding no matches,
`diff` detecting differences — guard it explicitly:

```bash
if grep -q "pattern" "$file"; then
    echo "found" >&2
fi

# Or when you only care about continuing:
diff "$a" "$b" || true
```

### Exit codes

- **0** — success.
- **1** — general error.
- **2** — usage error (wrong arguments, missing input).

The exit code is the script's contract with its caller. It is how a
scheduler knows whether to retry, how a pipeline knows whether to
continue, how an operator knows whether to investigate. A script that
exits 0 on failure is lying to its orchestrator.

### Retries

**A script that fails exits with a nonzero code. The caller decides
whether to retry.** A Slurm job that fails gets resubmitted by the
scheduler; a pipeline reruns the failed step; an operator reads the
error and acts. Internal retry loops hide failures from the
orchestrator, make resource usage unpredictable from the outside, and
turn a clear "this failed" into an opaque hang.

The exception is transient failure within a single operation — a
network request to a flaky endpoint, a resource that is briefly
unavailable. Bounded retries are justified here, but keep them
minimal:

```bash
fetch_with_retry() {
    local -r url=$1
    local -r max_attempts=5
    local attempt=1
    while (( attempt <= max_attempts )); do
        if curl -sf "$url"; then
            return 0
        fi
        echo "attempt $attempt/$max_attempts failed, retrying..." >&2
        sleep $(( attempt * 2 ))
        (( attempt++ ))
    done
    return 1
}
```

If you find yourself adding retry wrappers around many calls, the
script is doing too much.

## 7. Resources and cleanup

### Temporary files

Create temporary files and directories with `mktemp`. Clean them up
with a `trap`:

```bash
main() {
    local tmpdir
    tmpdir=$(mktemp -d)
    trap 'rm -rf "$tmpdir"' EXIT

    # ... all work uses $tmpdir for scratch space
}
```

The `EXIT` trap fires on normal exit, `set -e` failures, and signals
— it is the shell equivalent of `try/finally`. Place it immediately
after creating the resource it cleans up.

Never hardcode `/tmp/myscript_output` or similar. Hardcoded paths
collide between concurrent invocations — violating the design goal
directly.

### Atomic writes

Never redirect directly to the destination file:

```bash
# Bad — if process_data fails, output.csv is truncated to zero
process_data > output.csv

# Good — output.csv is either the old version or the new version, never empty
tmpfile=$(mktemp output.XXXXXX)
process_data > "$tmpfile"
mv "$tmpfile" output.csv
```

The temporary file must live in the same directory (or at least the
same filesystem) as the destination — `mv` is only atomic within a
single filesystem.

### Signals

When the script manages long-running child processes, trap `SIGINT`
and `SIGTERM` to ensure cleanup:

```bash
cleanup() {
    # Kill child processes, remove temp files
    [[ -n "${child_pid:-}" ]] && kill "$child_pid" 2>/dev/null || true
    rm -rf "$tmpdir"
}
trap cleanup EXIT
```

A script that is safe to `kill` is a script that is safe to schedule.

## 8. Logging and diagnostics

All diagnostic output goes to stderr. Use a simple logging convention:

```bash
log() { echo "[$(basename "$0")] $*" >&2; }
warn() { echo "[$(basename "$0")] WARNING: $*" >&2; }
err() { echo "[$(basename "$0")] ERROR: $*" >&2; }
```

Include the script name — when the output of several scripts is
interleaved (a common situation under orchestration), unnamed
messages are useless.

**Do not log success at every step.** When a thousand copies of the
script run in parallel, per-step progress messages become noise that
obscures real problems. Log what matters: errors, warnings, and
significant milestones (started, finished, skipped).

## 9. Testing

Shell scripts are tested through their external behavior — the same
interface their callers use:

- **[bats-core](https://github.com/bats-core/bats-core)** as the test
  framework.
- Each test runs in an isolated temporary directory.
- Assert on exit codes, stdout content, stderr content, and files
  written — not on internal function calls or variable state.

```bash
@test "transform rejects empty input" {
    run bash transform.sh /dev/null output.csv
    [[ "$status" -eq 1 ]]
    [[ "$output" == *"empty input"* ]]
    [[ ! -f output.csv ]]
}
```

Test the script as a subprocess (`run bash script.sh`), not by
sourcing it into the test — the script's contract is its process
boundary, and that is what the tests should exercise.

## 10. What _not_ to copy

Some shell scripts — init systems, package managers, CI harnesses —
have a density of edge-case handling that comes from their domain:
signal juggling, lock files, platform detection, fallback paths. That
handling is intrinsic to those problems, not a virtue to emulate.

Do not cargo-cult defensive patterns from Stack Overflow into scripts
whose domain does not demand them. The order of preference is:

1. **Make the problem disappear.** Use `mktemp` instead of managing
   unique names. Use `set -e` instead of checking every exit code. Use
   arrays instead of escaping spaces.
2. **Make it the caller's problem.** Exit with a meaningful code and
   let the orchestrator retry, reschedule, or alert. Do not build
   recovery logic the script's caller already provides.
3. Only then, **handle it explicitly** — with a comment explaining why
   it can occur.

If you find yourself writing many special cases, first ask whether the
script should be a program in a language that can handle complexity
safely. The most important skill in shell is knowing when to leave it.
