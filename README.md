# wheredirs

Show where standard application directories (config, cache, data, state, logs, runtime) resolve to — for the current platform, or for any platform and convention you ask about:

- the [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/latest/)
- Apple's Standard Directories
- Windows' Known Folder Locations
- the Unix single-folder convention (`~/.myapp`)

A **ghost library** in the style of [whenwords](https://github.com/dbreunig/whenwords): this repo ships a spec (`SPEC.md`) and language-agnostic test cases (`tests.yaml`), not an implementation. Point your coding agent at them and generate the library in the language you actually want.

Reference libraries for the problem space: [platformdirs](https://github.com/tox-dev/platformdirs) (Python) and [etcetera](https://github.com/lunacookies/etcetera) (Rust). They disagree on macOS — `wheredirs` exposes the strategy choice instead of hiding it.

## Status

Early. `SPEC.md` and `tests.yaml` not yet written.
