# wheredirs

Show where standard application directories (config, cache, data, state, logs, runtime) resolve to — for the current platform, or for any platform and convention you ask about:

- the [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/latest/)
- Apple's Standard Directories
- Windows' Known Folder Locations
- the Unix single-folder convention (e.g. `~/.my-app`; the strategy trims, lowercases, and collapses each run of ASCII whitespace to a single `-`, so `"My App"` → `.my-app`)

A **ghost library** in the style of [whenwords](https://github.com/dbreunig/whenwords): this repo ships a spec (`SPEC.md`) and language-agnostic test cases (`tests.yaml`), not an implementation. Point your coding agent at them and generate the library in the language you actually want.

## Using it

1. Drop `SPEC.md` and `tests.yaml` into your project (or just point your coding assistant at this repo).
2. Follow [`INSTALL.md`](INSTALL.md): give your assistant the prompt, name the language, let it generate the library and a runner for `tests.yaml`.
3. Add your own cases to the `project:` block at the bottom of `tests.yaml` to lock in the exact paths your apps need.

## Design notes

- **Strategies:** `xdg`, `apple`, `windows`, `single-folder`. Pass one via `strategy=`, or omit for auto-mode.
- **Auto-mode is native everywhere**: `xdg` on Linux/BSD, `apple` on macOS, `windows` on Windows. To pick a non-native layout (e.g. `xdg` on macOS for cross-platform CLI consistency), pass `strategy=` explicitly.
- **Windows uses `%LOCALAPPDATA%` only.** No `roaming` flag — Microsoft [deprecated roaming AppData as of Windows 11](https://learn.microsoft.com/en-us/windows/apps/design/app-settings/store-and-retrieve-app-data#roaming-data) and recommends Local for all new applications. If `%LOCALAPPDATA%` is unset or empty, the resolver falls back to `%USERPROFILE%/AppData/Local`, then to `$HOME/AppData/Local`; if all three are unset, it raises. See [`SPEC.md` §Windows](SPEC.md#windows-strategywindows) for the full chain.
- **`author` follows each platform's own convention** — not a single uniform rule. On Apple it **dot-joins** with `app` into one segment matching Apple's bundle-ID recommendation (`~/Library/Application Support/com.acme.MyApp`). On Windows it **nests** as `<Company>\<Product>` per Microsoft guidance (`%LOCALAPPDATA%\Acme\MyApp`). XDG and single-folder ignore it.
- **No `version` parameter.** Versioned paths aren't part of any platform's spec — they're an `appdirs`/`platformdirs` Python convention. Callers that want versioned subdirs compose them themselves: `Path(config_dir("MyApp")) / "1.0"`.

## Comparison

How `wheredirs` differs from other libraries that solve the same problem. New libraries can be added as additional `###` subsections.

### platformdirs

[platformdirs](https://github.com/tox-dev/platformdirs) is the active Python library. `wheredirs` aligns with it on the auto-mode-is-native default (both return Apple-style paths on macOS by default, XDG on Linux/BSD, Windows-style on Windows). Major divergences:

- **No `roaming` flag.** platformdirs exposes a `roaming` boolean that affects `user_data_dir` / `user_config_dir` / `user_state_dir` / `user_log_dir` (cache is always local). `wheredirs` drops the flag entirely because Microsoft [deprecated roaming as of Windows 11](https://learn.microsoft.com/en-us/windows/apps/design/app-settings/store-and-retrieve-app-data#roaming-data) and recommends Local for all new applications.
- **No `version` parameter.** platformdirs auto-appends a version subdir when given. `wheredirs` drops it because no platform spec specifies a version-subdirectory convention — callers compose `Path(config_dir("MyApp")) / "1.0"` if they want one.
- **`author` works on Apple too.** platformdirs ignores `appauthor` on macOS (path is just `~/Library/Application Support/MyApp`). `wheredirs` dot-joins `author` with `app` into a bundle-ID-shaped single segment (`~/Library/Application Support/com.acme.MyApp`) matching Apple's File System Programming Guide recommendation.
- **`state_dir` and `runtime_dir` return `null` on Apple and Windows.** platformdirs always returns a path (aliased to other directories where the platform has no concept). `wheredirs` surfaces the platform asymmetry rather than papering over it; callers fall back via `state_dir(...) ?? data_dir(...)` if they want a path.
- **`single-folder` strategy.** platformdirs has no equivalent — there's no built-in way to get `~/.myapp/` paths.
- **`strategy=` parameter to override platform.** platformdirs picks the platform automatically from `sys.platform`; to force a different layout you instantiate a specific class (`Unix`, `MacOS`, `Windows`). `wheredirs` makes this a runtime keyword argument.
- **Smaller API surface.** `wheredirs` exposes seven functions (six dir resolvers + `which_strategy`). platformdirs exposes ~40 (including `site_*_dir`, `user_documents_dir`, `user_downloads_dir`, `user_pictures_dir`, etc.). `wheredirs` deliberately stops at user-app dirs; system-wide and user-content dirs are out of scope for v0.1.

### etcetera

[etcetera](https://github.com/lunacookies/etcetera) is the Rust reference library. Its design influenced several `wheredirs` decisions (Apple bundle-ID convention, returning `null` for absent concepts). Divergences:

- **`wheredirs` has auto-mode; etcetera does not.** etcetera offers two explicit constructors — `choose_native_strategy()` (uses `Apple` on macOS) and `choose_app_strategy()` (uses `Xdg` on macOS) — and forces the caller to pick. `wheredirs` makes auto-mode the default and provides `strategy="xdg"` as the opt-out for cross-platform CLI consistency on macOS.
- **Identity model.** etcetera takes three required strings: `top_level_domain`, `author`, `app_name`. It machine-generates the Apple bundle ID via `bundle_id() = "{tld}.{lowercase-dash(author)}.{dash(app_name)}"`. `wheredirs` takes two: `app` (required) and `author` (optional), and trusts the caller to pre-compose the dotted prefix if they want strict reverse-DNS (`author="com.acme"`). Less machinery, less normalisation; the caller does it themselves if they care.
- **Apple `config_dir` location.** etcetera puts config under `~/Library/Preferences/`. `wheredirs` puts it under `~/Library/Application Support/`. Apple's docs distinguish: `Preferences/` is for plist files managed by NSUserDefaults, `Application Support/` is for arbitrary app data files. `wheredirs` callers write their own TOML/JSON/YAML config (not NSUserDefaults plists), so `Application Support/` is the more accurate destination. platformdirs makes the same choice.
- **`single-folder` strategy.** Not in etcetera. In `wheredirs`.
- **`logs_dir` exists.** etcetera doesn't implement `log_dir` at all (it returns `None` everywhere). `wheredirs` implements it: `~/Library/Logs/{seg}` on Apple, `<state>/{app}/logs` on XDG (a wheredirs extension), `%LOCALAPPDATA%/{author}/{app}/Logs` on Windows.
- **Convergences worth noting** (where `wheredirs` aligned with etcetera over platformdirs): both return `null` for `state_dir`/`runtime_dir` on platforms with no equivalent concept; neither has a `version` parameter; neither has a `roaming` flag.

### appdirs

[appdirs](https://github.com/ActiveState/appdirs) is the older Python library that `platformdirs` was forked from. It is **no longer actively maintained** — its README explicitly redirects users to `platformdirs`. The API is very similar to platformdirs (it predates it), so all the [platformdirs divergences](#platformdirs) above apply to `appdirs` as well: no `roaming`, no `version`, `author` works on Apple, `state_dir`/`runtime_dir` return null on Apple/Windows, plus the `single-folder` and `strategy=` additions.

If you're choosing between `appdirs` and `wheredirs` for a new project, the comparison is really `platformdirs` vs `wheredirs` — `appdirs` shouldn't be a serious candidate in either column.

## Related projects

- [**whenwords**](https://github.com/dbreunig/whenwords) — the ghost-library template `wheredirs` is modelled after, for natural-language time-expression parsing.

## References

The behavioral specification and the official platform guides it derives from are listed in [`SPEC.md` § References](SPEC.md#references).

## Status

v0.1.0 — spec frozen, ready for generation. See [`SPEC.md`](SPEC.md) and [`tests.yaml`](tests.yaml).
