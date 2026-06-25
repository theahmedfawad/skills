---
name: reference-lookup
description: Look up and cite authoritative external references the code must conform to — datasheets, reference manuals, errata, schematics, communication-protocol specs (CAN/USB/BLE/Modbus), standards (MISRA C, ISO 26262, IEC 61508, DO-178C), RFCs, and vendor SDK/API docs. Use when you need any externally-defined fact: a register address/bitfield/electrical limit/timing, a protocol field or frame layout, a standard's requirement, or an API contract. Local-first, web-fallback, always cite; never invent a value.
---

# Reference Lookup

Embedded code conforms to external sources of truth — the silicon docs, the protocol spec, the standard, the vendor SDK — not to what seems reasonable. Inventing a register offset, a protocol field width, a timing limit, or a standard's requirement is the fastest way to ship a bug that only shows up on hardware or in certification. This skill is the discipline for getting those facts right and **citing** them.

It is reference-type-agnostic: any authoritative document the project depends on can be added to the manifest and looked up the same way.

## The manifest

The project's sources of truth are indexed in `docs/references/REFERENCES.md` (created by `/setup-matt-pocock-skills`, or lazily on first need). Each entry names the document, its exact identifier and version/revision, a local path, and a fallback URL:

```md
# References

| Type | Document | Id / version | Local | Fallback URL |
|---|---|---|---|---|
| Hardware | MCU datasheet | STM32F407VG | docs/references/stm32f407-ds.pdf | https://www.st.com/.../stm32f407vg.pdf |
| Hardware | Reference manual | RM0090 rev 19 | docs/references/rm0090.pdf | https://www.st.com/.../rm0090.pdf |
| Hardware | Errata | ES0182 rev 10 | — | https://www.st.com/.../es0182.pdf |
| Hardware | Register map (SVD) | STM32F407 | svd/STM32F407.svd | — |
| Hardware | Board schematic | board v3 | hardware/board-v3.pdf | — |
| Protocol | CAN 2.0 | Bosch 1991 | docs/references/can2.0.pdf | — |
| Protocol | Modbus application | v1.1b3 | — | https://modbus.org/docs/Modbus_Application_Protocol_V1_1b3.pdf |
| Standard | MISRA C | 2012 (+Amd) | docs/references/misra-c-2012.pdf | — |
| Standard | Functional safety | ISO 26262:2018 | — | (purchased / controlled copy) |
| SDK/API | Vendor HAL docs | SDK v2.14 | docs/references/hal-v2.14/ | https://vendor.example/sdk/2.14 |
```

Version/revision matters — errata, protocol amendments, and standard editions differ. Always resolve the entry for the **exact identifier and version** the project targets.

## When to reach for this

Any time an externally-defined fact is load-bearing:

- **Hardware** — register address/offset/bitfield/reset value; electrical limits (Vdd, max current, absolute maximums); timing (clock tree, dividers, setup/hold, baud); pin alternate-functions; peripheral sequences; errata.
- **Protocol / comms** — frame layout, field widths, byte order, CRC polynomial, address ranges, timing/timeout requirements (CAN, USB, BLE, Modbus, I2C/SPI device protocols, MQTT, etc.).
- **Standards / compliance** — a specific rule or requirement (MISRA C rule text, an ISO 26262 / IEC 61508 / DO-178C clause).
- **Vendor SDK / API** — the documented contract of an SDK call, its preconditions, and error semantics.

## Lookup procedure

1. **Read the manifest** (`docs/references/REFERENCES.md`). Resolve the document for the exact identifier + version.
2. **Local first.** Open the local copy if present — the PDF page/section, the SVD register, the schematic net, the SDK doc. The local copy is the controlled source.
3. **Manifest URL.** If the local copy is missing, fetch the URL in the manifest entry.
4. **Web fallback.** If there's no manifest entry, search the web for the **official / authoritative** document for that exact identifier and version (prefer the vendor's or standards body's own site). Add it to the manifest once found.
5. **Check for amendments/errata.** Cross-check errata sheets, protocol amendments, or standard corrigenda for the version in use — a value "correct per the base document" may be superseded.
6. **Cite.** State the source inline: document, version/rev, section/page/clause, and the specific register, field, or rule. Citations make the fact verifiable instead of trusted.
7. **Never fabricate.** If a value can't be found in a local doc, the manifest URL, or an authoritative web source, say so explicitly and ask the user — do not guess a register address, protocol field, timing value, or standard requirement.

## Citing a fact

Make the source checkable, e.g.:

> `USART1->BRR = 0x0683;` — 9600 baud @ 16 MHz PCLK2. Source: RM0090 rev 19, §30.5.4 (USART_BRR); baud formula §30.3.4.

> Standard CAN data frame: 11-bit identifier, DLC 0–8 bytes. Source: CAN 2.0A (Bosch 1991), §3.1.1.

> Avoid implicit conversion that loses sign. Source: MISRA C:2012, Rule 10.3.

## Recording decisions

A looked-up fact is not a domain term — keep it out of `CONTEXT.md`. But a **constraint that shapes the design** — "ISR latency budget is 10 µs per the peripheral's max interrupt rate", "no DMA on SPI3, see ES0182 §2.x", "no dynamic allocation in the safety path per ISO 26262" — is exactly the "constraint not visible in the code" case for an ADR. Record those via `/domain-modeling`, citing the source.

## Keeping the manifest current

When you resolve a document from the web that wasn't in the manifest, add it (type, identifier, version, URL, and a local path once the team checks in a controlled copy). The manifest is the team's curated set of sources of truth — grow it as you go.
