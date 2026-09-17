# NFC/RFID driver architecture — reference

Verified against the `private_v4` tree. Re-grep symbol names if line numbers
have drifted.

## 1. Drivers and the factory

Four chip drivers, `extras/mmu/unit/nfc/`:

| Driver | Lines | Notes |
|---|---|---|
| `pn532_driver.py` | ~1988 | `_PN532Base`, shared across transports |
| `pn532_uart_driver.py` | ~739 | HSU/UART transport for PN532 |
| `pn5180_driver.py` | ~869 | no non-blocking probe (see below) |
| `pn7160_driver.py` | ~1848 | needs `irq_pin` for full-rate probing |
| `rc522_driver.py` | ~1359 | |

`reader_factory.py`:
- `SUPPORTED_READER_TYPES = ('pn532', 'pn5180', 'pn7160', 'rc522')`, default
  `pn532` (`:21-22`).
- `SUPPORTED_INTERFACES` maps each chip to the transports it has a *driver*
  for (`:29-34`): `pn532`→(i2c,spi,uart), `pn5180`→(spi,), `pn7160`→(i2c,),
  `rc522`→(spi,). This reflects driver coverage, not silicon capability.
- `create_reader()` (`:300-381`) is the single dispatch point building a
  concrete driver from a `[mmu_nfc_reader NAME]` config section.
- Driver contract (`:6-10`): `init()`, `is_alive()`, `read_tag()`,
  `read_target()`, optional `probe_start/probe_poll/probe_stop`.

**Maturity caveats, enforced in code, not just comments:**
- PN532-over-SPI logs "UNTESTED against real hardware" on every build
  (`reader_factory.py:320-329`).
- PN5180 has no presence-probe implementation — falls back to the blocking
  shim during homing, deliberately (`pn5180_driver.py:771-780`).
- PN7160 only gets full-rate non-blocking probing with a wired `irq_pin`;
  without it, same blocking-shim fallback.

`MmuNfcReader` (`mmu_nfc_reader.py`, ~785 lines) is the per-instance facade
above drivers. Its own docstring (`:11-12`) states it excludes lane state
machines, Spoolman lookups, LEDs, and scan-jog motion by design — those live
in `mmu_nfc_manager.py` and `mmu_filament_movement.py`.

## 2. jog_scan

Config: `nfc_gate_jog_scan_window`, `nfc_preload_jog_scan_window` — each a
`(neg, pos)` floatlist, default `[0.0, 0.0]` (off unless configured)
(`mmu_unit_parameters.py:189-190`).

Triggers: `MMU_NFC_SCAN` (`commands/mmu_nfc_scan.py:110` → `mmu._jog_scan()`),
and automatically during `MMU_PRELOAD` on a miss (via
`_home_to_gate_with_nfc`, `mmu_filament_movement.py:529`).

Implementation: orchestrator `_jog_scan()` (`mmu_filament_movement.py:632`)
delegates the actual motion to `_scan_sweeps()` (`:849`, sweeps the
larger-magnitude window side first since a hit short-circuits the rest) and
`_scan_datum_leg`/`_scan_leg` beneath it.

Why it exists: fast-path comment at `mmu_filament_movement.py:709-716` — the
tag may already be sitting on the reader (no jog needed); when it isn't, the
filament is jogged within the window, homing against the reader as a virtual
endstop (`MmuNfcEndstop`, `unit/nfc/mmu_nfc_endstop.py`), until the tag
enters the RF coupling volume, then re-parked exactly once off a real gate
datum — the `_jog_scan` docstring (`:640-653`) explains why the re-park must
happen once, off a datum, to avoid walking the filament backward on repeated
invocations.

## 3. Reader-pair sharing (shipped, `52737f53`)

User-facing config (`config/base/mmu_hardware.cfg:196-205`): with
`MMU_HAS_PER_GATE_NFC_READERS` set, list one `nfc_readers` name per gate;
**repeating a name shares one physical reader between neighbouring gates**:

```
nfc_readers: a, a, b, b   # reader 'a' serves gates 0/1, reader 'b' serves gates 2/3
```

(matching explanation: `extras/mmu/mmu_unit.py:216-222`). In-tree example:
ViViD board, `installer/boards/custom/Kconfig.vvd:119-133` (two `rc522`
readers each serving a gate pair).

What `mmu_nfc_manager.py` does differently for a shared-pair reader:
- `_lookup_or_create_reader()` (`:154-167`) looks up `[mmu_nfc_reader NAME]`
  as a Klipper printer object first; a second gate naming the same section
  gets the *same* object back (`:158-160`).
- `_unique_readers()` (`:558-580`) dedupes `gate_readers` by object identity
  into `[(reader, [global_gate,...])]`. `_init_all_readers()` (`:582-596`)
  iterates *this*, not `gate_readers` directly — so a paired reader's
  hardware `init()` runs exactly once. **This was the actual bug `52737f53`
  fixed** — before it, a shared chip was re-initialized once per gate slot.
- Two distinct `MmuNfcEndstop` objects still exist, one per gate
  (`:143-151`), both wrapping the one reader — homing moves on the two
  paired gates are strictly serialized by the shared hardware.
- `enabled`/`active` flags stay **independent per gate slot** — plain
  arrays indexed by local-gate index (`self.gate_enabled[lg]`,
  `self.gate_active[lg]`, `:298-339`), not per-reader-object. Disabling gate
  0's reader does not disable gate 1's, even though it's the same chip.
- Bootup reader count (`_handle_mmu_bootup`, `:463`) is computed off the
  deduped set, so it logs "N readers" correctly rather than double-counting
  a shared pair.

**What it does not do:** no arbitration over whose tag is currently in the
shared reader's field between the two paired gates — the manager trusts
whichever tag the chip reports. This is a *different* gap from the
crosstalk problem below (this is one chip genuinely shared by two gates by
config; crosstalk is two *separate* chips picking up each other's field) —
see [noisy-neighbour-unmerged.md](noisy-neighbour-unmerged.md).

## 4. Fixed bug: `Kconfig.vvd` hardcoded `unit1_nfc01/23` (not templated)

`"MMU NFC reader section [mmu_nfc_reader NAME] not found!"` is raised from
two call sites — `mmu_unit.py:238` (validated at config-load time, over
every name in `nfc_reader` + `nfc_readers`) and `mmu_nfc_manager.py:167`
(same check again at reader-lookup time). Both just mean: some name in
`nfc_reader`/`nfc_readers` has no matching `[mmu_nfc_reader NAME]` section
anywhere in the merged config — a name/section mismatch, not a missing
feature.

**Root cause found and fixed 2026-09-18** in
`installer/boards/custom/Kconfig.vvd` (the BTT ViViD board preset). Its
`PARAM_MISC_HARDWARE` default is a hand-written config block (used because
`config/base/mmu_hardware.cfg`'s auto-generated `[mmu_nfc_reader ...]`
sections are gated off by `CUSTOM_NFC_READER_SETUP`, which this board turns
on to hand-wire its two `rc522`s on a shared SPI bus, `installer/Kconfig.
capabilities:113`). Every other line in that block correctly uses the
`$(UNIT_NAME)` macro (`neopixel $(UNIT_NAME)_leds_1`, `heater_generic
$(UNIT_NAME)_heater`, …) — but the two `[mmu_nfc_reader ...]` section
headers had the literal string `unit1_nfc01`/`unit1_nfc23` hardcoded
instead of `$(UNIT_NAME)_nfc01`/`$(UNIT_NAME)_nfc23` (was
`Kconfig.vvd:120,128`). Meanwhile the *names* referencing those sections —
`PARAM_NFC_READER_0..3` in `installer/mmu_types/Kconfig.vvd:133-140` —
correctly derive from `$(UNIT_NAME)_nfc01`/`_nfc23`. So for any printer
where ViViD is `unit0`, the reader names resolved to `unit0_nfc01/23` while
the hand-written sections stayed `unit1_nfc01/23` — guaranteed mismatch,
100% reproducible, nothing to do with the user's own config.

**Why it shipped unnoticed:** the only test profile exercising this board,
`ERCF_VVD` (`test/hh/profiles.py:536-625`), is "a transcription of a real
machine" where ViViD genuinely *is* `unit1` (paired with ERCF as `unit0`,
`test/hh/profiles.py:492-497`). `$(UNIT_NAME)` resolves to `"unit1"` there,
so the hardcoded literal happened to match by coincidence and every
existing assertion (`test_mmu_nfc.py:392-395`,
`test_mmu_profiles.py:676`) passed either way. **No test profile installs
ViViD as a standalone/first unit (`unit0`)** — that's the untested
configuration the bug actually lived in. If you add one, expect
`nfc_readers` to read `unit0_nfc01, unit0_nfc01, unit0_nfc23, unit0_nfc23`
and the `[mmu_nfc_reader ...]` section names to now match it post-fix.

**The fix**: `Kconfig.vvd:120,128` now read `[mmu_nfc_reader
$(UNIT_NAME)_nfc01]` / `$(UNIT_NAME)_nfc23]`, matching every other line in
the block. Verified with `make UT='test_mmu_nfc*.py' test` and `make
UT='test_mmu_profiles.py' test` (both pass unchanged, since `$(UNIT_NAME)`
still resolves to `unit1` for the one profile that exercises this board).

When debugging this error in general (not just on ViViD), always diff the
*resolved* `nfc_reader`/`nfc_readers` value against the actual
`[mmu_nfc_reader NAME]` section names in the merged config — that's where a
divergence like this actually lives, whether it's a template bug or a
user's hand-edited `CUSTOM_NFC_READER_SETUP` block drifting out of sync.
