# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

`wheredirs` is a **ghost library** in the style of [`whenwords`](https://github.com/dbreunig/whenwords): the repo ships a specification and a language-agnostic test suite, **not** an implementation. End users hand their coding agent a prompt that points at `SPEC.md` and `tests.yaml`, and the agent generates the library in their target language locally.

Concretely that means:

- There is no source code in this repo, and there should not be. No `src/`, no language-specific package manifests for the library itself.
- The deliverables are: `SPEC.md` (behavior contract), `tests.yaml` (input/output cases), `INSTALL.md` (the prompt users paste into their agent), and optionally a `VERIFY.md` checklist. The `README.md` explains the model.
- "Tests" in this repo means *spec test cases* in YAML, not runnable code. There is nothing to `pytest` or `cargo test` here.

If you find yourself about to add a Python module, a `pyproject.toml` for the library, or a CI matrix that runs code — stop. That work belongs in a downstream implementation repo (see `whenwords-examples` for the pattern), not here.

## Domain

`wheredirs` resolves where standard application directories (config, data, cache, state, logs, runtime) live for a given app on the current platform. The three reference libraries that define the problem space:

- [`platformdirs`](https://github.com/tox-dev/platformdirs) (Python) — opinionated per-OS resolution; XDG on Linux, native conventions on macOS/Windows. Exposes a `roaming` flag for Windows.
- [`appdirs`](https://github.com/ActiveState/appdirs) (Python) — the older library `platformdirs` was forked from; no longer actively maintained.
- [`etcetera`](https://github.com/lunacookies/etcetera) (Rust) — strategy-based; the caller picks XDG / Apple / Windows explicitly. Machine-generates the Apple bundle ID from (TLD, author, app_name).

Current spec position on the key design pressure points (see SPEC.md and the README "Related projects" section for the full divergence table vs platformdirs):

- **macOS auto-mode picks `apple`** (native), matching platformdirs and diverging from etcetera's explicit-only design. Callers wanting cross-platform CLI consistency pass `strategy="xdg"`.
- **No `roaming` flag.** Microsoft deprecated roaming AppData as of Windows 11; the Windows strategy uses `%LOCALAPPDATA%` only. Do not reintroduce `%APPDATA%` without explicit user direction.
- **`author` is per-platform-convention.** Apple dot-joins into one bundle-ID segment; Windows path-joins as `<Company>\<Product>`; XDG and single-folder ignore it.
- **`state_dir` and `runtime_dir` return `null` on Apple and Windows** — those platforms have no equivalent concept. Callers fall back via `state_dir(...) ?? data_dir(...)`.
- **Four strategies**: `xdg`, `apple`, `windows`, `single-folder`. The fourth is a wheredirs addition (platformdirs and etcetera have no equivalent).

When considering a spec change, run it through the principle: **"strategies should apply what their conventions recommend, not opinions."** If wheredirs is inventing behavior the source platform doesn't define, flag it with a "wheredirs extension" callout in SPEC.md; if the platform genuinely has no concept, return `null` rather than inventing an alias.

## Working on the spec

When editing `SPEC.md` or `tests.yaml`:

- Every test case must pin down all inputs that affect the output: platform, relevant env vars (`HOME`, `XDG_CONFIG_HOME`, `XDG_DATA_HOME`, `XDG_CACHE_HOME`, `XDG_STATE_HOME`, `XDG_RUNTIME_DIR`, `LOCALAPPDATA`, `USERPROFILE`), app name, author. A case that depends on ambient state a target-language implementation can't reproduce is a bug in the case.
- Expected paths in `tests.yaml` should use forward slashes and a placeholder for the home directory (e.g. `$HOME`) so they're comparable across languages without each implementation re-deriving the user's actual home path.
- A case may have `output:` set to (a) a path string with placeholders, (b) `null` to assert the function returns the language's null value, or (c) omit `output` and set `raises: true` to assert an error is raised.
- Prefer adding a failing test case over describing a behavior in prose only. The YAML is the source of truth that downstream implementations are graded against; SPEC.md is the human-readable rationale.

## Conventions

- Commits: Conventional Commits.
- Branches: Conventional Branch.
- PR titles: Conventional Pull Request action format.
