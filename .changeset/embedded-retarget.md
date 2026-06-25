---
"mattpocock-skills": minor
---

Retarget the skill collection from web/TypeScript to embedded systems (C/C++, firmware, RTOS, HAL, drivers), and merge in embedded skills.

- **`tdd`** folds in the `embedded-unit-tests` content: Unity (default) / CppUTest / Google Test references, embedded edge-cases, and hardware-mocking mechanics (HAL function-pointer structs, register/peripheral fakes, controllable time, ISR/callback simulation, link-time substitution). Examples are C; the red-green-refactor discipline is unchanged.
- **`user-stories`** added as a new model-invoked engineering skill — a standalone INVEST backlog (acceptance criteria, Given/When/Then scenarios, priority, index). It delegates interactive elicitation to `/grilling` and stays distinct from `to-prd`/`to-issues`.
- **`reference-lookup`** added as a new model-invoked engineering skill — looks up and cites authoritative external references the code must conform to (datasheets, errata, schematics, register-map/SVD, protocol specs like CAN/USB/BLE/Modbus, standards like MISRA C / ISO 26262 / IEC 61508 / DO-178C, RFCs, vendor SDK/API docs) via a `docs/references/REFERENCES.md` manifest (local-first, web-fallback, always cite, never fabricate). `setup-matt-pocock-skills` gains an external-references step, and `tdd`/`diagnosing-bugs`/`codebase-design`/`domain-modeling`/`prototype` cross-link to it.
- Examples retargeted to embedded across `codebase-design` (HAL/struct injection), `diagnosing-bugs` (JTAG/RTT/serial/DWT), `prototype` (host-sim device-UI variants + firmware logic prototype), `domain-modeling` (ADR/context examples), `to-prd`, `triage`, `ask-matt`, and `setup-matt-pocock-skills`.
- **`setup-pre-commit`** rewritten around the pre-commit framework (clang-format + cppcheck on commit; build/test gate on pre-push) instead of Husky/Prettier/lint-staged. **`scaffold-exercises`** retargeted to embedded course authoring. **`migrate-to-shoehorn`** deprecated (TypeScript-only).
- The heavyweight `adr-best-practices`/`adr-template` skills were intentionally not merged; only `domain-modeling`'s ADR examples were retargeted.
