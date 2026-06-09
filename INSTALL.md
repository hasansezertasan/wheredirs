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
   - macOS auto-mode returns the XDG layout, NOT the Apple layout.
     This is deliberate and differs from platformdirs; do not
     "correct" it. Apple paths are only returned when the caller
     passes strategy="apple" explicitly.
   - Substitute env-var placeholders ($HOME, $XDG_*, $APPDATA,
     $LOCALAPPDATA, $USERPROFILE) in expected outputs from each
     case's `env` block before comparing. Substitute the
     longest-named placeholder first (e.g. $XDG_CONFIG_HOME
     before $HOME); a naive replace will corrupt other names.
   - Variables absent from `env` are unset.
   - Cases with `raises: true` must raise an idiomatic exception
     in your language.
   - which_strategy accepts only the platform names enumerated
     in SPEC.md. Empty string, null, and unknown values raise.
5. Run the test suite until every case passes.
```

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
| `xdg`             |  25                |
| `apple`           |  10                |
| `windows`         |  20                |
| `single-folder`   |  13                |
| `errors`          |   9                |
| `project`         |   7+               |

Don't gate the implementation on these numbers — gate it on "every case passes". The table exists so you can spot wildly off-base parsing (e.g. only 2 cases loading from `xdg`).

Add your own cases in the `project:` block at the bottom of `tests.yaml` — those are the ones most likely to catch a generated implementation that's *almost* right for your real apps.

## Per-case runner steps

For each case:

1. Read `function`, `args`, `strategy`, `roaming`, `platform`, `env`.
2. Set up the implementation as if `platform` were the current OS and `env` were `os.environ`. (Implementations may take these as explicit inputs to make testing trivial — that's encouraged.)
3. If `raises: true`, assert calling the function raises.
4. Otherwise, call the function with the args and compare to `output` after substituting `$VAR` placeholders from `env`.
5. Normalise separators to `/` before comparison.

## Out of scope (don't let your agent invent these)

`wheredirs` deliberately stops at the seven functions listed above. If your generating agent starts adding `site_config_dir`, `user_documents_dir`, `user_downloads_dir`, `XDG_CONFIG_DIRS` handling, `%PROGRAMDATA%` lookups, or directory-creation helpers, push back — those belong in a heavier library like `platformdirs`, not here. See SPEC.md "Out of scope for v0.1".
