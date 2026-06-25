# Logic Prototype

A tiny interactive terminal app that lets the user drive a state model by hand. Use this when the question is about **business logic, state transitions, or data shape** — the kind of thing that looks reasonable on paper but only feels wrong once you push it through real cases.

## When this is the right shape

- "I'm not sure if this state machine handles the edge case where X then Y."
- "Does this data model actually let me represent the case where..."
- "I want to feel out what the API should look like before writing it."
- Anything where the user wants to **press buttons and watch state change**.

If the question is about what the device presents or how the user interacts — wrong branch. Use [UI.md](UI.md).

## Process

### 1. State the question

Before writing code, write down what state model and what question you're prototyping. One paragraph, in the prototype's README or a comment at the top of the file. A logic prototype that answers the wrong question is pure waste — make the question explicit so it can be checked later, whether the user is watching now or returning to it AFK.

### 2. Pick the language

Use whatever the host project uses. For firmware, build the logic for the **host** (compile it for your dev machine, not the target) so the prototype runs as an ordinary terminal program — no flashing, no debugger, instant iteration. Include external dependcies if needed.

Match the project's existing conventions for tooling — don't add a new build system or toolchain just for the prototype.

### 3. Isolate the logic in a portable module

Put the actual logic — the bit that's answering the question — behind a small, pure interface that could be lifted out and dropped into the real codebase later. The TUI around it is throwaway; the logic module shouldn't be.

The right shape depends on the question:

- **A state machine** — explicit states and transitions. Good when "which actions are even legal right now" is part of the question.
- **A small set of pure functions** over a plain data type. Good when there's no implicit current state — just transformations.
- **A module with a clear method surface** when the logic genuinely owns ongoing internal state.

Pick whichever shape best fits the question being asked, *not* whichever is easiest to wire to a TUI. Keep it pure: no I/O, no register access, no terminal code, no `printf` for control flow. The TUI calls into it; nothing flows the other direction. This is exactly the host-testable shape the `/tdd` skill wants — the logic compiles for the host with no hardware behind it.

This is what makes the prototype useful past its own lifetime. When the question's been answered, the validated state machine / function set can be lifted straight into the real firmware module — the TUI shell gets deleted.

### 4. Build the smallest TUI that exposes the state

Build it as a **lightweight TUI** — on every tick, clear the screen (`print("\033[2J\033[H")` / equivalent) and re-render the whole frame. The user should always see one stable view, not an ever-growing scrollback.

Each frame has two parts, in this order:

1. **Current state**, pretty-printed and diff-friendly — one struct field per line. Use **bold** for field names or section headers and **dim** for less important context (tick counts, handles, derived values). Native ANSI escape codes are fine — `\x1b[1m` bold, `\x1b[2m` dim, `\x1b[0m` reset. No need to pull in a styling library.
2. **Keyboard shortcuts**, listed at the bottom: `[u] value up  [d] value down  [t] tick clock  [i] inject IRQ  [q] quit`. Bold the key, dim the description, or vice-versa — whatever reads cleanly.

Behaviour:

1. **Initialise state** — a single in-memory struct. Render the first frame on start.
2. **Read one keystroke (or one line)** at a time, dispatch to a handler that drives the state machine. Map keys to the events the firmware will really see — a timer tick, an IRQ, a received frame, a button edge.
3. **Re-render** the full frame after every action — don't append, replace.
4. **Loop until quit.**

The whole frame should fit on one screen.

### 5. Make it runnable in one command

Add a target to the project's existing build system (`Makefile`, `CMakeLists.txt`, `justfile`). The user should run `make <prototype-name>` (or `cmake --build … && ./<host-sim>`) — never need to remember a path.

If the host project has no task runner, just put the command at the top of the prototype's README.

### 6. Hand it over

Give the user the run command. They'll drive it themselves; the interesting moments are when they say "wait, that shouldn't be possible" or "huh, I assumed X would be different" — those are the bugs in the _idea_, which is the whole point. If they want new actions added, add them. Prototypes evolve.

### 7. Capture the answer

When the prototype has done its job, the answer to the question is the only thing worth keeping. If the user is around, ask what it taught them. If not, leave a `NOTES.md` next to the prototype so the answer can be filled in (or filled in by you, if you've watched the session) before the prototype gets deleted.

## Anti-patterns

- **Don't add tests.** A prototype that needs tests is no longer a prototype.
- **Don't wire it to real hardware.** Drive peripherals, registers, and timers through in-memory stubs unless the question is specifically about a device's behaviour.
- **Don't generalise.** No "what if we wanted to support X later." The prototype answers one question.
- **Don't blur the logic and the TUI together.** If the state machine references `printf`, blocking input, or terminal escape codes, it's no longer portable — and no longer host-testable. Keep the TUI as a thin shell over a pure module.
- **Don't ship the TUI shell into production.** The shell is optimised for being driven by hand from a terminal. The logic module behind it is the bit worth keeping.
