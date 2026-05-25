## 1. Data Model — Add AXP fields to board descriptors

- [x] 1.1 Add `uint8_t axp_addr` and `uint8_t axp_chip` fields (default 0) to `board_t` struct in `ConfigStore.h`
- [x] 1.2 Add matching `axp_addr` and `axp_chip` fields to `BoardDef` struct in `test/test_board_detection/board_table.h`
- [x] 1.3 Add named constants for AXP chip IDs to a shared header (e.g. `AXP192_CHIP_ID = 0x03`, `AXP2101_CHIP_ID = 0x4A`) where both `ConfigStore.cpp` and `Power.cpp` can use them (or reuse existing `XPOWERS_AXP192_CHIP_ID` / `XPOWERS_AXP2101_CHIP_ID` already referenced in `Power.cpp`)

## 2. Board Table — Populate AXP fields for all boards

Board-by-board AXP assignments (confirmed from schematics / LILYGO documentation):

**ESP32 classic — boards with AXP:**
- [x] 2.1 `"433MHz T-BEAM + OLED"` (TBEAM_OLED_LF, idx 8): `axp_addr=0x34, axp_chip=0x03` (AXP192)
- [x] 2.2 `"868-915MHz T-BEAM + OLED"` (TBEAM_OLED_HF, idx 9): `axp_addr=0x34, axp_chip=0x03` (AXP192)
- [x] 2.3 `"433 Mhz T-Beam SX1268 V1.0"` (TBEAM_SX1268_TCXO, idx 22): ⚠️ **TODO — HW confirmation needed.** Likely `axp_addr=0x34, axp_chip=0x4A` (AXP2101) based on LILYGO's migration to AXP2101 on newer T-Beam PCBs. Until confirmed: leave `axp_addr=0, axp_chip=0x00` and add a `// TODO: verify AXP2101 on T-Beam SX1268 V1.0` comment.

- [x] 2.4 `"433MHz T-BEAM V1.0 + OLED"` (TBEAM_OLED_v1_0_LF, idx 14): `axp_addr=0x34, axp_chip=0x03` (AXP192)
- [x] 2.5 `"868-915MHz T-BEAM V1.0 + OLED"` (TBEAM_OLED_v1_0_HF, idx 18): `axp_addr=0x34, axp_chip=0x03` (AXP192)

**ESP32 classic — boards without AXP (axp_addr=0, axp_chip=0x00):**
- [x] 2.6 All remaining ESP32 classic boards: TTGO LoRa 32 V2 (idx 6, 7), all bus-A OLED boards (Heltec V1/V2, TTGO V1, idx 0-5), custom SX126x boards (idx 10-13), FOSSA boards, LILYGO T3 boards, WT32-ETH01 boards — all remain `axp_addr=0, axp_chip=0x00`

**ESP32-S3 — boards with AXP:**
- [x] 2.7 `"433 Mhz TTGO T-Beam Sup SX1262 V1.0"` (TTGO_TBEAM_SX1262, idx 2): `axp_addr=0x34, axp_chip=0x4A` (AXP2101)

**ESP32-S3 — boards without AXP (axp_addr=0, axp_chip=0x00):**
- [x] 2.8 All remaining ESP32-S3 boards: Heltec LORA32 V3 (idx 0), Custom S3 SX1278/SX1268/SX1262/LR2021 (idx 1, 4-7), LILYGO T3S3 (idx 3), Waveshare ETH S3 — all remain `axp_addr=0, axp_chip=0x00`

- [x] 2.9 Mirror all of the above assignments to the matching entries in `test/test_board_detection/board_table.h` (esp32 and esp32s3 namespaces)


## 3. Detection Logic — Extend boardDetection() Phase 3a

- [x] 3.1 After the OLED bus is confirmed and the log message emitted, probe I2C address 0x34 reg 0x03 on the confirmed bus (Wire is still initialized from the OLED probe); store result as `uint8_t detectedAxpChip`
- [x] 3.2 Suppress `i2c.master` IDF logs around the AXP probe (extend the existing `esp_log_level_set` window or add a new narrow window)
- [x] 3.3 In the candidate-filtering loop (Phase 3a), replace the comment-only stub with the bidirectional strict-equality check: `if (_boards[ite].axp_chip != detectedAxpChip) continue;` — this eliminates both AXP-declaring boards when no AXP is found **and** no-AXP boards when an AXP is found
- [x] 3.4 Log the AXP probe result (found: chip ID hex, or "no AXP") to `LOG_CONSOLE` for diagnostics
- [x] 3.5 Pass `detectedAxpChip` to `power.checkAXP()` after detection completes (or store it so `checkAXP()` can retrieve it)

## 4. Power Module — Avoid redundant I2C probe

- [x] 4.1 Add parameter `uint8_t knownChipID = 0` to `Power::checkAXP()` declaration in `Power.h`
- [x] 4.2 In `Power::checkAXP()`: if `knownChipID != 0`, skip `Wire.begin()` + I2C probe and use `knownChipID` as `ChipID` directly; document this shortcut with a comment
- [x] 4.3 Update the call site in `tinyGS.ino` to pass the detected AXP chip ID from `boardDetection()` (requires a getter or small refactor to make the detected value available at call time); leave the fallback `checkAXP(0)` path intact for cases where detection didn't run

## 5. Web UI — Display AXP chip in wizard

- [x] 5.1 Expose `AXPchip` value (already set in Power.cpp) to the web server, either via a getter in `Power.h` or by reading the global where accessible
- [x] 5.2 In `WebServer.cpp` (or `html.h`), add a board-info line that prints "AXP192" when `AXPchip == 1`, "AXP2101" when `AXPchip == 2`, and nothing (or omit the row) when `AXPchip == 0`

## 6. Unit Tests — Cover AXP-disambiguated scenarios

### 6.1 — board_table.h: extend BoardDef and detect_board() for AXP

- [x] 6.1.1 Add `uint8_t axp_chip` field to `struct BoardDef` in `test/test_board_detection/board_table.h`
- [x] 6.1.2 Add `axp_matches()` predicate method (or inline in detect_board): returns `candidate.axp_chip == target.axp_chip` — strict equality, no "don't care"
- [x] 6.1.3 In `detect_board()`: after the OLED-bus filter and before the radio filter, apply the AXP filter. Keep only candidates where `board.axp_chip == target.axp_chip`. This mirrors the bidirectional logic in `boardDetection()`
- [x] 6.1.4 Add a `print_board_table()` note to print the `axp_chip` value alongside existing fields (for diagnostics when a test fails)

### 6.2 — test_esp32.cpp: existing collision boards that change outcome

The following boards previously collided on bus B (SDA=21/SCL=22). With AXP, outcomes change:

| Board | idx | Old outcome | New expected outcome | Reason |
|---|---|---|---|---|
| T-Beam V1.1 433MHz | 8 | COLLISION → detected as idx 6 | **COLLISION → detected as idx 8** | AXP192 pool = {8,9,14,18}; SX1278 family first-wins → idx 8 ✓ |
| T-Beam V1.1 868MHz | 9 | COLLISION → detected as idx 6 | **COLLISION → detected as idx 8** | AXP192 pool = {8,9,14,18}; SX1276≡SX1278, first wins → idx 8 |
| T-Beam V1.0 433MHz | 14 | COLLISION → detected as idx 6 | **COLLISION → detected as idx 8** | AXP192 pool = {8,9,14,18}; V1.1 appears first → idx 8 |
| T-Beam V1.0 868MHz | 18 | COLLISION → detected as idx 6 | **COLLISION → detected as idx 8** | Same AXP192 pool; V1.1 first → idx 8 |

- [x] 6.2.1 Update `check_board()` or add a new `check_board_axp()` helper that accepts expected outcome (idx + whether it is "direct" or "collision fallback") and the AXP chip value of the target board
- [x] 6.2.2 Test: board idx 8 (T-Beam V1.1 LF) with `axp_chip=0x03` → detected as idx 8 (AXP192 pool = {8,9,14,18}; SX1278 family, first-wins; same board → correct)
- [x] 6.2.3 Test: board idx 9 (T-Beam V1.1 HF) with `axp_chip=0x03` → detected as idx 8 (SX1276≡SX1278; first in AXP192 pool; document as accepted collision)
- [x] 6.2.4 Test: board idx 14 (T-Beam V1.0 LF) with `axp_chip=0x03` → detected as idx 8 (AXP192 pool; V1.1 appears before V1.0 in table; document as accepted V1.0→V1.1 collision)
- [x] 6.2.5 Test: board idx 18 (T-Beam V1.0 HF) with `axp_chip=0x03` → detected as idx 8 (same AXP192 pool; accepted collision)
- [x] 6.2.6 Test: TTGO LoRa 32 V2 (idx 6) with `axp_chip=0x00` → detected as idx 6 directly (no-AXP pool, first match; AXP filter eliminates all T-Beam boards)

### 6.3 — test_esp32s3.cpp: T-Beam S3 Supreme AXP confirmation

- [x] 6.3.1 Test: board idx 2 with `axp_chip=0x4A` → detected as idx 2 directly (was already unique by radio pin NSS=10; AXP2101 now provides a second discriminant — test that it still passes and add a comment noting both discriminants)
- [x] 6.3.2 Negative test: simulate idx 0 (Heltec LORA32 V3, `axp_chip=0x00`) with AXP probe returning 0x4A → idx 2 (T-Beam S3 Supreme) SHALL NOT be selected; this simulates a Heltec board with a spurious 0x34 ACK — the filter eliminates it since `axp_chip(candidate=0) != detectedAxpChip(0x4A)` ... wait: a Heltec with `axp_chip=0x00` target, AXP probe returns 0x4A → filter keeps only candidates with `axp_chip=0x4A` → only idx 2 remains, but target is idx 0. This confirms the limit of the approach: if an external I2C device at 0x34 is present, it can cause misdetection. Document as a known corner-case limit in the test.

### 6.4 — Sanity tests for board table consistency

- [x] 6.4.1 Test: for every board in the ESP32 namespace where `axp_chip != 0`, verify `axp_addr == 0x34` (no board should declare an AXP chip but leave the address at 0)
- [x] 6.4.2 Test: for every board in the ESP32-S3 namespace where `axp_chip != 0`, verify `axp_addr == 0x34`
- [x] 6.4.3 Test: for every board in the ESP32-C3 and ESP32-C6 namespaces, all entries have `axp_chip == 0` and `axp_addr == 0` (these SoCs have no known T-Beam or AXP-fitted board variant in TinyGS)

### 6.5 — Run all test suites

- [x] 6.5.1 `pio test -e native_esp32` — all tests pass including updated collision expectations
- [x] 6.5.2 `pio test -e native_esp32s3` — T-Beam S3 Supreme still detected, AXP annotation present
- [x] 6.5.3 `pio test -e native_esp32c3` — existing tests unchanged (no AXP boards)


## 7. Verification

- [x] 7.1 Build firmware for ESP32 classic target and confirm no compiler errors or warnings related to new fields
- [x] 7.2 Build firmware for ESP32-S3 target and confirm clean build
- [x] 7.3 On physical T-Beam V1.1 hardware: confirm serial log shows "AXP192 found" and correct board is selected during boot
- [x] 7.4 On physical T-Beam V1.2 hardware (if available): confirm "AXP2101 found" and correct board selected
- [x] 7.5 On a board without AXP (e.g. TTGO LoRa 32 V2): confirm no spurious AXP log and correct board still detected
