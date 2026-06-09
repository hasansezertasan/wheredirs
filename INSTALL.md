# Generating wheredirs

`wheredirs` is a **ghost library** — it ships a specification and test cases instead of executable code. To use it, give the files to an AI coding assistant and tell it what language you want.

## Quick start

Hand your coding assistant this prompt:

```
Implement the wheredirs library in [LANGUAGE].

1. Read SPEC.md for the complete behavioral specification.
   Treat SPEC.md as the contract; do not infer behavior from
   platformdirs, etcetera, or any other reference library.
2. Parse tests.yaml and generate a test file. Each top-level key
   (which_strategy, xdg, apple, windows, single-folder, errors,
   project) is a group of cases.
3. Implement exactly these seven functions, and no others:
     config_dir, data_dir, cache_dir, state_dir, logs_dir,
     runtime_dir, which_strategy
   matching the common signature in SPEC.md. Do NOT add helpers
   like site_config_dir, user_documents_dir, etc. — see SPEC.md
   "Out of scope".
4. Implementation rules:
   - Pure functions; no filesystem I/O, no directory creation.
   - Forward slashes in returned paths (the test runner normalises).
   - Auto-mode (strategy=None) picks the native layout on every
     platform: xdg on Linux/BSD, apple on macOS/darwin, windows
     on Windows. Auto never picks single-folder. Explicit strategy
     overrides platform — strategy="xdg" on macOS returns XDG
     paths, strategy="windows" on Linux returns Windows paths, etc.
   - Substitute env-var placeholders ($HOME, $XDG_*, $APPDATA,
     $LOCALAPPDATA, $USERPROFILE) in expected outputs from each
     case's `env` block before comparing. Substitute the
     longest-named placeholder first (e.g. $XDG_CONFIG_HOME
     before $HOME); a naive replace will corrupt other names.
   - Variables absent from `env` are unset.
   - Cases with `raises: true` must raise an idiomatic exception
     in your language.
   - Cases with `output: null` mean the function must return the
     language's null/none value (Python None, Rust Option::None,
     Go zero string + ok=false, TypeScript null). This applies to
     state_dir and runtime_dir on Apple and Windows — those
     platforms have no equivalent concept and wheredirs surfaces
     that asymmetry rather than aliasing to another directory.
     Return type must accommodate null; callers fall back via
     `state_dir(...) ?? data_dir(...)` if they want a path.
   - which_strategy accepts only the platform names enumerated
     in SPEC.md. Empty string, null, and unknown values raise.
5. Public API: the six dir functions take only the user-facing
   parameters in SPEC.md (app, author, strategy).
   They MUST read platform / env vars from the process environment
   for the public call path. For testability, expose a parallel
   internal entry point (e.g. `_resolve(platform, env, ...)`) that
   the test runner can call with the case's pinned platform and env
   dict — do NOT monkeypatch os.environ from the runner. Keep both
   paths thin wrappers around one pure resolver.
6. Run the test suite until every case passes. The runner MUST exit
   non-zero if any case fails, print one line per failure with the
   case `name` plus expected vs actual, and print a final summary
   `passed N / total M`. That contract is what makes "did the
   generated impl pass" mechanically checkable.
```

### Language-idiom notes

The prompt is deliberately language-agnostic but a few things should be picked per language so the generated impl is usable:

- **Python**: package as a single module `wheredirs.py` plus a `pyproject.toml` (use `uv`); errors are `ValueError`; signatures like `config_dir(app: str, author: str | None = None, *, strategy: str | None = None) -> str | None`; test runner via `pytest`.
- **Rust**: a single crate with one `lib.rs`; errors via `Result<String, WheredirsError>` (no panics); the test runner is `cargo test` reading `tests.yaml` via `serde_yaml`.
- **Go**: package `wheredirs` with `(string, error)` returns — there is no exception equivalent, so "raises" cases assert the error is non-nil; test runner is `go test` reading via `gopkg.in/yaml.v3`.
- **TypeScript / Node**: single file + a `package.json` (use `bun`); errors are `throw new Error(...)`; test runner with `vitest` or `bun test`.

For any language not listed: pick the conventional way to (a) package one module / one crate, (b) signal an error per the language's idiom, (c) run unit tests. Avoid inventing project structure beyond that.

### Test runner contract

When the runner compares an implementation's return value to a case's `output`:

1. Substitute `$VAR` placeholders in `output` using `env`, longest name first.
2. Normalise both sides to forward slashes (`s.replace("\\", "/")`). Skipping this step will spuriously fail every Windows case if the implementation returns native separators.
3. Strip any trailing `/` from both sides before comparison.

## Verification

After generation, the runner should pass **every** case in `tests.yaml`. The current case counts are indicative — they change whenever a case is added — but as of this writing:

| Group             | Cases (indicative) |
| ----------------- | ------------------ |
| `which_strategy`  |   9                |
| `xdg`             |  24                |
| `apple`           |  18                |
| `windows`         |  18                |
| `single-folder`   |  12                |
| `errors`          |  10                |
| `project`         |   8+               |

Don't gate the implementation on these numbers — gate it on "every case passes". The table exists so you can spot wildly off-base parsing (e.g. only 2 cases loading from `xdg`).

Add your own cases in the `project:` block at the bottom of `tests.yaml` — those are the ones most likely to catch a generated implementation that's *almost* right for your real apps.

## Per-case runner steps

For each case:

1. Read `function`, `args`, `strategy`, `platform`, `env`.
2. Call the implementation's internal entry point with `platform` and `env` as explicit inputs (per "Public API" rule above). Do NOT mutate `os.environ` from the runner — that creates cross-test bleed when cases run in parallel and forces the runner to know which env vars to clear between cases. The internal entry point's signature should take exactly the per-case inputs.
3. `which_strategy` cases have no `platform` / `env` — pass `args` positionally only.
4. If `raises: true`, assert calling the function raises an error in your language's idiomatic style. Don't assert on the error type or message — only that it raised.
5. If `output: null`, assert the function returns the language's null/none value — no exception raised, no string returned.
6. Otherwise, call the function with the args and compare to `output` after substituting `$VAR` placeholders from `env`.
7. Normalise separators to `/` before comparison.

## Out of scope (don't let your agent invent these)

`wheredirs` deliberately stops at the seven functions listed above. If your generating agent starts adding `site_config_dir`, `user_documents_dir`, `user_downloads_dir`, `XDG_CONFIG_DIRS` handling, `%PROGRAMDATA%` lookups, or directory-creation helpers, push back — those belong in a heavier library like `platformdirs`, not here. See SPEC.md "Out of scope for v0.1".
