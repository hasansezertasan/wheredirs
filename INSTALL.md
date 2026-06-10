# Generating wheredirs

`wheredirs` is a **ghost library** — it ships a specification and test cases instead of executable code. To use it, give the files to an AI coding assistant and tell it what language you want.

## Quick start

Hand your coding assistant the prompt below, after **replacing `[LANGUAGE]`** with the language you want (e.g. `Python`, `Rust`, `Go`, `TypeScript`). The generated implementation and its test runner will be written into the current working directory unless you tell the assistant otherwise — adjust the prompt's first line if you want them in a subdirectory or a sibling repo.

```
Implement the wheredirs library in [LANGUAGE].

1. Read SPEC.md for the complete behavioral specification.
   Treat SPEC.md as the contract; do not infer behavior from
   platformdirs, etcetera, or any other reference library.
2. Parse tests.yaml and generate a test file. Each top-level key
   (which_strategy, xdg, apple, windows, single-folder, errors,
   project) is a group of cases.
3. Implement exactly these seven public functions, and no others
   — the six directory resolvers plus the strategy-picker helper:
     config_dir, data_dir, cache_dir, state_dir, logs_dir,
     runtime_dir, which_strategy
   matching the common signature in SPEC.md (the six dir
   resolvers share one shape; which_strategy takes a single
   platform string and returns a strategy string). Do NOT add
   helpers like site_config_dir, user_documents_dir, etc. — see
   SPEC.md "Out of scope".
4. Implementation rules:
   - Pure functions; no filesystem I/O, no directory creation.
   - Forward slashes in returned paths (the test runner normalises).
   - Auto-mode (strategy=None) picks the native layout on every
     platform: xdg on Linux/BSD, apple on macOS/darwin, windows
     on Windows. Auto never picks single-folder. Explicit strategy
     overrides platform — strategy="xdg" on macOS returns XDG
     paths, strategy="windows" on Linux returns Windows paths, etc.
   - Cases with `raises: true` must raise an idiomatic exception
     in your language.
   - Cases with `output: null` mean the function must return the
     language's null/none value (Python None, Rust Option::None,
     TypeScript null; Go pick one signature shape per the
     per-language notes below and stick to it). This applies to
     state_dir and runtime_dir on Apple and Windows — those
     platforms have no equivalent concept and wheredirs surfaces
     that asymmetry rather than aliasing to another directory.
     Callers fall back via `state_dir(...) ?? data_dir(...)`
     if they want a path. The return type accommodates null
     uniformly across all six dir resolvers (not just state_dir
     and runtime_dir) so the public API is a single shape — this
     is an API-uniformity choice, not a claim that config_dir
     can ever return null in practice.
   - which_strategy accepts only the platform names enumerated
     in SPEC.md. Matching is case-sensitive. Empty string, null,
     mixed-case ("Linux"), and unknown values raise.
5. Public API: the six dir functions take only the user-facing
   parameters in SPEC.md (app, author, strategy).
   They MUST read platform / env vars from the process environment
   for the public call path. For testability, expose a parallel
   internal entry point named exactly `_resolve(platform, env,
   function_name, args)` (or `wheredirs._resolve` / module-private
   equivalent in your language) that the test runner can call
   with the case's pinned platform and env dict — do NOT
   monkeypatch os.environ from the runner. Keep both paths thin
   wrappers around one pure resolver. The runner contract below
   assumes this name; if you pick a different one, document it
   in the generated README so a future test sweep can find it.
6. Run the test suite until every case passes. The runner MUST exit
   non-zero if any case fails, print one line per failure with the
   case `name` plus expected vs actual, and print a final summary
   `passed N / total M`. That contract is what makes "did the
   generated impl pass" mechanically checkable.
```

### Language-idiom notes

The prompt is deliberately language-agnostic but a few things should be picked per language so the generated impl is usable:

- **Python**: package as a single module `wheredirs.py` plus a `pyproject.toml` (use `uv`); errors are `ValueError`; signatures like `config_dir(app: str, author: str | None = None, *, strategy: str | None = None) -> str | None`. The `str | None` return is uniform across all six resolvers (only `state_dir` / `runtime_dir` actually return `None` in practice — `config_dir` etc. either return a string or raise — but one signature shape keeps the public API uniform). Test runner via `pytest`.
- **Rust**: a single crate with one `lib.rs`; errors via `Result<Option<String>, WheredirsError>` (no panics) so the `Ok(None)` arm represents the `output: null` cases and `Err(_)` represents `raises: true` cases. The test runner is `cargo test` reading `tests.yaml` via `serde_yaml`.
- **Go**: package `wheredirs` with `(string, error)` returns — there is no exception equivalent, so "raises" cases assert the error is non-nil. **Use `error`, never `panic`** — panicking is not the Go idiom for "raises" and would break callers wrapping these functions. To represent `output: null` (Apple/Windows `state_dir`/`runtime_dir`) without conflating "no value" with "error", expose those as `(string, bool, error)` *or* return the empty string + `nil` error and document the convention in your generated README — pick one and apply it consistently. Test runner is `go test` reading via `gopkg.in/yaml.v3`.
- **TypeScript / Node**: single file + a `package.json` (use `bun`); errors are `throw new Error(...)`; signatures return `string | null`. Test runner with `vitest` or `bun test`.

For any language not listed: pick the conventional way to (a) package one module / one crate, (b) signal an error per the language's idiom, (c) run unit tests. Avoid inventing project structure beyond that.

### Test runner contract

There are two distinct things the runner does with `env`. Don't conflate them:

- **(a) Building the call inputs.** For dir-resolver cases, pass `env` straight through to the impl's `_resolve(platform, env, ...)` entry point. Variables absent from `env` are simply not in the dict the impl receives — they are "unset" from the impl's point of view. Do NOT mutate `os.environ`.
- **(b) Substituting placeholders in `output`.** Walk the expected-output string and replace each `$VAR` whose `VAR` is a key in `env`. `$VAR` placeholders whose name is NOT in `env` are left as literal text (a common source of false negatives: do not coerce missing names to empty string).

When the runner compares an implementation's return value to a case's `output`:

1. Substitute `$VAR` placeholders in `output` using `env`, longest name first (e.g. `$XDG_CONFIG_HOME` before `$HOME`).
2. Normalise both sides to forward slashes (`s.replace("\\", "/")`). Skipping this step will spuriously fail every Windows case if the implementation returns native separators.
3. Collapse runs of `/` to a single `/` on both sides. Needed for cases that pin env values with trailing slashes (e.g. `HOME: "/home/alice/"` substituted into `"$HOME/.config/MyApp"` would otherwise compare as `/home/alice//.config/MyApp`).
4. Strip any trailing `/` from both sides before comparison.

The `args` schema per function:

- **which_strategy** cases use `args` as a **list**: `args: ["linux"]`. Call as `which_strategy(args[0])`.
- **All six dir resolvers** use `args` as a **map**: `args: { app: "MyApp", author: "Acme" }`. Call as `<function>(**args)` (Python keyword-spread) or the equivalent in your language. `author` is optional — omit it if absent from the map.

## Verification

After generation, the runner should pass **every** case in `tests.yaml`. The current case counts are indicative — they change whenever a case is added — but as of this writing:

| Group             | Cases (indicative) |
| ----------------- | ------------------ |
| `which_strategy`  |  12                |
| `xdg`             |  33                |
| `apple`           |  21                |
| `windows`         |  22                |
| `single-folder`   |  14                |
| `errors`          |  16                |
| `project`         |   8+               |

Don't gate the implementation on these numbers — gate it on "every case passes". The table exists so you can spot wildly off-base parsing (e.g. only 2 cases loading from `xdg`).

Add your own cases in the `project:` block at the bottom of `tests.yaml` — those are the ones most likely to catch a generated implementation that's *almost* right for your real apps.

## Per-case runner steps

For each case:

1. Read `function`, `args`, `strategy`, `platform`, `env`.
2. Call the implementation's internal entry point with `platform` and `env` as explicit inputs (per "Public API" rule above). Do NOT mutate `os.environ` from the runner — that creates cross-test bleed when cases run in parallel and forces the runner to know which env vars to clear between cases. The internal entry point's signature should take exactly the per-case inputs.
3. `which_strategy` cases have no `platform` / `env` — `args` is a list with one element (`args: ["linux"]`); call as `which_strategy(args[0])`.
4. If `raises: true`, assert calling the function raises an error in your language's idiomatic style. Don't assert on the error type or message — only that it raised.
5. If `output: null`, assert the function returns the language's null/none value — no exception raised, no string returned.
6. Otherwise, call the function with the args and compare to `output` after substituting `$VAR` placeholders from `env`.
7. Apply the normalisation steps from the "Test runner contract" section above: `\` → `/`, collapse `/+`, strip trailing `/`.

## Out of scope (don't let your agent invent these)

`wheredirs` deliberately stops at the seven functions listed above. If your generating agent starts adding `site_config_dir`, `user_documents_dir`, `user_downloads_dir`, `XDG_CONFIG_DIRS` handling, `%PROGRAMDATA%` lookups, or directory-creation helpers, push back — those belong in a heavier library like `platformdirs`, not here. See SPEC.md "Out of scope for v0.1".
