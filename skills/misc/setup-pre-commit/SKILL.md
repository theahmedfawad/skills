---
name: setup-pre-commit
description: Set up pre-commit-framework hooks for an embedded C/C++ repo — clang-format, cppcheck, and a build/test gate. Use when the user wants to add pre-commit hooks, configure clang-format/cppcheck/clang-tidy on commit, or add commit-time formatting/static-analysis/testing.
---

# Setup Pre-Commit Hooks

Sets up the [pre-commit framework](https://pre-commit.com) for a C/C++ firmware repo.

## What This Sets Up

- **pre-commit** framework, installed as the repo's git hook
- **clang-format** — formats staged `*.c/*.h/*.cpp/*.hpp` in place
- **cppcheck** — static analysis on staged sources, failing the commit on warnings
- A **build + test gate** (compile warnings-as-errors, then run the test suite) on `pre-push`
- A `.clang-format` config (if missing)

## Steps

### 1. Detect the build system

Look for:
`CMakeLists.txt` (CMake)
`Makefile` (Make)
`meson.build` (Meson)
`.project/.cproject` (STM32CubeIDE)
`platformio.ini` (PlatformIO)

This decides the build/test commands in step 4. If unclear, ask.

| Build system | Build | Test |
|---|---|---|
| CMake | `cmake --build build` | `ctest --test-dir build --output-on-failure` |
| Make | `make` | `make test` (or `make check`) |
| Meson | `meson compile -C build` | `meson test -C build` |
| STM32CubeIDE | `headless-build.bat -data <workspace> -import <project> -cleanBuild <project>/<config>` | — (host build) |
| PlatformIO | `pio run` | `pio test` |

For IDE / on-target build systems (STM32CubeIDE, and PlatformIO when targeting hardware), the firmware build only *compiles*. Unit tests still run on the **host** — point the `tests` hook at the host test runner (Ceedling/ctest), or use `pio test -e native` for PlatformIO's native environment.

### 2. Install pre-commit

Prefer an isolated install so it doesn't touch the project's toolchain:

```bash
pipx install pre-commit   # or: pip install --user pre-commit
```

Also ensure `clang-format` and `cppcheck` are installed (`apt`, `brew`, or the project's toolchain).

### 3. Create `.pre-commit-config.yaml`

```yaml
# Pin revs to current releases — run `pre-commit autoupdate` after creating this.
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-merge-conflict

  - repo: https://github.com/pocc/pre-commit-hooks
    rev: v1.3.5
    hooks:
      - id: clang-format
        args: [-i]                       # format staged sources in place
      - id: cppcheck
        args: [--enable=warning, --inline-suppr, --error-exitcode=1]

  # Build + tests are slower — run them on push, not on every commit.
  - repo: local
    hooks:
      - id: build
        name: build (warnings as errors)
        entry: cmake --build build       # adapt to the detected build system
        language: system
        pass_filenames: false
        stages: [pre-push]
      - id: tests
        name: unit tests
        entry: ctest --test-dir build --output-on-failure
        language: system
        pass_filenames: false
        stages: [pre-push]
```

**Adapt**: swap the `build`/`tests` `entry:` lines for the detected build system (step 1). If the repo has no test target yet, omit the `tests` hook and tell the user. To also run `clang-tidy`, add its `pocc` hook — but it needs a `compile_commands.json` (CMake: `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON`), so only add it when that exists.

The "typecheck" analog for C/C++ is **compile warnings-as-errors**: make sure the build is configured with `-Wall -Wextra -Werror` so the build hook actually catches type/usage mistakes.

### 4. Create `.clang-format` (if missing)

Only create if no `.clang-format` exists. Pick a base style the project already follows; a reasonable embedded default:

```yaml
---
BasedOnStyle: LLVM
IndentWidth: 4
TabWidth: 4
UseTab: Never
ColumnLimit: 100
PointerAlignment: Right
AllowShortFunctionsOnASingleLine: None
```

### 5. Install the hooks

```bash
pre-commit install                      # commit-stage hooks (clang-format, cppcheck)
pre-commit install --hook-type pre-push # build + tests on push
```

### 6. Verify

- [ ] `.pre-commit-config.yaml` and `.clang-format` exist
- [ ] `.git/hooks/pre-commit` and `.git/hooks/pre-push` exist (created by `pre-commit install`)
- [ ] `clang-format` and `cppcheck` are on `PATH`
- [ ] Run `pre-commit run --all-files` and confirm it passes (or only reformats)

### 7. Commit

Stage all changed/created files and commit with message: `Add pre-commit hooks (clang-format + cppcheck + build/test gate)`.

This runs through the new commit-stage hooks — a good smoke test that everything works.

## Notes

- Commit-stage hooks stay fast (format + static analysis on staged files only); the slow build/test gate runs on `pre-push`.
- `cppcheck --error-exitcode=1` makes findings fail the commit; tune `--enable=` to the project's tolerance.
- `pre-commit autoupdate` keeps the hook revs current — run it after first setup and periodically.
