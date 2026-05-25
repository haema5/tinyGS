## Why

Several boards supported by TinyGS share the same OLED I2C bus (SDA/SCL) and radio SPI pins, making them indistinguishable during `boardDetection()` even though they have physically different hardware. The most prominent example is the T-Beam V1.1 (with AXP192) and T-Beam V1.2 (with AXP2101) versus the TTGO LoRa 32 V2 (no AXP), all of which share SDA=21 / SCL=22. Adding AXP power-management chip detection as an extra probe signal during boot provides a reliable, low-cost discriminator that is already probed later in `checkAXP()` — we just need to use its result earlier.

## What Changes

- Add two new fields to `board_t` and `BoardDef`: `axp_addr` (I2C address, 0 = none) and `axp_chip` (expected chip ID byte: 0x03 = AXP192, 0x4A = AXP2101, 0 = none/unknown).
- Populate those fields for every board in `ConfigStore.cpp` and the test mirror `board_table.h` based on hardware research (see design.md).
- Extend `boardDetection()` Phase 3a (OLED-matching candidates) to probe the AXP address and read the chip-ID register (0x03) on the already-open I2C bus, then filter candidates by matching `axp_chip`.
- Move `checkAXP()` to **not** reinitialize Wire when the AXP bus is already known from detection; reuse detection result to avoid double probing.
- Expose the detected AXP chip type in the web configuration wizard and status display so users can confirm hardware identification.
- Update unit-test `board_table.h` / `test_board_detection.cpp` to cover the AXP-differentiated cases.

## Capabilities

### New Capabilities
- `axp-probe`: Probe AXP power-management chip on the I2C bus and use the chip-ID as a board-identification signal during `boardDetection()`.

### Modified Capabilities
- `board-detection`: The existing board-detection algorithm gains a new AXP-probe sub-step inside Phase 3a (OLED-candidates filtering), improving disambiguation accuracy without breaking existing detection paths.

## Impact

- **`tinyGS/src/Network/ConfigStore.h`** — `board_t` struct gains two fields (`axp_addr`, `axp_chip`); no ABI change for NVS because neither field is stored.
- **`tinyGS/src/Network/ConfigStore.cpp`** — board table entries updated; `boardDetection()` Phase 3a extended.
- **`tinyGS/src/Power/Power.cpp` / `Power.h`** — `checkAXP()` can accept an already-known chip ID to skip redundant I2C probing; exports detected chip to `ConfigStore` for web display.
- **`tinyGS/src/Network/WebServer.cpp`** / **`html.h`** — AXP type shown in wizard status line and board info page.
- **`test/test_board_detection/board_table.h`** — `BoardDef` gains matching fields; test covers AXP-disambiguated boards.
- **`test/test_board_detection/test_board_detection.cpp`** — new test cases.
- No breaking changes to NVS config format, MQTT protocol, or user-visible board numbers.
