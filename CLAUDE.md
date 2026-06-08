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

`wheredirs` resolves where standard application directories (config, cache, data, state, logs, runtime, plus their "local" vs "roaming" variants on Windows) live for a given app on the current platform. The two reference libraries that define the problem space:

- [`platformdirs`](https://github.com/tox-dev/platformdirs) (Python) — opinionated per-OS resolution; XDG on Linux, native conventions on macOS/Windows.
- [`etcetera`](https://github.com/lunacookies/etcetera) (Rust) — strategy-based; the caller picks XDG / Apple / Windows explicitly.

They disagree on macOS in particular (XDG-style `~/.config/...` vs native `~/Library/Application Support/...`). The spec must take a position on this, or expose strategy as an input — silently picking one will make `tests.yaml` non-portable.

## Working on the spec

When editing `SPEC.md` or `tests.yaml`:

- Every test case must pin down all inputs that affect the output: platform, relevant env vars (`HOME`, `XDG_CONFIG_HOME`, `XDG_DATA_HOME`, `XDG_CACHE_HOME`, `XDG_STATE_HOME`, `XDG_RUNTIME_DIR`, `APPDATA`, `LOCALAPPDATA`), app name, author, version, and any roaming/opinion flags. A case that depends on ambient state a target-language implementation can't reproduce is a bug in the case.
- Expected paths in `tests.yaml` should use forward slashes and a placeholder for the home directory (e.g. `$HOME`) so they're comparable across languages without each implementation re-deriving the user's actual home path.
- Prefer adding a failing test case over describing a behavior in prose only. The YAML is the source of truth that downstream implementations are graded against; SPEC.md is the human-readable rationale.

## Conventions

- Commits: Conventional Commits.
- Branches: Conventional Branch.
- PR titles: Conventional Pull Request action format.
