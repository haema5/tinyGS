## Context

TinyGS firmware auto-detects hardware at startup via `ConfigStore::boardDetection()`. The algorithm probes in three phases: OLED I2C → Ethernet SPI → radio SPI. Several boards share the same OLED bus (SDA=21, SCL=22, addr=0x3C) and identical radio SPI pin sets — most notably the TTGO LoRa32 V2 family, T-Beam V1.0, and T-Beam V1.1/1.2. They are currently distinguished only by the radio SPI probe, which fails when the radio chip family matches (SX1276 vs SX1276 = same silicon, unresolvable by SPI alone).

After `boardDetection()` completes, `Power::checkAXP()` runs separately and already probes I2C address 0x34, reading register 0x03 to identify the AXP chip. The AXP result is stored in the global `AXPchip` variable but is never fed back into board identification. This change wires that probe into the detection loop.

**AXP assignment per board — ESP32 classic (SDA=21/SCL=22 bus):**

| Board name | AXP chip | axp_chip byte | Notes |
|---|---|---|---|
| TTGO LoRa 32 V2 LF/HF (idx 6, 7) | none | 0x00 | No PMU, simple LDO |
| T-Beam V1.0 LF/HF (idx 14, 18) | **AXP192** | **0x03** | Confirmed AXP192 |
| T-Beam V1.1 LF/HF (idx 8, 9) | **AXP192** | **0x03** | First T-Beam PMU revision |
| T-Beam SX1268 V1.0 (idx 22) | **AXP2101** | **0x4A** | ⚠ Needs HW confirmation — likely AXP2101 (newer PCB) |
| All other bus-B boards (SX126x custom, FOSSA, LILYGO T3) | none | 0x00 | External power or no PMU |
| OLED bus-A boards (Heltec V1/V2, TTGO V1) | none | 0x00 | No PMU |

**AXP assignment per board — ESP32-S3 (SDA=17/SCL=18 bus):**

| Board name | AXP chip | axp_chip byte | Notes |
|---|---|---|---|
| TTGO T-Beam S3 Supreme SX1262 (idx 2) | **AXP2101** | **0x4A** | Confirmed — LILYGO schematic |
| HELTEC LORA32 V3 (idx 0) | none | 0x00 | Built-in TP4054 charger, no AXP |
| All other S3 boards (custom, LILYGO T3S3, EBYTE) | none | 0x00 | No PMU on these modules |

**Disambiguation gain with bidirectional strict-equality filter:**

| Target hardware | Before (radio-only) | After (AXP + radio) |
|---|---|---|
| T-Beam V1.1 433 MHz (idx 8) | COLLISION → detected as idx 6 (TTGO V2) | COLLISION within {8,9,14,18} (AXP192 pool, SX1278 family) → first = **idx 8** ✓ |
| T-Beam V1.1 868 MHz (idx 9) | COLLISION → detected as idx 6 | COLLISION within {8,9,14,18} (AXP192 pool, SX1276≡SX1278) → idx 8 |
| T-Beam V1.0 433 MHz (idx 14) | COLLISION → detected as idx 6 | COLLISION within {8,9,14,18} (AXP192 pool) → idx 8 |
| T-Beam V1.0 868 MHz (idx 18) | COLLISION → detected as idx 6 | COLLISION within {8,9,14,18} (AXP192 pool) → idx 8 |
| T-Beam S3 Supreme (idx 2) | PASS (unique SPI pins NSS=10) | PASS + AXP confirms ✓ |

## Goals / Non-Goals

**Goals:**
- Add `axp_addr` / `axp_chip` fields to `board_t` and test-mirror `BoardDef`.
- Populate those fields based on hardware research for all existing boards.
- Insert an AXP probe step in Phase 3a of `boardDetection()`, using the already-open Wire bus from the OLED probe, to filter OLED-matching candidates before the radio SPI fallback.
- Pass the detected chip ID to `checkAXP()` so it can skip a redundant I2C init/probe.
- Expose the AXP chip name in the web wizard board status line.
- Add unit-test coverage for the AXP-disambiguated boards.

**Non-Goals:**
- Changing AXP power-rail configuration logic inside `checkAXP()`.
- Adding AXP probing for boards without OLED (Ethernet and radio-only boards already have no AXP).
- Supporting AXP chips beyond AXP192 and AXP2101 (those are the only chips present on supported boards).
- Probing AXP when no OLED is found (AXP chips always share the OLED bus in supported hardware).

## Decisions

### D1 — Two new fields: `axp_addr` and `axp_chip` (not a single enum)

**Chosen:** Two separate `uint8_t` fields — `axp_addr` (I2C address, 0 = none) and `axp_chip` (chip-ID register value at reg 0x03: 0x03 = AXP192, 0x4A = AXP2101, 0x00 = no AXP fitted).

**Rationale:** `axp_addr` is needed at probe time (to issue the Wire.beginTransmission), and `axp_chip` is needed for comparison. Keeping them separate avoids hard-coding address 0x34 in the detection loop and allows future chips at different addresses. All currently-known AXP chips used in TinyGS hardware live at address 0x34, so in practice `axp_addr` will always be 0x34 or 0.

**Alternative considered:** Single enum `{NONE, AXP192, AXP2101}` with addresses encoded internally — rejected because it hides the address and complicates future extension.

### D2 — Probe position: inside Phase 3a, before radio SPI, with strict-equality filtering

**Chosen:** After the OLED bus is confirmed and the candidate list is built, probe AXP _once_ on the confirmed bus. Compute `detectedAxpChip`. Apply bidirectional strict-equality filter: keep only candidates where `candidate.axp_chip == detectedAxpChip`. The filtered list then enters the existing radio SPI disambiguation step unchanged.

**Rationale:** AXP is on the same Wire bus already open from the OLED probe (~1 ms overhead). Bidirectional filtering eliminates boards on both sides: when AXP192 is found, TTGO LoRa 32 V2 (no AXP) is eliminated; when no AXP is found, T-Beam V1.1 (AXP192) is eliminated. This turns previously-unresolvable SX127X collisions into direct unique detections in several cases (e.g. T-Beam V1.1 LF on bus B).

**Alternative considered:** A dedicated Phase 1b before radio — adds unnecessary code duplication when the bus is already confirmed by Phase 1.

### D3 — `axp_chip = 0` means "no AXP" and filtering is bidirectional (strict equality)

**Chosen:** `axp_chip = 0` means the board has no AXP chip. Filtering uses strict equality: only candidates whose `axp_chip` value matches `detectedAxpChip` survive, in both directions:
- AXP found (e.g. 0x03) → candidates declaring no AXP (`axp_chip = 0`) or a different AXP variant (e.g. 0x4A) are eliminated.
- No AXP found (0x00) → candidates declaring any AXP (`axp_chip != 0`) are eliminated.

**Rationale:** This is the only rule that actually resolves collisions. With a "don't care" rule, T-Beam V1.1 (AXP192) and TTGO LoRa 32 V2 (no AXP) would remain in the candidate set together after the AXP probe, gaining nothing. Strict equality also enforces AXP192 vs AXP2101 differentiation: a board with AXP2101 (0x4A) is eliminated when AXP192 (0x03) is detected and vice versa.

Boards with genuinely unknown AXP status (hardware not yet verified) MUST NOT be added to the board table until their AXP state is confirmed — they should remain with their current table entry and no `axp_chip` field until researched.

**Alternative considered:** "0 = don't care" — rejected because it eliminates zero collisions; the TTGO V2 (axp_chip=0) would survive alongside the T-Beam V1.1 (axp_chip=0x03) even when AXP192 is detected, defeating the purpose of the probe.

### D4 — `checkAXP()` accepts an optional pre-detected chip ID

**Chosen:** Add an overload (or default parameter) `checkAXP(uint8_t knownChipID = 0)`. When `knownChipID != 0`, `checkAXP()` skips the Wire.begin + I2C probe and proceeds directly to the configuration step. `boardDetection()` calls it with the ID it already read; the existing call site in `tinyGS.ino` continues to pass 0 (probe from scratch) as a safe fallback.

**Rationale:** Avoids double probing and a second Wire.begin/end cycle. No logic change needed at the existing call site.

### D5 — Store detected AXP chip ID in `Power` module, expose via getter

**Chosen:** `AXPchip` global (already in Power.cpp) is set by `checkAXP()` as before. Add a `Power::getAXPchip()` getter (or expose the global via header) so `WebServer` can display the chip name.

## Risks / Trade-offs

| Risk | Mitigation |
|---|---|
| AXP probe on boards without AXP generates a NACK (benign but logged) | IDF i2c.master log level is already suppressed to ESP_LOG_NONE during the OLED probe window; extend the suppressed window to cover the AXP probe. |
| A board with `axp_chip=0` (no AXP declared) is physically running with an AXP fitted on a revision not in the table | Per D3, such a board would be falsely eliminated when an AXP is found. Mitigation: only declare `axp_chip=0` for boards whose *entire production run* is verified as having no AXP. Any board with revision-dependent AXP must be split into separate table entries (one per revision). |
| T-Beam SX1268 V1.0 AXP state unknown | Until hardware is confirmed, leave `axp_chip=0` for idx 22 and add a prominent TODO comment. This board continues to use the existing radio-collision fallback. Once confirmed, update to 0x03 or 0x4A accordingly. |
| Wire bus left open between OLED probe and AXP probe | OLED probe already does Wire.begin(); AXP probe reuses the same Wire instance without re-initializing. Wire.end() is called once, after both probes are done. This matches existing behavior in `checkAXP()` which also does begin/end. |
| T-Beam V1.0 and V1.1 all carry AXP192 — they remain in the same collision pool after AXP filter | AXP does not disambiguate V1.0 from V1.1; the four boards {idx 8, 9, 14, 18} all have AXP192 and two radio families (SX1278 and SX1276, same silicon). The SX1278 pool {8, 14} and SX1276 pool {9, 18} are resolved by first-wins within each sub-pool. T-Beam V1.0 boards will always be detected as idx 8 (V1.1 LF) or idx 9 (V1.1 HF) since V1.1 appears first in the table. This is a known, accepted limitation — distinguishing V1.0 from V1.1 would require a PROG-pin or hardware-version register probe not currently supported. |

## Migration Plan

1. Add fields to `board_t` with default value 0 (backward-compatible struct extension).
2. Populate board table entries in `ConfigStore::initBoardTable()`.
3. Extend `boardDetection()` Phase 3a.
4. Update `checkAXP()` signature.
5. Update `WebServer` status display.
6. Update test `board_table.h` and add test cases.
7. Rebuild and run native unit tests: `pio test -e native_esp32`.

Rollback: fields are zero-initialized by default, so reverting the detection change still results in correct behavior for all boards (Phase 3a simply won't filter by AXP and falls through to radio probe as before).

## Open Questions

- **T-Beam SX1268 V1.0** (TBEAM_SX1268_TCXO, idx 22 on ESP32 classic): does production hardware ship with AXP192 (0x03) or AXP2101 (0x4A)? The board is a newer PCB revision (likely AXP2101 based on LILYGO's migration pattern) but needs hardware confirmation. Until then: `axp_chip = 0` with TODO comment, existing radio-fallback behavior preserved.
- **T-Beam V1.0 vs V1.1 disambiguation**: Both revisions carry AXP192, so AXP detection alone cannot tell them apart. If this distinction becomes important (e.g. for power-rail or PROG-pin differences), a separate discriminant (e.g. GPIO level check on a version-indicator pin) would be needed. For now, T-Beam V1.0 hardware will be identified as T-Beam V1.1 (first in pool), which is acceptable since their radio and power configuration is equivalent in TinyGS.
