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

Give each entry a **section/page index** for the topics this project actually touches, so lookups read a bounded region instead of scanning the whole document. Grow it as you resolve facts — either a per-doc anchor sub-list under the table, or anchors inline:

```md
## Section index (read these page ranges, not the whole PDF)

- **RM0090 rev 19** — USART → §30; USART_BRR → §30.5.4 / pp.990–992; RCC clock tree → §6 / pp.150–230; DMA → §10.
- **STM32F407 datasheet** — absolute maximum ratings → §6.2 / pp.65–68; pin alternate functions → Table 9.
```

These docs are enormous (RM0090 is ~1,750 pages). Reading one to find a single bitfield can cost tens of thousands of tokens — the index plus the lookup procedure below exist to keep that cheap.

## When to reach for this

Any time an externally-defined fact is load-bearing:

- **Hardware** — register address/offset/bitfield/reset value; electrical limits (Vdd, max current, absolute maximums); timing (clock tree, dividers, setup/hold, baud); pin alternate-functions; peripheral sequences; errata.
- **Protocol / comms** — frame layout, field widths, byte order, CRC polynomial, address ranges, timing/timeout requirements (CAN, USB, BLE, Modbus, I2C/SPI device protocols, MQTT, etc.).
- **Standards / compliance** — a specific rule or requirement (MISRA C rule text, an ISO 26262 / IEC 61508 / DO-178C clause).
- **Vendor SDK / API** — the documented contract of an SDK call, its preconditions, and error semantics.

## Lookup procedure

Resolve facts in **cheapest-source-first** order. Each step is far cheaper than the next — don't skip ahead to reading a PDF when an earlier step answers it.

0. **Check the resolved-facts ledger first** (`docs/references/facts.md`). If the fact is already there *and* the source version matches the manifest, use it directly — a ledger hit costs tens of tokens versus thousands. (Version mismatch → treat the row as stale; re-verify below.)
1. **Read the manifest** (`docs/references/REFERENCES.md`) for the document at the exact identifier + version, and its section index.
2. **Prefer machine-readable in-repo sources.** For a register address/offset/bitfield/reset-value, a clock constant, or a pin/AF number, the **CMSIS device header** (e.g. `stm32f4xx.h`), the **vendor HAL headers**, and the **SVD** already encode it, live in the repo, and are grep-able for almost nothing — `grep -n USART1_BASE stm32f4xx.h` beats reading RM0090. Reach for the PDF only for what headers can't carry: field *semantics*, peripheral power-up/configuration *sequences*, *timing*, *electrical* limits, and *errata*.
3. **Local PDF — read surgically, never whole.** Use the manifest's section index to read a **bounded page range** with the Read tool's `pages` argument (it's required for PDFs over 10 pages anyway), or `grep` a pre-extracted `.txt`. Never load a whole datasheet/manual into context. If the topic has no anchor yet, locate it once and **record the anchor in the manifest** so the next lookup is targeted.
4. **Delegate unavoidable fat reads to a subagent.** When a fresh, large region genuinely must be read (no header carries it, no anchor exists yet), spawn an **Explore subagent** to read the pages and return only the `value + citation`. The multi-thousand-token PDF stays in the subagent's context; the main thread gets back ~100 tokens.
5. **Manifest URL, then web.** If there's no local copy, fetch the manifest URL. If there's no manifest entry at all, search the web for the **official / authoritative** document for that exact identifier and version (prefer the vendor's or standards body's own site), then add it to the manifest.
6. **Check for amendments/errata.** Cross-check errata sheets, protocol amendments, or standard corrigenda for the version in use — a value "correct per the base document" may be superseded.
7. **Cite.** State the source inline: document, version/rev, section/page/clause, and the specific register, field, or rule. Citations make the fact verifiable instead of trusted.
8. **Write back to the ledger.** Append every freshly-resolved fact (value + full citation) to `docs/references/facts.md` so the next lookup is a step-0 hit.
9. **Never fabricate.** If a value can't be found in an in-repo source, a local doc, the manifest URL, or an authoritative web source, say so explicitly and ask the user — do not guess a register address, protocol field, timing value, or standard requirement.

## Citing a fact

Make the source checkable, e.g.:

> `USART1->BRR = 0x0683;` — 9600 baud @ 16 MHz PCLK2. Source: RM0090 rev 19, §30.5.4 (USART_BRR); baud formula §30.3.4.

> Standard CAN data frame: 11-bit identifier, DLC 0–8 bytes. Source: CAN 2.0A (Bosch 1991), §3.1.1.

> Avoid implicit conversion that loses sign. Source: MISRA C:2012, Rule 10.3.

## Resolved-facts ledger

`docs/references/facts.md` is the project's cache of facts already looked up — check it first (step 0), append to it last (step 8). It eliminates the dominant cost: re-reading the same datasheet region for a fact resolved an hour ago.

```md
# Resolved facts

| Fact | Value | Source (doc, version, §/page) |
|---|---|---|
| USART1 BRR @ 9600 baud, 16 MHz PCLK2 | `0x0683` | RM0090 rev 19, §30.5.4 |
| Standard CAN data frame ID width | 11 bits, DLC 0–8 | CAN 2.0A (Bosch 1991), §3.1.1 |
| Vdd absolute maximum | 4.0 V | STM32F407 datasheet, §6.2 |
```

Rules:

- **Every row carries a citation.** The ledger is a cache of *verified* facts, not a shortcut around verification — an uncited row is a fabrication waiting to be trusted, so it doesn't belong here.
- **Version-keyed.** A row is only valid for the source version it cites. When a manifest doc's version/revision bumps, rows citing it are suspect — re-verify before reuse.
- It doubles as a **human-reviewable audit trail** of every external fact the firmware depends on.

## Pre-extracting documents (optional)

For a doc you hit repeatedly, convert it to text once — `pdftotext rm0090.pdf docs/references/rm0090.txt` — and check the `.txt` in. Future lookups then `grep` cheaply instead of vision-reading the PDF. Caveats: register/timing **tables** often extract messily (verify against the PDF when a table value is load-bearing), and some standards (ISO 26262, MISRA C) **license-forbid redistribution** — only extract docs you're permitted to store as text. Opt-in per document.

## Recording decisions

A looked-up fact is not a domain term — keep it out of `CONTEXT.md`. But a **constraint that shapes the design** — "ISR latency budget is 10 µs per the peripheral's max interrupt rate", "no DMA on SPI3, see ES0182 §2.x", "no dynamic allocation in the safety path per ISO 26262" — is exactly the "constraint not visible in the code" case for an ADR. Record those via `/domain-modeling`, citing the source.

## Keeping the manifest current

When you resolve a document from the web that wasn't in the manifest, add it (type, identifier, version, URL, and a local path once the team checks in a controlled copy). The manifest is the team's curated set of sources of truth — grow it as you go.
