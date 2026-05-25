## ADDED Requirements

### Requirement: Board descriptor declares AXP chip variant
Every entry in the board table (`board_t` in ConfigStore and `BoardDef` in the test mirror) SHALL include two fields: `axp_addr` (I2C address of the AXP chip, 0 = not present) and `axp_chip` (chip-ID register value: 0x03 = AXP192, 0x4A = AXP2101, 0x00 = no AXP fitted). The value 0x00 is exclusive — it means the board definitively has no AXP, not "unknown". Boards with unconfirmed AXP status SHALL NOT be assigned a non-zero `axp_chip` until hardware is verified.

#### Scenario: AXP192 board declared
- **WHEN** a board entry is for hardware fitted with AXP192 (e.g. T-Beam V1.1 LF or HF on ESP32 classic)
- **THEN** `axp_addr` SHALL be 0x34 and `axp_chip` SHALL be 0x03 (XPOWERS_AXP192_CHIP_ID)

#### Scenario: AXP2101 board declared
- **WHEN** a board entry is for hardware fitted with AXP2101 (e.g. T-Beam S3 Supreme on ESP32-S3)
- **THEN** `axp_addr` SHALL be 0x34 and `axp_chip` SHALL be 0x4A (XPOWERS_AXP2101_CHIP_ID)

#### Scenario: Board without AXP declared
- **WHEN** a board entry is for hardware verified as having no AXP chip (e.g. TTGO LoRa 32 V2, Heltec boards, T-Beam V1.0)
- **THEN** `axp_addr` SHALL be 0x00 and `axp_chip` SHALL be 0x00

### Requirement: AXP chip is probed on the OLED I2C bus during board detection
After confirming the OLED bus (Phase 1 of `boardDetection()`), the firmware SHALL probe address 0x34 on the confirmed I2C bus and read register 0x03 to obtain a chip-ID byte. The probe SHALL reuse the Wire instance already initialized for the OLED probe without calling `Wire.begin()` again.

#### Scenario: AXP192 detected on open I2C bus
- **WHEN** the confirmed OLED bus has address 0x34 ACKing
- **AND** register 0x03 returns 0x03
- **THEN** `detectedAxpChip` SHALL be set to 0x03

#### Scenario: AXP2101 detected on open I2C bus
- **WHEN** register 0x03 at address 0x34 returns 0x4A
- **THEN** `detectedAxpChip` SHALL be set to 0x4A

#### Scenario: No AXP on I2C bus
- **WHEN** address 0x34 does not ACK on the confirmed OLED bus
- **THEN** `detectedAxpChip` SHALL be 0x00

### Requirement: AXP probe result is applied as bidirectional strict-equality filter
After the AXP probe, the list of OLED-matching board candidates SHALL be filtered by strict equality: only candidates where `candidate.axp_chip == detectedAxpChip` SHALL remain. This applies in both directions — boards declaring no AXP are eliminated when one is found, and boards declaring an AXP are eliminated when none is found.

#### Scenario: AXP192 detected — no-AXP candidates eliminated
- **WHEN** `detectedAxpChip` == 0x03
- **AND** the OLED-matching candidate list includes boards with `axp_chip == 0x03` and boards with `axp_chip == 0x00`
- **THEN** boards with `axp_chip == 0x00` SHALL be eliminated
- **AND** only boards with `axp_chip == 0x03` SHALL remain

#### Scenario: AXP192 detected — AXP2101 candidates eliminated
- **WHEN** `detectedAxpChip` == 0x03
- **AND** a candidate has `axp_chip == 0x4A`
- **THEN** that candidate SHALL be eliminated

#### Scenario: AXP2101 detected — AXP192 candidates eliminated
- **WHEN** `detectedAxpChip` == 0x4A
- **AND** a candidate has `axp_chip == 0x03`
- **THEN** that candidate SHALL be eliminated

#### Scenario: No AXP detected — AXP-declaring candidates eliminated
- **WHEN** `detectedAxpChip` == 0x00
- **AND** candidates include T-Beam V1.0 (idx 14/18, `axp_chip == 0x03`), T-Beam V1.1 (idx 8/9, `axp_chip == 0x03`), T-Beam S3 Supreme (`axp_chip == 0x4A`), or any other board with `axp_chip != 0x00`
- **THEN** all such candidates SHALL be eliminated

#### Scenario: Filter preserves remaining radio-probe step
- **WHEN** more than one candidate survives the AXP filter
- **THEN** the radio SPI probe SHALL be applied to the surviving candidates as before

### Requirement: IDF i2c.master log suppressed during AXP probe
The firmware SHALL suppress IDF `i2c.master` error logs at ESP_LOG_NONE during the AXP I2C probe to avoid logging expected NACKs on boards without AXP. The log level SHALL be restored to ESP_LOG_WARN after the probe.

#### Scenario: Log suppression during AXP probe
- **WHEN** the AXP probe is about to execute
- **THEN** `esp_log_level_set("i2c.master", ESP_LOG_NONE)` SHALL be called before the probe
- **AND** `esp_log_level_set("i2c.master", ESP_LOG_WARN)` SHALL be called after the probe

### Requirement: checkAXP accepts pre-detected chip ID to avoid redundant probing
`Power::checkAXP()` SHALL accept an optional parameter `knownChipID` (default 0). When `knownChipID` is non-zero, it SHALL skip `Wire.begin()` and the I2C probe and proceed directly to the power-rail configuration step using the provided chip ID.

#### Scenario: checkAXP called with known AXP192 chip ID
- **WHEN** `boardDetection()` detected AXP192 and calls `power.checkAXP(0x03)`
- **THEN** `checkAXP()` SHALL configure AXP192 power rails without re-probing I2C

#### Scenario: checkAXP called with known AXP2101 chip ID
- **WHEN** `boardDetection()` detected AXP2101 and calls `power.checkAXP(0x4A)`
- **THEN** `checkAXP()` SHALL configure AXP2101 power rails without re-probing I2C

#### Scenario: checkAXP called with zero (legacy / fallback path)
- **WHEN** `checkAXP(0)` or `checkAXP()` is called (as from `tinyGS.ino`)
- **THEN** `checkAXP()` SHALL perform the full Wire.begin + I2C probe as before
