# wheredirs Specification v0.1.0

This document describes the behavior of the `wheredirs` library. Any implementation must conform to this specification and pass all test cases in `tests.yaml`.

## Overview

`wheredirs` resolves the on-disk paths where a given application should store its config, data, cache, state, logs, and runtime files. It supports four strategies — **XDG**, **Apple**, **Windows**, and **single-folder** — and can either pick one automatically from the current platform or use one the caller specifies.

The library is a pure path resolver: it does **not** create directories, read files, or touch the filesystem. Every function maps `(inputs) -> path string`.

## Design Principles

1. **Pure functions.** No I/O, no filesystem mutation, no global state. Inputs in, path string out.
2. **Deterministic.** Given the same inputs (args + platform + env vars), output is always the same.
3. **Forward slashes in the spec.** All expected paths in `tests.yaml` use `/` as the separator and `$HOME`-style placeholders, so cases are language- and host-independent. Implementations may render native separators (`\` on Windows) but must round-trip equivalently.
4. **Explicit beats ambient.** Anything that affects the result — platform, strategy, env vars — is an explicit input. Implementations may read `os.environ` and `sys.platform` as a convenience, but the spec is graded against the inputs declared in each test case.

## Functions

The library provides six directory-resolver functions plus one strategy-picker helper.

| Function       | Returns the directory for…       |
| -------------- | --------------------------------- |
| `config_dir`   | User configuration files         |
| `data_dir`     | User-specific persistent data    |
| `cache_dir`    | Non-essential cached data        |
| `state_dir`    | Persistent app state             |
| `logs_dir`     | Log files                        |
| `runtime_dir`  | Runtime files (sockets, pids, …) |
| `which_strategy` | The strategy auto-mode would pick on a given platform |

### Common signature

All six directory functions share the same parameter shape:

```
<kind>_dir(
    app,
    author=None,
    version=None,
    *,
    strategy=None,
    roaming=False,
) -> string
```

| Parameter  | Type     | Required | Default | Description                                                                 |
| ---------- | -------- | -------- | ------- | --------------------------------------------------------------------------- |
| `app`      | string   | Yes      | -       | Application name. Used as the leaf-or-last-but-one path component.          |
| `author`   | string?  | No       | `None`  | Organisation / publisher. Used **only** on the Windows strategy.            |
| `version`  | string?  | No       | `None`  | Version subdir appended after `app` (e.g. `MyApp/1.0`).                     |
| `strategy` | string?  | No       | `None`  | One of `"xdg"`, `"apple"`, `"windows"`, `"single-folder"`. `None` ⇒ auto.   |
| `roaming`  | boolean  | No       | `False` | Windows only. `True` switches the base from `%LOCALAPPDATA%` to `%APPDATA%`. Ignored by other strategies. |

## Strategies

### Auto (`strategy=None`)

The strategy is picked from the runtime platform:

| Platform     | Auto strategy |
| ------------ | ------------- |
| Linux / *BSD | `xdg`         |
| macOS        | `xdg`         |
| Windows      | `windows`     |

Auto mode never picks `apple` or `single-folder` — those are opt-in only. The `which_strategy(platform)` helper returns the strategy string auto mode would select for a given platform.

> **macOS note.** Auto-mode returns `xdg` on macOS, not `apple`. The native Apple layout is available by passing `strategy="apple"` explicitly. This matches `etcetera`'s default and diverges from `platformdirs`; the rationale is cross-platform consistency for headless and developer tooling, which is the dominant use of `wheredirs`.

### XDG (`strategy="xdg"`)

Follows the [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/latest/). Used on Linux/BSD and (per above) by default on macOS.

| Kind       | Path                                                       |
| ---------- | ---------------------------------------------------------- |
| config     | `${XDG_CONFIG_HOME:-$HOME/.config}/{app}[/{version}]`      |
| data       | `${XDG_DATA_HOME:-$HOME/.local/share}/{app}[/{version}]`   |
| cache      | `${XDG_CACHE_HOME:-$HOME/.cache}/{app}[/{version}]`        |
| state      | `${XDG_STATE_HOME:-$HOME/.local/state}/{app}[/{version}]`  |
| logs       | `<state_dir>/logs`                                         |
| runtime    | `${XDG_RUNTIME_DIR}/{app}[/{version}]` — see fallback rule |

**XDG env-var resolution rules:**

1. If the variable is unset or empty, use the documented default.
2. If the variable is set but **not** an absolute path (does not start with `/`), per the XDG spec it must be treated as unset.
3. Otherwise use the variable's value verbatim.

**Runtime fallback rule.** XDG defines `XDG_RUNTIME_DIR` as having *no* default — applications "should fall back to a replacement directory with similar capabilities and print a warning". For determinism, `wheredirs` falls back to **`cache_dir(...)`** when `XDG_RUNTIME_DIR` is unset or non-absolute. Implementations should emit a warning at the language's idiomatic warning channel but the return value is fully determined.

`author` is **ignored** under XDG.

### Apple (`strategy="apple"`)

Native macOS layout under `~/Library`.

| Kind       | Path                                                                |
| ---------- | ------------------------------------------------------------------- |
| config     | `$HOME/Library/Application Support/{app}[/{version}]`               |
| data       | `$HOME/Library/Application Support/{app}[/{version}]`               |
| cache      | `$HOME/Library/Caches/{app}[/{version}]`                            |
| state      | `$HOME/Library/Application Support/{app}[/{version}]`               |
| logs       | `$HOME/Library/Logs/{app}[/{version}]`                              |
| runtime    | `$HOME/Library/Caches/{app}[/{version}]` (aliased to cache)         |

`author` is **ignored** under Apple. `data` and `state` deliberately collapse to the same path as `config` — Apple has no XDG-style separation.

### Windows (`strategy="windows"`)

Uses `%APPDATA%` (roaming) or `%LOCALAPPDATA%` (local). The default is **local**; pass `roaming=True` to opt into the roaming base.

| Kind       | Base                                | Suffix              |
| ---------- | ----------------------------------- | ------------------- |
| config     | `%LOCALAPPDATA%` (or `%APPDATA%`)   | _(none)_            |
| data       | `%LOCALAPPDATA%` (or `%APPDATA%`)   | _(none)_            |
| cache      | `%LOCALAPPDATA%` (or `%APPDATA%`)   | `/Cache`            |
| state      | `%LOCALAPPDATA%` (or `%APPDATA%`)   | _(none)_            |
| logs       | `%LOCALAPPDATA%` (or `%APPDATA%`)   | `/Logs`             |
| runtime    | `%LOCALAPPDATA%` (or `%APPDATA%`)   | _(none)_            |

The full pattern is:

```
{base}/{author}/{app}[/{version}]{suffix}
```

If `author` is `None`, the `{author}` segment (and its separator) is omitted:

```
{base}/{app}[/{version}]{suffix}
```

**Env-var resolution.** If `%LOCALAPPDATA%` is unset or empty, fall back to `${USERPROFILE}/AppData/Local`. If `%APPDATA%` is unset or empty, fall back to `${USERPROFILE}/AppData/Roaming`. If `%USERPROFILE%` is also unset, fall back to `$HOME`.

### Single-folder (`strategy="single-folder"`)

The classic Unix single-dotfolder convention: one folder under `$HOME` holds everything.

| Kind       | Path                          |
| ---------- | ----------------------------- |
| _all six_  | `$HOME/.{app}[/{version}]`    |

`author` and `roaming` are **ignored**. `app` is lowercased and any whitespace is replaced with `-` before being prefixed with `.` — so `app="My App"` produces `~/.my-app`. (This is the only strategy that mangles `app`; the others use it verbatim.)

## App, author, version: normalisation rules

- `app` is used verbatim under XDG, Apple, and Windows. Under single-folder it is lowercased + `\s+` → `-`. Other characters are preserved (including `.`, `_`).
- `author` is verbatim where used (Windows).
- `version` is verbatim. Implementations should not parse or validate it.
- `None` (or the language's equivalent — `null`, `nil`, `undefined`, `Option::None`) means "omit this segment".

## `which_strategy(platform) -> string`

Helper for callers that want to inspect what auto-mode would do.

```
which_strategy("linux")   -> "xdg"
which_strategy("macos")   -> "xdg"
which_strategy("windows") -> "windows"
```

`platform` is a lowercase short name. Implementations should accept the conventional aliases of their language (e.g. `"darwin"` ≡ `"macos"`, `"win32"` ≡ `"windows"`).

## Error Handling

### Required errors

1. `strategy` set to anything other than `"xdg"`, `"apple"`, `"windows"`, `"single-folder"`, or `None`.
2. `app` is `None`, empty string, or contains a path separator (`/` or `\`).
3. `which_strategy` called with an unrecognised platform string.

The exception type is language-idiomatic (`ValueError` in Python, `Error` in Go, etc.). Tests assert *that* an error is raised, not its concrete type.

### Graceful handling

1. `author` / `version` set on strategies that ignore them: silently ignored.
2. `roaming=True` on non-Windows strategies: silently ignored.
3. Trailing slash on env vars (e.g. `XDG_CONFIG_HOME=/tmp/`): stripped before joining.

## Path output format

All paths returned by `wheredirs` use **forward slashes** in the spec and tests, with no trailing slash. Implementations on Windows may translate to backslashes at the API boundary, but must produce the same logical path. Test runners normalise to `/` before comparing.

## Test fixture format

Each case in `tests.yaml` declares the full input environment. The expected `output` is a path with `$HOME`-style placeholders matching the keys in `env`. The test runner substitutes those placeholders from `env` before comparing to the implementation's output.

```yaml
- name: "xdg-config-linux-default"
  function: config_dir
  args:
    app: "MyApp"
  strategy: null            # auto
  platform: "linux"
  env:
    HOME: "/home/alice"
  output: "$HOME/.config/MyApp"
```

For Windows cases, `env` may include `APPDATA`, `LOCALAPPDATA`, `USERPROFILE`. For XDG cases, any of `HOME`, `XDG_CONFIG_HOME`, `XDG_DATA_HOME`, `XDG_CACHE_HOME`, `XDG_STATE_HOME`, `XDG_RUNTIME_DIR`. Variables not declared in `env` are treated as unset.

## Out of scope for v0.1

- System-wide / site-wide directories (`/etc`, `XDG_CONFIG_DIRS`, `%PROGRAMDATA%`).
- Per-user vs per-machine selection on Windows beyond the roaming/local flag.
- Creating directories on disk.
- Finding existing files within those directories.
- Unicode normalisation of `app` / `author` / `version`.

## Changelog

- **v0.1.0** (2026-06-09): Initial specification.
