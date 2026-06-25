# Device UI Prototype

Generate **several radically different variations** of the device's user-facing surface — a display/menu layout, an indicator (LED/buzzer) scheme, or a serial CLI — switchable by a build flag or a runtime command. The user flips between variants, picks one (or steals bits from each), then throws the rest away.

If the question is about logic/state rather than what the device presents — wrong branch. Use [LOGIC.md](LOGIC.md).

## When this is the right shape

- "What should this OLED screen / menu tree look like?"
- "I want to see a few options for the status LED scheme before committing."
- "Try a different layout for the settings menu / a different shape for the serial command set."
- Any time the user would otherwise spend a day picking between three vague pictures in their head.

## What "the surface" is, for embedded

Pick the one the question is about:

- **Display** — LCD/OLED/e-ink screen layouts, menu trees, what's shown in each state.
- **Indicators** — LED blink codes, RGB colour states, buzzer/haptic patterns.
- **Serial CLI / protocol** — the command names, arguments, prompts, and output format a user or tool sees over UART/USB.

## Where to run it — strongly prefer the host simulator

A device UI is much faster to judge when it renders on a **host build** — compile the presentation layer for your dev machine and draw the display to a terminal (or an SDL/ncurses window), print the LED/buzzer state as text, or expose the CLI on a local pty. No flashing, no rebuild-and-reflash cycle between variants; switching is instant. Default to the host simulator whenever the presentation layer can be separated from the hardware it drives.

Only prototype **on target** when the thing being judged genuinely depends on real hardware — actual display contrast/refresh, real LED colour/brightness, real button timing. Then flash each variant (or one build that switches at runtime) and capture the result over serial/RTT, a logic analyzer, or a photo/recording of the device.

## Process

### 1. State the question and pick N

Default to **3 variants**. More than 5 stops being radically different and starts being noise — cap there.

Write down the plan in one line, in the prototype's location or a top-of-file comment:

> "Three layouts for the status screen, switchable via the `VARIANT` build flag, rendered by the host display simulator."

This works whether the user is here to push back or not.

### 2. Generate radically different variants

Draft each variant. Hold each one to:

- The surface's purpose and the data the firmware actually has to show.
- The project's existing drawing/HAL primitives (the display driver's `draw_text`/`draw_icon`, the LED API, the CLI registration macros) — don't invent a new graphics stack.
- A clear named entry point, e.g. `render_variant_a()`, `render_variant_b()`, `render_variant_c()`.

Variants must be **structurally different** — different information hierarchy, different menu depth, different primary affordance, not just different fonts or colours. Three near-identical status screens isn't a prototype, it's wallpaper. If two drafts come out too similar, redo one with explicit "do not reuse that layout" guidance.

### 3. Wire them together

Select the active variant at **compile time** (a build flag) or **run time** (a serial command). Keep the firmware's real data feeding the selected renderer; only the rendering swaps.

```c
// pseudo-code — adapt to the project's build system / CLI
#ifndef VARIANT
#define VARIANT 'A'
#endif

void render_screen(const ui_state_t *s) {
    switch (active_variant) {       // compile-time default, or set by a serial command
        case 'A': render_variant_a(s); break;
        case 'B': render_variant_b(s); break;
        case 'C': render_variant_c(s); break;
    }
}
```

Build-flag switching (`-DVARIANT=B`) is simplest when reflashing is cheap or you're on the host sim. A runtime serial command (`variant b`) is better on target, so the user cycles variants without a rebuild.

### 4. Build the variant switcher

Give the user a single, obvious way to cycle — the embedded analog of a switcher bar:

- **Host simulator** — a keypress (`←`/`→` or `n`/`p`) cycles the variant and re-renders; show the current variant key and name in a status line, e.g. `B — compact list`.
- **On target** — a serial command (`variant a|b|c`, plus `variant next`) selects and redraws; echo the active variant name back over serial. A spare button can cycle if no console is attached.

Make the selector visually/textually distinct from the surface being evaluated (a status line, a log prefix) so it's obviously not part of the design. Gate it out of release builds — `#if PROTOTYPE_UI` or an equivalent flag — so a stray merge can't ship the switcher.

### 5. Hand it over

Tell the user how to switch (the build flag values, or the serial command and keys). They'll flip through whenever they get to it. The interesting feedback is usually **"I want the top bar from B with the menu from C"** — that's the actual design they want.

### 6. Capture the answer and clean up

Once a variant has won, write down which one and why (commit message, ADR, issue, or a `NOTES.md` next to the prototype if running AFK). Then:

- Delete the losing `render_variant_*` functions and the switcher.
- Fold the winner into the real presentation layer, behind the project's normal drawing/HAL primitives.

Don't leave variant renderers or the switcher lying around. They rot fast and confuse the next reader.

## Anti-patterns

- **Variants that differ only in font or colour.** That's a tweak, not a prototype. Real variants disagree about structure and hierarchy.
- **Sharing too much code between variants.** Shared low-level primitives (`draw_text`, the LED API) are fine; a shared layout function defeats the point. Each variant should be free to throw out the layout.
- **Driving real actuators/outputs with side effects.** Read-only/display-only prototypes are fine. If a variant would trigger a motor, relay, or write to flash, point it at a stub — the question is "what should this look like", not "does the actuator work".
- **Promoting the prototype directly to production.** The variant code was written under prototype constraints (no tests, minimal error handling). Rewrite it properly when you fold it in.
- **Forcing on-target when the host sim would do.** If the presentation layer can be separated from the hardware, prototype on the host — reflashing between variants kills the iteration speed that makes this worthwhile.
