# Generating wheredirs

`wheredirs` is a **ghost library** — it ships a specification and test cases instead of executable code. To use it, give the files to an AI coding assistant and tell it what language you want.

## Quick start

Hand your coding assistant this prompt:

```
Implement the wheredirs library in [LANGUAGE].

1. Read SPEC.md for the complete behavioral specification.
2. Parse tests.yaml and generate a test file. Each top-level key
   (which_strategy, xdg, apple, windows, single-folder, errors,
   project) is a group of cases.
3. Implement these functions:
     config_dir, data_dir, cache_dir, state_dir, logs_dir,
     runtime_dir, which_strategy
   matching the common signature in SPEC.md.
4. Implementation rules:
   - Pure functions; no filesystem I/O, no directory creation.
   - Forward slashes in returned paths (the test runner normalises).
   - Substitute env-var placeholders ($HOME, $XDG_*, $APPDATA,
     $LOCALAPPDATA, $USERPROFILE) in expected outputs from each
     case's `env` block before comparing.
   - Variables absent from `env` are unset.
   - Cases with `raises: true` must raise an idiomatic exception
     in your language.
5. Run the test suite until every case passes.
```

## Verification

After generation, the runner should pass every case in `tests.yaml`. Expect roughly:

| Group             | Cases |
| ----------------- | ----- |
| `which_strategy`  |   6   |
| `xdg`             |  21   |
| `apple`           |  10   |
| `windows`         |  16   |
| `single-folder`   |  10   |
| `errors`          |   6   |
| `project`         |   7   |

Add your own cases in the `project:` block at the bottom of `tests.yaml` — those are the ones most likely to catch a generated implementation that's *almost* right for your real apps.

## Test runner contract

A correct runner does the following for each case:

1. Read `function`, `args`, `strategy`, `roaming`, `platform`, `env`.
2. Set up the implementation as if `platform` were the current OS and `env` were `os.environ`. (Implementations may take these as explicit inputs to make testing trivial — that's encouraged.)
3. If `raises: true`, assert calling the function raises.
4. Otherwise, call the function with the args and compare to `output` after substituting `$VAR` placeholders from `env`.
5. Normalise separators to `/` before comparison.
