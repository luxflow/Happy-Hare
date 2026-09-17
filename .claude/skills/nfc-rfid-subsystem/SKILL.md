---
name: nfc-rfid-subsystem
description: Explains Happy Hare's NFC/RFID subsystem — the driver abstraction for pn532/pn5180/pn7160/rc522 readers, jog_scan mechanics, and shipped reader-pair sharing — plus a clearly-flagged PROPOSED, UNMERGED, untested-on-hardware design for mitigating RF-crosstalk between neighbouring gate readers (a "noisy neighbour" classification ladder and eviction-by-jogging) that a future contributor may need to resume rather than re-invent. Use this whenever touching NFC/RFID reader code, mmu_nfc_manager.py, reader_factory.py, jog_scan, Spoolman tag lookups or auto-create, nfc_readers config, or debugging cross-gate tag misattribution — even for something as simple as "the NFC reader isn't detecting the right spool" or "how do I wire two gates to one reader."
---

# NFC/RFID subsystem

Two things are true about this subsystem that aren't obvious from the code
alone:

1. There are **two different sharing problems** that sound similar and are
   easy to conflate: reader-*pairing* (one physical chip serving two gates,
   shipped and working) vs. RF-*crosstalk* (a neighbour's tag showing up in
   your reader's field even though each gate has its own chip, proposed but
   never merged). If you're debugging a misattributed tag, work out which
   one you're actually looking at before reaching for either fix.
2. A chunk of the design work for the crosstalk problem exists only in a
   sibling local checkout's git history — it never got a PR, was never
   pushed anywhere else, and only survives in one commit message and this
   skill. If you're picking this up, read
   [references/noisy-neighbour-unmerged.md](references/noisy-neighbour-unmerged.md)
   before writing new code — it's likely faster to resume than redesign.

## Current architecture (shipped)

Four chip drivers under `extras/mmu/unit/nfc/` (`pn532_driver.py`,
`pn532_uart_driver.py`, `pn5180_driver.py`, `pn7160_driver.py`,
`rc522_driver.py`), dispatched by `reader_factory.py`'s `create_reader()`.
Drivers share a contract (`init()`, `is_alive()`, `read_tag()`,
`read_target()`, optional `probe_start/probe_poll/probe_stop` for
non-blocking homing). Maturity isn't uniform — PN532-over-SPI logs
"UNTESTED against real hardware" on every build, PN5180 has no
non-blocking probe at all, PN7160 needs a wired `irq_pin` for full-rate
probing. Check the factory's own warnings before assuming a driver is as
solid as another.

`MmuNfcReader` (`extras/mmu/unit/nfc/mmu_nfc_reader.py`) is the per-instance
facade above the drivers; it deliberately excludes lane state machines,
Spoolman lookups, LEDs, and scan-jog motion — those live in
`mmu_nfc_manager.py` and `mmu_filament_movement.py`. Don't add them to the
reader facade; that separation is intentional.

**`jog_scan`**: a tag's position relative to the fixed reader isn't
guaranteed, so `_jog_scan()` (`mmu_filament_movement.py`) first checks for a
tag already in range (no motion needed), then jogs the filament within a
configurable `(neg, pos)` mm window, homing against the reader as a virtual
endstop, until the tag enters the RF coupling volume — then re-parks exactly
once off a real gate datum. Triggered by `MMU_NFC_SCAN` directly, and
automatically by `MMU_PRELOAD` on a miss.

**Reader-pair sharing** (shipped, `52737f53`): repeating a reader name in
`nfc_readers:` (e.g. `nfc_readers: a, a, b, b`) shares one physical chip
between two gates. `mmu_nfc_manager.py` looks up the printer object by name,
so both gates get the same Python object; hardware `init()` runs once per
*deduped* reader, not once per gate slot (`_unique_readers()`); `enabled`/
`active` state stays independent per gate slot even though the chip is
shared. There is **no arbitration** over whose tag is in the field between
the two paired gates today — the manager trusts whatever the chip reports.
That gap is exactly what the unmerged crosstalk design addresses, for the
different (separate-reader) case — see the reference doc.

**`"MMU NFC reader section [...] not found!"`** almost always means a
name/section mismatch, not a missing feature. §4 of the reference doc
documents a real, now-fixed instance: `installer/boards/custom/Kconfig.vvd`
hardcoded `unit1_nfc01`/`unit1_nfc23` in its `PARAM_MISC_HARDWARE` default
instead of using the `$(UNIT_NAME)` macro like every surrounding line —
guaranteed to break for any printer running ViViD as `unit0` (the shipped
test profile only ever installs ViViD as `unit1`, which is why it went
unnoticed). Fixed 2026-09-18. If this error resurfaces elsewhere, the same
diagnostic applies: diff the resolved `nfc_reader`/`nfc_readers` names
against the actual `[mmu_nfc_reader NAME]` section names in the merged
config.

Full file:line citations for all of the above are in
[references/driver-architecture.md](references/driver-architecture.md).

## Unmerged design: RF-crosstalk / "noisy neighbour" mitigation

Per-gate readers can sit close enough that a neighbouring gate's spool is
inside gate G's own RF field — `_jog_scan`'s fast path and the preload
compound endstop both trusted "a tag is at my reader" to mean "this gate's
tag," which is only sometimes true. A classification ladder
(CLEAR/MINE/NEIGHBOUR/FOREIGN) plus eviction-by-jogging was designed and
partially implemented to fix this — **it never made it into `private_v4`**.
It exists as one commit on a branch in a sibling local checkout, off by
default, with an explicitly-noted test gap (the eviction *motion* itself
can't be exercised by the harness). Read
[references/noisy-neighbour-unmerged.md](references/noisy-neighbour-unmerged.md)
for the full design, the exact commit to pull from, and the caveats that
matter most if you resume it.
