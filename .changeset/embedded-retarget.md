---
"mattpocock-skills": minor
---

Retarget the skill collection from web/TypeScript to embedded systems (C/C++, firmware, RTOS, HAL, drivers), and merge in embedded skills.

- **`tdd`** folds in the `embedded-unit-tests` content: Unity (default) / CppUTest / Google Test references, embedded edge-cases, and hardware-mocking mechanics (HAL function-pointer structs, register/peripheral fakes, controllable time, ISR/callback simulation, link-time substitution). Examples are C; the red-green-refactor discipline is unchanged.
- **`user-stories`** added as a new model-invoked engineering skill — a standalone INVEST backlog (acceptance criteria, Given/When/Then scenarios, priority, index). It delegates interactive elicitation to `/grilling` and stays distinct from `to-prd`/`to-issues`.
- Examples retargeted to embedded across `codebase-design` (HAL/struct injection), `diagnosing-bugs` (JTAG/RTT/serial/DWT), `prototype` (host-sim device-UI variants + firmware logic prototype), `domain-modeling` (ADR/context examples), `to-prd`, `triage`, `ask-matt`, and `setup-matt-pocock-skills`.
- **`setup-pre-commit`** rewritten around the pre-commit framework (clang-format + cppcheck on commit; build/test gate on pre-push) instead of Husky/Prettier/lint-staged. **`scaffold-exercises`** retargeted to embedded course authoring. **`migrate-to-shoehorn`** deprecated (TypeScript-only).
- The heavyweight `adr-best-practices`/`adr-template` skills were intentionally not merged; only `domain-modeling`'s ADR examples were retargeted.
