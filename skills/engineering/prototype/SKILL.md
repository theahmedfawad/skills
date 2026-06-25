---
name: prototype
description: Build a throwaway prototype to flesh out a design — a runnable terminal app for state/business-logic questions, or several radically different device-UI variations (display/menu/indicator/CLI) switchable by a build flag or serial command.
disable-model-invocation: true
---

# Prototype

A prototype is **throwaway code that answers a question**. The question decides the shape.

## Pick a branch

Identify which question is being answered — from the user's prompt, the surrounding code, or by asking if the user is around:

- **"Does this logic / state model feel right?"** → [LOGIC.md](LOGIC.md). Build a tiny interactive terminal app (a host build) that pushes the state machine through cases that are hard to reason about on paper.
- **"What should the device present / how should the user interact?"** → [UI.md](UI.md). Generate several radically different device-UI variations — display/menu layouts, indicator (LED/buzzer) patterns, or the serial CLI — switchable by a build flag or a runtime serial command.

The two branches produce very different artifacts — getting this wrong wastes the whole prototype. If the question is genuinely ambiguous and the user isn't reachable, default to whichever branch better matches the surrounding code (a state machine or driver → logic; a display, menu, indicator, or CLI surface → device UI) and state the assumption at the top of the prototype.

## Rules that apply to both

1. **Throwaway from day one, and clearly marked as such.** Locate the prototype code close to where it will actually be used (next to the module or driver it's prototyping for) so context is obvious — but name it so a casual reader can see it's a prototype, not production. For throwaway build targets or variant flags, obey whatever build-system convention the project already uses; don't invent a new top-level structure.
2. **One command to run.** Whatever the project's existing task runner supports — `make <target>`, `cmake --build … && ./<host-sim>`, a flashing script, etc. The user must be able to start it without thinking.
3. **No persistence by default.** State lives in RAM. Persistence is the thing the prototype is _checking_, not something it should depend on. If the question explicitly involves non-volatile storage, hit a scratch flash/EEPROM region or a host file with a clear "PROTOTYPE — wipe me" name.
4. **Skip the polish.** No tests, no error handling beyond what makes the prototype _runnable_, no abstractions. The point is to learn something fast and then delete it.
5. **Surface the state.** After every action (logic) or on every variant switch (UI), print or render the full relevant state so the user can see what changed.
6. **Delete or absorb when done.** When the prototype has answered its question, either delete it or fold the validated decision into the real code — don't leave it rotting in the repo.

## When done

The _answer_ is the only thing worth keeping from a prototype. Capture it somewhere durable (commit message, ADR, issue, or a `NOTES.md` next to the prototype) along with the question it was answering. If the user is around, that capture is a quick conversation; if not, leave the placeholder so they (or you, on the next pass) can fill in the verdict before deleting the prototype.
