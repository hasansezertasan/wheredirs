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
| logs       | `<state_base>/{app}[/{version}]/logs`                      |
| runtime    | `${XDG_RUNTIME_DIR}/{app}[/{version}]` — see fallback rule |

Where `<state_base>` is `${XDG_STATE_HOME:-$HOME/.local/state}`. The `logs` segment is always the **last** path component, after `{app}` and any `{version}` subdir. So `logs_dir(app="MyApp", version="2.3.1")` resolves to `$HOME/.local/state/MyApp/2.3.1/logs`, never `$HOME/.local/state/MyApp/logs/2.3.1`.

**XDG env-var resolution rules:**

1. If the variable is unset or empty, use the documented default.
2. If the variable is set but **not** an absolute path (does not start with `/`), per the XDG spec it must be treated as unset.
3. Otherwise use the variable's value verbatim.

**Runtime fallback rule.** XDG defines `XDG_RUNTIME_DIR` as having *no* default — applications "should fall back to a replacement directory with similar capabilities and print a warning". `wheredirs` chooses **`cache_dir(...)`** as that fallback when `XDG_RUNTIME_DIR` is unset, empty, or non-absolute. The choice is deliberate: `cache_dir` is always resolvable from `$HOME`, agrees across implementations, and matches the "ephemeral, may disappear" semantics callers expect from runtime. `platformdirs` raises or returns a tempdir here, and `etcetera` has no runtime concept at all; `wheredirs` picks one answer so `tests.yaml` is portable. Implementations should emit a warning at the language's idiomatic warning channel; the return value is fully determined.

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

`author` and `roaming` are **ignored**. `app` is mangled in three steps, in this order, before being prefixed with `.`:

1. **Trim** leading and trailing whitespace.
2. **Collapse** any run of one-or-more whitespace characters (`\s+`, i.e. spaces, tabs, newlines) to a single `-`.
3. **Lowercase** the result.

Examples:

| Input               | Mangled    | Final path     |
| ------------------- | ---------- | -------------- |
| `"MyApp"`           | `myapp`    | `~/.myapp`     |
| `"My App"`          | `my-app`   | `~/.my-app`    |
| `"My  App"`         | `my-app`   | `~/.my-app`    |
| `"  My\tApp  "`     | `my-app`   | `~/.my-app`    |
| `"Legacy Tool"`     | `legacy-tool` | `~/.legacy-tool` |

This is the only strategy that mangles `app`; the others use it verbatim. Non-whitespace characters (including `.`, `_`, digits, and non-ASCII letters) are left untouched after lowercasing.

## App, author, version: normalisation rules

- `app` is used verbatim under XDG, Apple, and Windows. Under single-folder it is trimmed + `\s+` → `-` + lowercased (see Single-folder section).
- `author` is verbatim where used (Windows). It is **not** validated; an `author` containing `/` or `\` produces a multi-segment path. This is intentional — callers may want a sub-vendor under a parent vendor.
- `version` is verbatim. Implementations should not parse or validate it. An `version` containing `/` (e.g. `"1.0/beta"`) likewise produces a multi-segment path; this is intentional and treated as caller intent, not an error.
- `None` (or the language's equivalent — `null`, `nil`, `undefined`, `Option::None`) means "omit this segment".

### What each strategy ignores

| Argument  | XDG      | Apple    | Windows         | Single-folder |
| --------- | -------- | -------- | --------------- | ------------- |
| `app`     | verbatim | verbatim | verbatim        | mangled       |
| `author`  | ignored  | ignored  | used            | ignored       |
| `version` | used     | used     | used            | used          |
| `roaming` | ignored  | ignored  | switches base   | ignored       |

"Ignored" means silently accepted without changing the output. No warning is emitted.

## `which_strategy(platform) -> string`

Helper for callers that want to inspect what auto-mode would do.

```
which_strategy("linux")   -> "xdg"
which_strategy("macos")   -> "xdg"
which_strategy("windows") -> "windows"
```

`platform` is a lowercase short name. The full set of recognised inputs is fixed by the spec so that conformance is portable across implementations:

| Input                                                    | Returns      |
| -------------------------------------------------------- | ------------ |
| `"linux"`, `"freebsd"`, `"openbsd"`, `"netbsd"`, `"dragonfly"` | `"xdg"`      |
| `"macos"`, `"darwin"`                                    | `"xdg"`      |
| `"windows"`, `"win32"`                                   | `"windows"`  |
| anything else (including `""`, `None`)                   | **raises**   |

Implementations must not silently extend this set with their language's own aliases; if a caller passes `sys.platform` directly and it isn't in the table, the implementation should map it before calling (or accept the error).

## Error Handling

### Required errors

1. `strategy` set to anything other than `"xdg"`, `"apple"`, `"windows"`, `"single-folder"`, or `None`.
2. `app` is `None`, empty string, or contains a path separator (`/` or `\`).
3. `which_strategy` called with anything not in the recognised-platform table — including `None`, `""`, and unknown strings such as `"plan9"`.

The exception type is language-idiomatic (`ValueError` in Python, `Error` in Go, etc.). Tests assert *that* an error is raised, not its concrete type.

### Graceful handling

1. `author` / `version` set on strategies that ignore them: silently ignored.
2. `roaming=True` on non-Windows strategies: silently ignored.
3. Trailing slash on **any** path-valued env var read by the spec — `HOME`, `XDG_CONFIG_HOME`, `XDG_DATA_HOME`, `XDG_CACHE_HOME`, `XDG_STATE_HOME`, `XDG_RUNTIME_DIR`, `APPDATA`, `LOCALAPPDATA`, `USERPROFILE` — is stripped before joining. Multiple trailing slashes are collapsed.

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

## Test fixture format — placeholder substitution

The test runner substitutes `$VAR`-style placeholders in each case's `output` using the matching key in the case's `env` block. Substitution rules:

1. Only keys present in the case's `env` are substituted. Variables not in `env` (e.g. `$HOMEBREW_PREFIX`) are left as literal text.
2. Use **longest-match-first** ordering when one placeholder name is a prefix of another (e.g. substitute `$XDG_CONFIG_HOME` before `$HOME`). A naive `s/$HOME/.../` pass over the string is incorrect.
3. The substituted string is compared after normalising path separators to `/` on both sides.

## Changelog

- **v0.1.0** (2026-06-09): Initial specification.
