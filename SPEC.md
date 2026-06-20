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
    *,
    strategy=None,
) -> string | null
```

| Parameter  | Type     | Required | Default | Description                                                                 |
| ---------- | -------- | -------- | ------- | --------------------------------------------------------------------------- |
| `app`      | string   | Yes      | -       | Application name. Used as the leaf path component.                          |
| `author`   | string?  | No       | `None`  | Organisation / publisher. Used on Windows (path-join) and Apple (dot-join). |
| `strategy` | string?  | No       | `None`  | One of `"xdg"`, `"apple"`, `"windows"`, `"single-folder"`. `None` ⇒ auto.   |

> **No `roaming` flag.** Microsoft [deprecated roaming AppData as of Windows 11](https://learn.microsoft.com/en-us/windows/apps/design/app-settings/store-and-retrieve-app-data#roaming-data) and recommends Local app data for all new applications. `wheredirs` does not expose a `roaming` parameter — the Windows strategy always uses `%LOCALAPPDATA%`. This also avoids the long-standing footgun where a single `roaming` boolean could be set to `True` and accidentally cause cache or runtime files to replicate across machines. Callers with an existing Windows app that has a long history of using `%APPDATA%` (and who explicitly want to preserve those paths) should read `APPDATA` themselves rather than going through `wheredirs`.

> **No `version` parameter.** None of the platform guides — XDG, Apple FSPG, Windows Known Folder IDs, FHS — specify a per-version subdirectory convention. The `version` parameter is an `appdirs`/`platformdirs` Python-tradition affordance that `wheredirs` deliberately omits. Callers that genuinely want version-isolated paths can compose them in one line: `Path(config_dir("MyApp")) / "1.0"`. Keeping the API surface minimal means there is no asymmetry to document about "where does version go on each platform" (Apple wants it nested? Windows wants it before the `/Logs` suffix? XDG doesn't care?) — those questions disappear because the spec doesn't take a position on them.

## At-a-glance: strategy × function matrix

The table below summarises what each function returns under each strategy. **`null`** means the underlying platform has no concept for that directory kind — implementations return the language's null/none value (Python `None`, Rust `Option::None`, Go zero-value with `ok=false`, TypeScript `null`). Callers must check before use.

| Function       | XDG                                                                  | Apple                                       | Windows                                                | single-folder      |
| -------------- | -------------------------------------------------------------------- | ------------------------------------------- | ------------------------------------------------------ | ------------------ |
| `config_dir`   | `${XDG_CONFIG_HOME:-$HOME/.config}/{app}`                            | `$HOME/Library/Application Support/{seg}`   | `%LOCALAPPDATA%/{a/}{app}`                             | `$HOME/.{app}`     |
| `data_dir`     | `${XDG_DATA_HOME:-$HOME/.local/share}/{app}`                         | `$HOME/Library/Application Support/{seg}`   | `%LOCALAPPDATA%/{a/}{app}`                             | `$HOME/.{app}`     |
| `cache_dir`    | `${XDG_CACHE_HOME:-$HOME/.cache}/{app}`                              | `$HOME/Library/Caches/{seg}`                | `%LOCALAPPDATA%/{a/}{app}/Cache`                       | `$HOME/.{app}`     |
| `state_dir`    | `${XDG_STATE_HOME:-$HOME/.local/state}/{app}`                        | **`null`**                                  | **`null`**                                             | `$HOME/.{app}`     |
| `logs_dir`     | `<state_base>/{app}/logs` ⁽ʷᵉˣᵗ⁾                                      | `$HOME/Library/Logs/{seg}`                  | `%LOCALAPPDATA%/{a/}{app}/Logs` ⁽ʷᵉˣᵗ⁾                  | `$HOME/.{app}`     |
| `runtime_dir`  | `${XDG_RUNTIME_DIR}/{app}` ⁽ʷᵉˣᵗ-fallback⁾                            | **`null`**                                  | **`null`**                                             | `$HOME/.{app}`     |

Notation:
- `{seg}` on Apple = `{app}` when `author` is `None`/empty, or `{author}.{app}` when given (dot-join into a single segment, per Apple bundle-ID convention).
- `{a/}` on Windows = the literal string `{author}/` when `author` is given, or **the empty string** when `author` is `None`/empty — both the author segment **and** its trailing separator are omitted, so `config_dir(app="MyApp")` is `%LOCALAPPDATA%/MyApp`, not `%LOCALAPPDATA%//MyApp`. (Path-join into nested segments, per Microsoft `<Company>\<Product>` convention.)
- ⁽ʷᵉˣᵗ⁾ marks a `wheredirs` extension — the source platform does not define this; `wheredirs` picks a value to keep tests portable. ⁽ʷᵉˣᵗ-fallback⁾ on `runtime_dir` flags both the XDG-defined-but-unspecified fallback path (resolves via `cache_dir`; see XDG section) and the wheredirs extension status.
- `single-folder` returns the same path for all six kinds — it has no per-kind suffixes or aliasing rules.

**Why does Apple/Windows return `null` for `state_dir` and `runtime_dir`?** Apple's File System Programming Guide defines no "state" or "runtime" directories; macOS apps that need persistent state put it in `Application Support/` and treat ephemeral runtime files via `NSTemporaryDirectory()` or app-private paths. Windows has no Known Folder ID for either. Rather than have `wheredirs` invent paths the platforms don't acknowledge (and risk callers writing wrongly-permissioned sockets or roaming state across machines), the spec returns `null`. Callers who want a "best-effort" path on every platform can fall back explicitly: `state_dir(...) ?? data_dir(...)`. This is consistent with `etcetera`'s approach and surfaces the platform asymmetry rather than papering over it.

## Strategies

### Auto (`strategy=None`)

The strategy is picked from the runtime platform — auto-mode resolves to whatever is native:

| Platform     | Auto strategy |
| ------------ | ------------- |
| Linux / *BSD | `xdg`         |
| macOS        | `apple`       |
| Windows      | `windows`     |

Auto mode never picks `single-folder` — that's a *layout* preference, not a platform default, and is opt-in only. The `which_strategy(platform)` helper returns the strategy string auto mode would select for a given platform.

**Any explicit strategy is legal on any platform.** `strategy="xdg"` on macOS returns XDG-layout paths under `$HOME/.config`, `$HOME/.cache`, etc.; `strategy="windows"` on Linux returns Windows-layout paths joined from `$LOCALAPPDATA` (or its `$USERPROFILE`/`$HOME` fallback). The strategy fully determines the layout — `platform` is only consulted by auto-mode. This is deliberate so cross-compilation, test fixtures, and "I want one layout everywhere" use cases all work uniformly.

> **macOS note.** Auto-mode on macOS returns `apple`, matching the native platform contract (`~/Library/Application Support/...` is what Time Machine, Migration Assistant, MDM scanners, and Finder expect). CLI tools or cross-platform headless services that prefer a single layout everywhere should pass `strategy="xdg"` explicitly — that gets `~/.config/MyApp` on macOS too, matching the Linux path. This makes wheredirs align with `platformdirs`' default and diverges from `etcetera`'s explicit-only design; the rationale is that `auto` should mean "auto-detect the native convention," not "auto-detect and apply a cross-platform consistency opinion." The escape hatch is symmetric (one parameter either way), but only the native-by-default direction lets the library do its job for GUI apps without requiring callers to write platform-detection logic.

### XDG (`strategy="xdg"`)

Follows the [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/latest/). Used on Linux/BSD by default; available on any platform via explicit `strategy="xdg"`.

The directory layout, env-var names, default paths, and the "non-absolute is unset" rule below all come directly from the XDG spec. Several of the rules in this section go beyond what XDG specifies — they're flagged as **`wheredirs` extensions** so implementers know which lines are derived from XDG and which are wheredirs' own opinion.

| Kind       | Path                                                       |
| ---------- | ---------------------------------------------------------- |
| config     | `${XDG_CONFIG_HOME:-$HOME/.config}/{app}`      |
| data       | `${XDG_DATA_HOME:-$HOME/.local/share}/{app}`   |
| cache      | `${XDG_CACHE_HOME:-$HOME/.cache}/{app}`        |
| state      | `${XDG_STATE_HOME:-$HOME/.local/state}/{app}`  |
| logs       | `<state_base>/{app}/logs`                      |
| runtime    | `${XDG_RUNTIME_DIR}/{app}` †                    |

† When `XDG_RUNTIME_DIR` is unset, empty, or non-absolute, the path falls back to `cache_dir(...)` — see the runtime-fallback rule below.

Where `<state_base>` is `${XDG_STATE_HOME:-$HOME/.local/state}`. The `logs` segment is always the **last** path component, after `{app}`. So `logs_dir(app="MyApp")` resolves to `$HOME/.local/state/MyApp/logs`.

> **wheredirs extension — XDG logs.** The XDG Base Directory Specification does not define a directory for log files. `wheredirs` places logs under the state directory (matching the systemd/journalctl convention of treating logs as machine-state) with a `/logs` suffix to keep them visually distinct from arbitrary state files. Implementations following XDG strictly would have to invent this themselves; `wheredirs` picks one answer so the path is portable.

**XDG env-var resolution rules** (apply to `XDG_CONFIG_HOME`, `XDG_DATA_HOME`, `XDG_CACHE_HOME`, `XDG_STATE_HOME`, `XDG_RUNTIME_DIR`):

1. If the variable is unset or the empty string, use the documented default. Values containing only whitespace are **not** treated as empty — they are passed through to rule 2 and almost certainly rejected by the absoluteness check.
2. If the variable is set but **not** an absolute path (does not start with `/`), per the XDG spec it must be treated as unset.
3. Otherwise use the variable's value verbatim.

**`HOME` is exempt from the absoluteness rule.** `HOME` is the host-provided base path, not an XDG variable. If `HOME` is unset or empty under the XDG strategy, the implementation **raises** — there is no fallback chain on Linux/BSD analogous to the Windows `LOCALAPPDATA → USERPROFILE → HOME` chain. A non-absolute `HOME` is used verbatim (which will usually produce a relative output path); validating `HOME` is the host's job, not `wheredirs`'.

**wheredirs extension — runtime fallback.** XDG defines `XDG_RUNTIME_DIR` as having *no* default — applications "should fall back to a replacement directory with similar capabilities and print a warning". `wheredirs` chooses **`cache_dir(app, author, strategy="xdg")`** as that fallback when `XDG_RUNTIME_DIR` is unset, empty, or non-absolute. The fallback re-runs the full `cache_dir` resolution — `XDG_CACHE_HOME` validation (rules 1–3 above) and trailing-slash stripping all apply. Implementations should call the same internal resolver they use for `cache_dir`, not inline the path. The choice is deliberate: `cache_dir` is always resolvable from `$HOME`, agrees across implementations, and matches the "ephemeral, may disappear" semantics callers expect from runtime. `platformdirs` raises or returns a tempdir here, and `etcetera` has no runtime concept at all; `wheredirs` picks one answer so `tests.yaml` is portable. Implementations should emit a warning at the language's idiomatic warning channel; the return value is fully determined.

> **⚠ Security note — runtime fallback.** The fallback path under `cache_dir` does **NOT** carry the guarantees a real `XDG_RUNTIME_DIR` does: not guaranteed `0700`, not guaranteed user-owned, not cleared on logout, and persists across reboots on most systems. Callers writing unix domain sockets, lock files, pid files, or any primitive that relies on those guarantees MUST verify or enforce the permissions themselves (e.g. `mkdir(..., 0o700)` + `chown` + stale-file removal) before use, OR detect that the fallback is in effect and refuse to operate. Treat `runtime_dir`'s output as "ephemeral *location*", not as "ephemeral *with isolation*", whenever `XDG_RUNTIME_DIR` was unset.

`author` is **ignored** under XDG.

### Apple (`strategy="apple"`)

Follows Apple's [File System Programming Guide](https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/FileSystemProgrammingGuide/MacOSXDirectories/MacOSXDirectories.html) — specifically the standard directories under `~/Library`. Used on macOS/darwin by default; available on any platform via explicit `strategy="apple"`. The full pattern is:

```
$HOME/Library/{base}/{author-app}
```

Where `{base}` varies by kind, and `{author-app}` is a **single path segment** formed by joining `author` and `app` with a dot. When `author` is `None`, the segment is just `app`. Concretely:

| Kind       | Base                       | With author                                                  | Without author                                |
| ---------- | -------------------------- | ------------------------------------------------------------ | --------------------------------------------- |
| config     | `Application Support`      | `$HOME/Library/Application Support/{author}.{app}`           | `$HOME/Library/Application Support/{app}`     |
| data       | `Application Support`      | `$HOME/Library/Application Support/{author}.{app}`           | `$HOME/Library/Application Support/{app}`     |
| cache      | `Caches`                   | `$HOME/Library/Caches/{author}.{app}`                        | `$HOME/Library/Caches/{app}`                  |
| **state**  | **(no Apple concept)**     | **returns `null`**                                           | **returns `null`**                            |
| logs       | `Logs`                     | `$HOME/Library/Logs/{author}.{app}`                          | `$HOME/Library/Logs/{app}`                    |
| **runtime**| **(no Apple concept)**     | **returns `null`**                                           | **returns `null`**                            |

`data` deliberately collapses to the same path as `config` — Apple has no XDG-style separation between them.

> **Why dot-join (one segment), not path-join (nested)?** Apple's File System Programming Guide is explicit (Table A-1): "all of these items should be put in a subdirectory whose name matches the **bundle identifier** of the app. For example, if your app is named MyApp and has the bundle identifier `com.example.MyApp`, you would put your app's user-specific data files and resources in the `~/Library/Application Support/com.example.MyApp/` directory." The recommended layout is **one directory whose name is the bundle ID**, not nested vendor/app directories. `wheredirs` honours this by joining `author` and `app` with `.` into a single segment, matching the bundle-ID convention. This deliberately differs from the Windows strategy, where `author` and `app` are separate path components — each strategy follows its own platform's convention rather than imposing a uniform join rule.

> **Recommendation — reverse-DNS naming.** Apple recommends reverse-DNS bundle identifiers (`com.acme.myapp`) to avoid collisions with apps that have similar display names. `wheredirs` does not normalize or validate `author` and `app` — they are joined verbatim with a dot. Callers that want strict bundle-ID conformance should pre-compose accordingly:
>
> - `author="com.acme", app="MyApp"` → `$HOME/Library/Application Support/com.acme.MyApp/` ← reverse-DNS, idiomatic Apple
> - `author="Acme", app="MyApp"` → `$HOME/Library/Application Support/Acme.MyApp/` ← still a single segment, functional but not strictly reverse-DNS
> - `app="MyApp"` (no author) → `$HOME/Library/Application Support/MyApp/` ← matches what platformdirs produces; less collision-resistant but acceptable for tools with unique names
>
> Unlike the Rust `etcetera` library, `wheredirs` does not split `top_level_domain` from `author` or apply lowercase/space-to-dash normalization automatically. The caller composes the dotted prefix themselves. This keeps the API surface smaller; the trade-off is that wheredirs trusts the caller not to pass syntactically-malformed identifiers.

> **No Apple concept — `state_dir` and `runtime_dir` return `null`.** Apple's File System Programming Guide does not define separate directories for app-state or runtime files. macOS apps that need persistent state put it in `Application Support/` and treat ephemeral runtime files via `NSTemporaryDirectory()` or app-private paths under `Caches/`. Rather than alias `state_dir` to `Application Support/` and `runtime_dir` to `Caches/` (creating two API surfaces for the same on-disk path, and risking callers writing socket files into a non-`0700` cache directory), `wheredirs` returns `null` for these on Apple. Callers who want a best-effort path should fall back explicitly:
>
> - "Use data dir if no state dir": `state_dir(...) ?? data_dir(...)`
> - "Use cache dir if no runtime dir, with the security caveat in mind": `runtime_dir(...) ?? cache_dir(...)`
>
> This makes the platform asymmetry visible to the caller instead of papering over it.

### Windows (`strategy="windows"`)

Follows Microsoft's [Known Folder IDs](https://learn.microsoft.com/en-us/windows/win32/shell/knownfolderid) — specifically `FOLDERID_LocalAppData` (`%LOCALAPPDATA%`). Used on Windows/win32 by default; available on any platform via explicit `strategy="windows"`.

The `<Company>\<Product>` author/app layout matches Microsoft's MSDN guidance for organizing files under AppData.

> **Why only `%LOCALAPPDATA%`?** Microsoft [deprecated roaming AppData as of Windows 11](https://learn.microsoft.com/en-us/windows/apps/design/app-settings/store-and-retrieve-app-data#roaming-data) and recommends Local for all new applications. Even on Windows 10, Microsoft already recommended Local over Roaming because "RoamingSettings data may not persist through Microsoft Store app updates." `wheredirs` therefore does not expose `%APPDATA%` at all — the spec is calibrated to Microsoft's current guidance, not its deprecated guidance. Callers porting legacy apps that genuinely need the historic roaming paths should read `%APPDATA%` themselves; the gain in API simplicity here outweighs the marginal loss in flexibility.

| Kind       | Base               | Suffix              |
| ---------- | ------------------ | ------------------- |
| config     | `%LOCALAPPDATA%`   | _(none)_            |
| data       | `%LOCALAPPDATA%`   | _(none)_            |
| cache      | `%LOCALAPPDATA%`   | `/Cache`            |
| **state**  | **(no Known Folder)** | **returns `null`**  |
| logs       | `%LOCALAPPDATA%`   | `/Logs`             |
| **runtime**| **(no Known Folder)** | **returns `null`**  |

The full pattern is:

```
%LOCALAPPDATA%/{author}/{app}{suffix}
```

If `author` is `None`, the `{author}` segment (and its separator) is omitted:

```
%LOCALAPPDATA%/{app}{suffix}
```

**Env-var resolution.** If `%LOCALAPPDATA%` is unset or the empty string, fall back to `${USERPROFILE}/AppData/Local`. If `%USERPROFILE%` is also unset or the empty string, fall back to `$HOME/AppData/Local`. If `$HOME` is also unset or the empty string, the implementation **raises** — there is no further fallback. The synthesized `/AppData/Local` segment is always appended with a single forward slash regardless of whether the base ended in `\`, `/`, or nothing (trailing-slash stripping per §Graceful handling fires first).

Unlike the XDG rules, Windows env vars are **not** checked for absoluteness. Any non-empty value is used verbatim. Windows paths legitimately take many shapes — drive-letter (`C:\...`), UNC (`\\server\share\...`), `\\?\` namespaced, mapped drives — and rejecting "non-absolute" forms would reject valid input. The only fallback trigger is unset-or-empty.

**Whitespace-only is not empty.** A value of `"   "` (spaces, tabs) is non-empty and is used verbatim — the fallback chain does not fire. This matches Win32's `GetEnvironmentVariable` behavior and avoids implementations diverging on whether to call `.strip()` before the empty check.

> **wheredirs extensions — `/Cache` and `/Logs` suffixes.** The Known Folder IDs catalog has no dedicated entries for per-app cache or log subdirectories under AppData (only `FOLDERID_LocalAppDataLow` for low-integrity processes, which `wheredirs` does not expose). The `/Cache` and `/Logs` suffixes are conventional — used by Microsoft and many Windows apps but not formally standardised. `wheredirs` picks them to keep cache and logs visually distinct from arbitrary state within the app folder.

> **No Windows concept — `state_dir` and `runtime_dir` return `null`.** Windows has no Known Folder for app state (apps that need it conventionally use subdirectories of `%LOCALAPPDATA%` chosen by the app itself) and no Known Folder for runtime files (apps use `%TEMP%`/`GetTempPath()` or app-private paths). Rather than alias `state_dir` to `data_dir` and `runtime_dir` to `%TEMP%/<app>` (which would surface as a path even though no Windows convention exists), `wheredirs` returns `null` for these on Windows. Callers wanting a best-effort path should fall back: `state_dir(...) ?? data_dir(...)` and similarly for runtime. This matches the Apple behavior and is consistent with the `etcetera` reference library.

### Single-folder (`strategy="single-folder"`)

The classic Unix single-dotfolder convention: one folder under `$HOME` holds everything.

> **No formal specification exists for this convention.** The `~/.appname/` dotfile pattern is unwritten Unix tradition predating XDG by decades — it is *not* covered by the [Filesystem Hierarchy Standard](https://www.pathname.com/fhs/), which formalises *system* directories (`/usr`, `/var`, `/etc`) but is silent on user-home layout. The closest things to a written reference are community discussions ([example: "File and directory naming conventions" on Unix Stack Exchange](https://unix.stackexchange.com/questions/15230/file-and-directory-naming-conventions)) and the XDG spec's own rationale section, which describes the dotfile-proliferation problem XDG was created to fix. `wheredirs` picks the pattern (lowercase, dash-collapse, dotted) used by long-running tools like `git`, `ssh`, and `vim` for their own private dot-directories.

| Kind       | Path                          |
| ---------- | ----------------------------- |
| _all six_  | `$HOME/.{app}`    |

`author` is **ignored**. `app` is mangled in three steps, in this order, before being prefixed with `.`:

1. **Trim** leading and trailing whitespace.
2. **Collapse** any run of one-or-more whitespace characters to a single `-`.
3. **Lowercase** the result.

**"Whitespace" means ASCII whitespace only** — the characters in the set `[ \t\n\r\f\v]` (U+0020, U+0009, U+000A, U+000D, U+000C, U+000B). Unicode whitespace (NBSP U+00A0, ideographic space U+3000, etc.) is **not** trimmed or collapsed; it passes through to the lowercasing step verbatim. This is pinned because regex `\s` semantics differ across languages — Python and JavaScript's `\s` defaults to Unicode-aware, Go and Rust default to ASCII, .NET varies by flag — and a uniform answer is needed for `tests.yaml` to be portable. Implementations using `\s` MUST configure it to ASCII-only or use the explicit character class.

Examples:

| Input               | Mangled    | Final path     |
| ------------------- | ---------- | -------------- |
| `"MyApp"`           | `myapp`    | `~/.myapp`     |
| `"My App"`          | `my-app`   | `~/.my-app`    |
| `"My  App"`         | `my-app`   | `~/.my-app`    |
| `"  My\tApp  "`     | `my-app`   | `~/.my-app`    |
| `"Legacy Tool"`     | `legacy-tool` | `~/.legacy-tool` |

This is the only strategy that mangles `app`; the others use it verbatim. Non-whitespace characters (including `.`, `_`, digits, and non-ASCII letters) are left untouched after lowercasing. Lowercasing is **ASCII-only**: characters outside `A-Z` are not folded (no Unicode case-folding, no locale-sensitive folding such as Turkish dotted/dotless I). This is intentional and consistent with the "Unicode normalisation out of scope" rule (see "Out of scope for v0.1").

**Why a peer strategy, not a layout axis?** `single-folder` is enumerated alongside the platform strategies for v0.1 because it's a complete, self-contained answer to "where does my app's stuff go" — it doesn't need a platform input. A future version may split layout (split-by-kind vs single-folder) from platform (xdg/apple/windows) into orthogonal axes; that refactor is deferred until a real use case appears for, say, "Apple layout collapsed under one folder".

## App and author: normalisation rules

- `app` is used verbatim under XDG, Apple, and Windows. Under single-folder it is trimmed + ASCII-whitespace-collapsed to `-` + lowercased (see Single-folder section).
- `author` is verbatim where used. On Windows it is **not** validated; an `author` containing `/` or `\` produces a multi-segment path. On Apple it is dot-joined with `app` verbatim — embedded dots, slashes, and backslashes are **not** rejected or escaped. An `author="a/b"` on Apple yields the segment `a/b.{app}`, which is two path components, not one — this is intentional, mirroring the Windows multi-segment behavior, and lets callers compose reverse-DNS sub-vendors without `wheredirs` second-guessing the shape. Callers who require a single bundle-ID segment must pre-validate.
- For `author`: `None` (or the language's equivalent — `null`, `nil`, `undefined`, `Option::None`) means "omit this segment". Omitting the parameter entirely and passing `None` explicitly are equivalent. The empty string `""` is **also** treated as "omit this segment" — there is no Apple/Windows path where an empty author segment would be useful, and the alternative (`{author}.{app}` → `.{app}` or `{author}/{app}` → `/{app}`) produces malformed paths. (`app` itself is required; see Error Handling.)

### What each strategy ignores

| Argument  | XDG      | Apple    | Windows         | Single-folder |
| --------- | -------- | -------- | --------------- | ------------- |
| `app`     | verbatim | verbatim | verbatim        | mangled       |
| `author`  | ignored  | dot-join | path-join       | ignored       |

"Ignored" means silently accepted without changing the output. No warning is emitted.

> **`author` asymmetry.** `author` is consumed by Windows and Apple but with **different join rules**, and ignored on XDG and single-folder:
>
> | Strategy        | What `author="Acme", app="MyApp"` produces                  | Join rule                                |
> | --------------- | ----------------------------------------------------------- | ---------------------------------------- |
> | XDG             | `…/MyApp`                                                   | `author` ignored (XDG has no vendor concept) |
> | Apple           | `…/Acme.MyApp`                                              | **dot-join into one segment** (matches Apple bundle-ID convention) |
> | Windows         | `…/Acme/MyApp`                                              | **path-join into nested segments** (matches Microsoft `<Company>\<Product>` convention) |
> | single-folder   | `…/.myapp`                                                  | `author` ignored                         |
>
> Each strategy uses the convention of the platform it represents — not a uniform "vendor segment" rule. A caller who develops on Linux with `author="Acme"` will see no change to the path; on macOS they will see `Acme.MyApp/`; on Windows they will see `Acme\MyApp\`. This is intentional. Downstream impls SHOULD document the per-platform behavior in their README. Callers who want one identical path across all platforms should either embed everything into `app` (e.g. `app="Acme-MyApp"`) or use `single-folder`.

## `which_strategy(platform) -> string`

Helper for callers that want to inspect what auto-mode would do.

```
which_strategy("linux")   -> "xdg"
which_strategy("macos")   -> "apple"
which_strategy("windows") -> "windows"
```

`platform` is a lowercase short name. The full set of recognised inputs is fixed by the spec so that conformance is portable across implementations:

| Input                                                    | Returns      |
| -------------------------------------------------------- | ------------ |
| `"linux"`, `"freebsd"`, `"openbsd"`, `"netbsd"`, `"dragonfly"` | `"xdg"`      |
| `"macos"`, `"darwin"`                                    | `"apple"`    |
| `"windows"`, `"win32"`                                   | `"windows"`  |
| anything else (including `""`, `None`, mixed case like `"Linux"`) | **raises**   |

Matching is **case-sensitive**: `"linux"` is accepted, `"Linux"` raises. Implementations must not silently extend this set with their language's own aliases; if a caller passes `sys.platform` directly and it isn't in the table, the implementation should map it before calling (or accept the error).

See §Test fixture format below for the canonical statement of how `which_strategy` cases encode `args` in `tests.yaml` (list form, one element).

## Error Handling

### Required errors

1. `strategy` set to anything other than `"xdg"`, `"apple"`, `"windows"`, `"single-folder"`, or `None`. This explicitly includes the empty string `""`. Strategy names are case-sensitive — `"XDG"`, `"Apple"`, `"Windows"` all raise.
2. `app` fails any of the following checks. Validation happens *before* any env resolution — `app` errors are raised regardless of whether the platform's required env vars are set.
   - `app` is `None` (or the language equivalent).
   - `app` is the empty string `""`.
   - `app` contains a path separator: `/` or `\`.
   - `app` contains an embedded NUL byte (`\0`).
   - `app` consists entirely of ASCII whitespace (would mangle to the empty string under `single-folder`, producing the malformed path `~/.`).
   - `app` is `.` or `..` (would produce `~/..` etc. — outright path traversal).
   - `app` starts or ends with `.` (e.g. `".emacs"`, `"MyApp."`). Rationale: under `single-folder` a leading dot doubles up (`.emacs` → `~/..emacs`, which reads as a traversal token even if filesystems don't interpret it as one); under Windows a trailing dot is silently stripped by Win32 file APIs, so `MyApp.` and `MyApp` collide. Embedded dots are legal (`com.acme.MyApp`, `vim.tiny` pass) — only the leading/trailing positions are rejected.
3. `which_strategy` called with anything not in the recognised-platform table — including `None`, `""`, mixed-case variants such as `"Linux"` or `"MACOS"`, and unknown strings such as `"plan9"`. Recognition is case-sensitive on the strings in the table below.

The exception type is language-idiomatic (`ValueError` in Python, `Error` in Go, etc.). Tests assert *that* an error is raised, not its concrete type.

### Graceful handling

1. `author` set on strategies that ignore it (XDG, single-folder): silently ignored.
2. Trailing slash on **any** path-valued env var read by the spec — `HOME`, `XDG_CONFIG_HOME`, `XDG_DATA_HOME`, `XDG_CACHE_HOME`, `XDG_STATE_HOME`, `XDG_RUNTIME_DIR`, `LOCALAPPDATA`, `USERPROFILE` — is stripped before joining. Multiple trailing slashes are collapsed.

## Path output format

All paths returned by `wheredirs` use **forward slashes** in the spec and tests, with no trailing slash. Implementations on Windows may translate to backslashes at the API boundary, but must produce the same logical path. Test runners normalise to `/` before comparing.

## Test fixture format

Each case in `tests.yaml` declares the full input environment. The expected `output` is one of:

- A path string with `$HOME`-style placeholders matching keys in `env` — the test runner substitutes those placeholders from `env` before comparing to the implementation's output.
- `null` — the implementation must return the language's null/none value (Python `None`, Rust `Option::None`, TypeScript `null`, Go: whatever the signature exposes — see INSTALL.md's per-language notes). Used for functions that have no concept on a given strategy (e.g. `state_dir`/`runtime_dir` on Apple and Windows).
- Omitted, with `raises: true` instead — the implementation must raise/return an error.

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

For Windows cases, `env` may include `LOCALAPPDATA`, `USERPROFILE`, `HOME`. For XDG cases, any of `HOME`, `XDG_CONFIG_HOME`, `XDG_DATA_HOME`, `XDG_CACHE_HOME`, `XDG_STATE_HOME`, `XDG_RUNTIME_DIR`. Variables not declared in `env` are treated as unset.

The fixture file also has a top-level `version:` key (e.g. `version: "0.1.0"`) that pins the spec version the cases conform to. Runners may use it as a sanity-check (refuse to run if the implementation targets an incompatible spec version) but the field has no per-case semantics and is not required for grading.

**`args` shape per function (canonical):** `which_strategy` cases use `args` as a **list** (one element): `args: ["linux"]`. The six dir resolvers use `args` as a **map**: `args: { app: "MyApp", author: "Acme" }`. This is the only shape distinction in the fixture schema. INSTALL.md restates this for the generating agent; SPEC.md is the source of truth.

## Out of scope for v0.1

- System-wide / site-wide directories (`/etc`, `XDG_CONFIG_DIRS`, `%PROGRAMDATA%`, FHS paths like `/var/lib/<app>`).
- Per-user vs per-machine selection on Windows beyond the Local AppData base (i.e. no `%PROGRAMDATA%`, no roaming).
- Windows `FOLDERID_LocalAppDataLow` (low-integrity processes).
- Creating directories on disk.
- Finding existing files within those directories.
- Unicode normalisation of `app` / `author`.

### Deferred candidate strategies

These are real conventions covered by formal specifications that `wheredirs` deliberately omits from v0.1 — listed here so the door is documented as open:

- **`system` / FHS** — system-daemon paths per the [Filesystem Hierarchy Standard](https://www.pathname.com/fhs/): `/etc/<app>`, `/var/lib/<app>`, `/var/cache/<app>`, `/var/log/<app>`, `/run/<app>`. This is the most obvious next strategy: it's formally specified, widely deployed, and complements the user-scoped four. A future version may add it as `strategy="system"` returning these paths regardless of platform (FHS applies on Linux and most Unixes; Windows daemons use a different model entirely and would not be covered).
- **`portable`** — everything under a caller-supplied base directory (typical for AppImage / portable apps). Achievable today by post-processing wheredirs' output; not worth a peer strategy unless a real use case appears.

## References

The behavioral rules above are derived from these specifications and conventions. Where `wheredirs` extends or diverges from a source, the extension is called out inline in the relevant section.

- **XDG Base Directory Specification** — <https://specifications.freedesktop.org/basedir-spec/latest/>
  Source of: all `XDG_*` env var names, default paths, the "non-absolute is unset" rule, and the `XDG_RUNTIME_DIR` warning-on-fallback guidance.
- **Apple's File System Programming Guide** — <https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/FileSystemProgrammingGuide/FileSystemOverview/FileSystemOverview.html>
  Source of: the `~/Library/Application Support/`, `Caches/`, `Logs/` directory layout and the reverse-DNS naming recommendation.
- **Windows Known Folder IDs** — <https://learn.microsoft.com/en-us/windows/win32/shell/knownfolderid>
  Source of: `FOLDERID_LocalAppData` and the `%LOCALAPPDATA%` env-var mapping. `FOLDERID_RoamingAppData` / `%APPDATA%` is documented here but not used by `wheredirs` (see "No roaming flag" note above).
- **Filesystem Hierarchy Standard (FHS)** — <https://www.pathname.com/fhs/>
  Referenced for the deferred `system` strategy. **Not** a reference for the `single-folder` strategy — FHS does not cover user-home layout.
- **Unix dotfile convention (informal)** — example: ["File and directory naming conventions" on Unix Stack Exchange](https://unix.stackexchange.com/questions/15230/file-and-directory-naming-conventions). No authoritative spec exists for the `~/.app/` pattern; `wheredirs` codifies the common form (lowercase, dash-collapse, leading dot).

## Test fixture format — placeholder substitution

The test runner substitutes `$VAR`-style placeholders in each case's `output` using the matching key in the case's `env` block. Substitution rules:

1. Only keys present in the case's `env` are substituted. Variables not in `env` (e.g. `$HOMEBREW_PREFIX`) are left as literal text.
2. Use **longest-match-first** ordering when one placeholder name is a prefix of another (e.g. substitute `$XDG_CONFIG_HOME` before `$HOME`). A naive `s/$HOME/.../` pass over the string is incorrect.
3. After substitution, on both expected and actual: normalise `\` → `/`, then **collapse runs of `/` to a single `/`**, then strip any trailing `/`. The collapse step matters when an env value has a trailing slash (e.g. `HOME="/home/alice/"` substituted into `"$HOME/.config/MyApp"` produces `/home/alice//.config/MyApp` — collapsing to `/home/alice/.config/MyApp` is what makes trailing-slash test cases pass without each case having to pre-strip its env values).

## Changelog

- **v0.1.0** (2026-06-09): Initial specification.
