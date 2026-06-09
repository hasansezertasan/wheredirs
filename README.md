# wheredirs

Show where standard application directories (config, cache, data, state, logs, runtime) resolve to — for the current platform, or for any platform and convention you ask about:

- the [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/latest/)
- Apple's Standard Directories
- Windows' Known Folder Locations
- the Unix single-folder convention (e.g. `~/.my-app`; the strategy lowercases and dash-joins, so `"My App"` → `.my-app`)

A **ghost library** in the style of [whenwords](https://github.com/dbreunig/whenwords): this repo ships a spec (`SPEC.md`) and language-agnostic test cases (`tests.yaml`), not an implementation. Point your coding agent at them and generate the library in the language you actually want.

Reference libraries for the problem space: [platformdirs](https://github.com/tox-dev/platformdirs) (Python) and [etcetera](https://github.com/lunacookies/etcetera) (Rust). On macOS they diverge: `platformdirs` defaults to the native Apple layout (`~/Library/Application Support/...`), while `etcetera` makes the caller pick a strategy explicitly. `wheredirs` follows etcetera — auto-mode resolves to XDG on macOS, and the Apple layout is opt-in via `strategy="apple"`.

## Using it

1. Drop `SPEC.md` and `tests.yaml` into your project (or just point your coding assistant at this repo).
2. Follow [`INSTALL.md`](INSTALL.md): give your assistant the prompt, name the language, let it generate the library and a runner for `tests.yaml`.
3. Add your own cases to the `project:` block at the bottom of `tests.yaml` to lock in the exact paths your apps need.

## Design notes

- **Strategies:** `xdg`, `apple`, `windows`, `single-folder`. Pass one via `strategy=`, or omit for auto-mode.
- **Auto-mode on macOS picks `xdg`**, not `apple`. The native Apple layout is opt-in via `strategy="apple"`. See SPEC.md for the rationale.
- **Windows defaults to local (`%LOCALAPPDATA%`)**. Pass `roaming=True` to switch to `%APPDATA%`.
- **`author` is Windows-only.** Every other strategy ignores it.
- **Version, when given, is appended as a final subdir** (`.../MyApp/1.0`).

## Status

v0.1.0 — spec frozen, ready for generation. See [`SPEC.md`](SPEC.md) and [`tests.yaml`](tests.yaml).
