# ADR Format

ADRs live in `docs/adr/` and use sequential numbering: `0001-slug.md`, `0002-slug.md`, etc.

Create the `docs/adr/` directory lazily — only when the first ADR is needed.

## Template

```md
# {Short title of the decision}

{1-3 sentences: what's the context, what did we decide, and why.}
```

That's it. An ADR can be a single paragraph. The value is in recording *that* a decision was made and *why* — not in filling out sections.

## Optional sections

Only include these when they add genuine value. Most ADRs won't need them.

- **Status** frontmatter (`proposed | accepted | deprecated | superseded by ADR-NNNN`) — useful when decisions are revisited
- **Considered Options** — only when the rejected alternatives are worth remembering
- **Consequences** — only when non-obvious downstream effects need to be called out

## Numbering

Scan `docs/adr/` for the highest existing number and increment by one.

## When to offer an ADR

All three of these must be true:

1. **Hard to reverse** — the cost of changing your mind later is meaningful
2. **Surprising without context** — a future reader will look at the code and wonder "why on earth did they do it this way?"
3. **The result of a real trade-off** — there were genuine alternatives and you picked one for specific reasons

If a decision is easy to reverse, skip it — you'll just reverse it. If it's not surprising, nobody will wonder why. If there was no real alternative, there's nothing to record beyond "we did the obvious thing."

### What qualifies

- **Architectural shape.** "This is RTOS based project." "ISRs, software timers and callbacks do minimal required action, set flag, push event to Queue or send signal to task; all processing happens in task or main loop."
- **Integration patterns between modules.** "HAL interface is a contract between driver and servic." "dependency app -> service -> HAL -> Driver." **Technology choices that carry lock-in.** MCU family/vendor, RTOS, toolchain, bootloader, bus protocol, build system. Not every library — just the ones that would take a quarter to swap out. "We're on Zephyr, not FreeRTOS." "Register-level access, not the vendor HAL."
- **Boundary and scope decisions.** "The HAL owns all peripheral access; nothing else includes the vendor headers." "Flash is partitioned bootloader / app / config; the app never writes outside its slot." The explicit no-s are as valuable as the yes-s.
- **Deliberate deviations from the obvious path.** "Static allocation only — no `malloc` after init." "We poll the ADC instead of using DMA because X." Anything where a reasonable reader would assume the opposite. These stop the next engineer from "fixing" something that was deliberate.
- **Constraints not visible in the code.** "Must fit in 64 KB flash / 8 KB RAM." "ISR latency must stay under 10 µs." "No floating point — the Cortex-M0 has no FPU." "ISO 26262 forbids dynamic allocation in the safety path."
- **Rejected alternatives when the rejection is non-obvious.** If you considered DMA-driven SPI and picked interrupt-driven for subtle buffer-coherency reasons, record it — otherwise someone will suggest DMA again in six months.
