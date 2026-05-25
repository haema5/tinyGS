## MODIFIED Requirements

### Requirement: Phase 3a candidate filtering uses AXP probe as bidirectional discriminator
The OLED-based board detection Phase 3a SHALL apply AXP chip filtering with bidirectional strict-equality before the radio SPI probe. After confirming the OLED bus and running the AXP probe, only candidates where `candidate.axp_chip == detectedAxpChip` SHALL proceed to radio probing. The radio SPI probe and "first candidate fallback" are otherwise unchanged.

#### Scenario: AXP192 separates T-Beam family from TTGO LoRa 32 V2 (ESP32 classic, bus B)
- **WHEN** OLED bus SDA=21/SCL=22 is confirmed
- **AND** AXP probe returns 0x03 (AXP192)
- **THEN** the candidate pool SHALL contain only boards with `axp_chip == 0x03`: T-Beam V1.1 LF (idx 8), T-Beam V1.1 HF (idx 9), T-Beam V1.0 LF (idx 14), T-Beam V1.0 HF (idx 18)
- **AND** TTGO LoRa 32 V2 LF (idx 6) and HF (idx 7) SHALL be eliminated from the candidate pool
- **AND** all other bus-B boards with `axp_chip == 0x00` SHALL be eliminated

#### Scenario: T-Beam V1.1 LF (SX1278) selected first from AXP192 pool
- **WHEN** the candidate pool after AXP filter is {idx 8, idx 9, idx 14, idx 18} (all AXP192)
- **AND** target radio is SX1278 (RF_SX127X)
- **AND** T-Beam V1.1 LF (idx 8) is first in the pool
- **THEN** T-Beam V1.1 LF (idx 8) SHALL be selected (radio probe confirms RF_SX127X family on first candidate)

#### Scenario: T-Beam V1.0 LF detected as T-Beam V1.1 LF (accepted collision within AXP192 pool)
- **WHEN** the candidate pool after AXP filter is {idx 8, idx 9, idx 14, idx 18}
- **AND** target hardware is T-Beam V1.0 LF (idx 14, SX1278)
- **THEN** idx 8 (T-Beam V1.1 LF) SHALL be returned (first in pool with matching radio family)
- **AND** this collision SHALL be documented as a known hardware limitation — AXP192 is present on both V1.0 and V1.1; distinguishing them would require a board-version register or PROG-pin probe not currently implemented

#### Scenario: T-Beam V1.1 HF still collides within AXP192 pool (SX1276≡SX1278 ambiguity)
- **WHEN** AXP filter leaves {idx 8, idx 9, idx 14, idx 18}
- **AND** target radio is SX1276 (RF_SX127X, same silicon as SX1278)
- **THEN** T-Beam V1.1 LF (idx 8) SHALL be returned (first in pool; SX1276 and SX1278 are indistinguishable by SPI)
- **AND** this collision SHALL be logged and documented as a known hardware limitation

#### Scenario: No AXP eliminates entire T-Beam family including V1.0 (TTGO LoRa 32 V2 scenario)
- **WHEN** OLED bus SDA=21/SCL=22 is confirmed
- **AND** AXP probe returns 0x00 (no AXP found)
- **THEN** T-Beam V1.1 LF (idx 8), HF (idx 9), T-Beam V1.0 LF (idx 14), and HF (idx 18) SHALL ALL be eliminated from candidates
- **AND** the candidate pool SHALL contain only boards with `axp_chip == 0x00` on that bus (e.g. TTGO LoRa 32 V2)

#### Scenario: AXP2101 uniquely identifies T-Beam S3 Supreme (ESP32-S3, bus SDA=17/SCL=18)
- **WHEN** OLED bus SDA=17/SCL=18 is confirmed on ESP32-S3
- **AND** AXP probe returns 0x4A (AXP2101)
- **THEN** the candidate pool SHALL contain only T-Beam S3 Supreme (idx 2, `axp_chip == 0x4A`)
- **AND** Heltec LORA32 V3 (idx 0), Custom S3 SX1278 (idx 1), LILYGO SX1280 (idx 3), and Custom S3 LR2021 (idx 4) SHALL be eliminated
- **AND** T-Beam S3 Supreme SHALL be selected directly (single candidate, no radio probe needed)

#### Scenario: AXP filtering does not affect boards on different OLED buses
- **WHEN** OLED bus SDA=4/SCL=15 is confirmed (bus A on ESP32 classic)
- **AND** AXP probe returns 0x00 (no AXP on that bus)
- **THEN** the candidate pool SHALL contain only boards on that bus with `axp_chip == 0x00`
- **AND** detection behavior on bus A SHALL be unchanged from pre-AXP firmware

### Requirement: Web wizard displays detected AXP chip variant
The board information shown in the web configuration wizard and status page SHALL include the specific AXP chip variant name when one has been detected: "AXP192" for chip ID 0x03, "AXP2101" for chip ID 0x4A, or nothing when no AXP is present.

#### Scenario: AXP192 shown in wizard
- **WHEN** `AXPchip == 1` (AXP192 detected, as set in Power.cpp)
- **THEN** the board info section of the web wizard SHALL display "AXP192"

#### Scenario: AXP2101 shown in wizard
- **WHEN** `AXPchip == 2` (AXP2101 detected, as set in Power.cpp)
- **THEN** the board info section of the web wizard SHALL display "AXP2101"

#### Scenario: No AXP, nothing shown
- **WHEN** `AXPchip == 0` (no AXP detected)
- **THEN** no AXP field SHALL be shown in the wizard board info section
