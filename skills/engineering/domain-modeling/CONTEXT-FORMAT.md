# CONTEXT.md Format

## Structure

```md
# {Context Name}

{One or two sentence description of what this context is and why it exists.}

## Language

**Sample**:
{A one or two sentence description of the term}
_Avoid_: Reading, measurement

**Frame**:
A complete, framed message on the comms bus.
_Avoid_: Packet, datagram, message

**Fault**:
A detected condition that takes the system out of normal operation.
_Avoid_: Error, failure, alarm
```

## Rules

- **Be opinionated.** When multiple words exist for the same concept, pick the best one and list the others under `_Avoid_`.
- **Keep definitions tight.** One or two sentences max. Define what it IS, not what it does.
- **Only include terms specific to this project's context.** General programming concepts (timeouts, error types, utility patterns) don't belong even if the project uses them extensively. Before adding a term, ask: is this a concept unique to this context, or a general programming concept? Only the former belongs.
- **Group terms under subheadings** when natural clusters emerge. If all terms belong to a single cohesive area, a flat list is fine.

## Single vs multi-context repos

**Single context (most repos):** One `CONTEXT.md` at the repo root.

**Multiple contexts:** A `CONTEXT-MAP.md` at the repo root lists the contexts, where they live, and how they relate to each other:

```md
# Context Map

## Contexts

- [Motor Control](./src/motor-control/CONTEXT.md) — runs the closed-loop control of the motor
- [Telemetry](./src/telemetry/CONTEXT.md) — collects sensor data and builds reports
- [Comms](./src/comms/CONTEXT.md) — handles the external protocol stack over the bus

## Relationships

- **Motor Control → Telemetry**: Motor Control emits `StateChanged` events; Telemetry samples them for reporting
- **Telemetry → Comms**: Telemetry enqueues frames; Comms transmits them over the bus
- **Motor Control ↔ Comms**: Shared types for `MotorId` and `Millivolts`
```

The skill infers which structure applies:

- If `CONTEXT-MAP.md` exists, read it to find contexts
- If only a root `CONTEXT.md` exists, single context
- If neither exists, create a root `CONTEXT.md` lazily when the first term is resolved

When multiple contexts exist, infer which one the current topic relates to. If unclear, ask.
