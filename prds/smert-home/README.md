# Smert Home Product Requirements Document

> **Note on the name**: "Smert" is an intentional misspelling of "Smart" — the irony of misspelling "smart" while building a genuinely thoughtful system is part of the charm.

---

## 1. System Overview

Smert Home is a distributed hardware and software ecosystem for intelligent home automation. The system uses a Controller Area Network (CAN) bus as its backbone — both for communication and power distribution — enabling daisy-chained wiring topologies that simplify installation.

### 1.1 Core Philosophy

- **Party Bus Architecture**: Single CAN bus for all devices (not star topology)
- **Power Over CAN**: 24V DC distributed on the bus powers all non-mains devices
- **Thoughtful Isolation**: Opto-isolation at critical boundaries prevents fault propagation
- **Flexible Front-Ends**: Scene controllers support swappable button plates via pogo connectors
- **Standalone Operation**: Full functionality without external cloud services or ecosystems
- **Standards Integration**: Native Matter support for interoperability with Apple Home, Google Home, etc.

### 1.2 Hardware Platform

| Component | MCU Family | Rationale |
|-----------|------------|-----------|
| Device Modules | STM32G4 | Mixed-signal capabilities, cost-effective, sufficient for I/O and sensing |
| Processor Module | STM32H7 | High performance, TCP/IP stack, Matter protocol processing, larger firmware |

---

## 2. Communication Architecture

### 2.1 CAN Bus Specification

- **Protocol**: Standard CAN (ISO 11898-1), NOT CAN FD
- **Bitrate**: 500 kbps (configurable, 125-1000 kbps range)
- **Topology**: Daisy-chain (four pins in, four pins out per device)
- **Pinout**: 24V+, 24V GND, CAN_H, CAN_L
- **Termination**: 120Ω resistor at each end of bus
- **Max Devices**: 64 (protocol limited, not electrical)

### 2.2 Opto-Isolation Boundaries

Isolation is applied at:

1. **Isolator/Power Booster**: Between upstream (non-isolated) and downstream CAN
2. **AC Load Controller**: Between 24V CAN side and AC mains side
3. **Processor Module**: Isolation on CAN ports (TBD based on installation context)

All other modules use standard CAN transceiver protection (TVS diodes, etc.) without full opto-isolation.

---

## 3. Power Architecture

### 3.1 Power Entry: Isolator/Power Booster

**Input**: USB-C PD (specific profile TBD, minimum 45W recommended)

**Outputs**:
- **24V DC**: Bus power for downstream devices (current limited, fused)
- **Isolated CAN**: Opto-isolated CAN signals pass through, power pins are locally generated

**Upstream CAN**: Uses data pins only (no power) — allows safe connection to non-isolated networks

### 3.2 Power Budget

| Module Type | Estimated Draw | Notes |
|-------------|----------------|-------|
| Isolator/Power Booster | 2-3W | Conversion losses, internal 3.3V supply |
| AC Load Controller | 1-3W | E-ink display + relay coil when active (brief, ~100mA @ 24V) |
| 0-10V Controller | 0.5-1W | E-ink display, minimal analog output current |
| Closed Contact Sensor | 0.5-1W | 16-channel wet contact supply |
| Doorbell Detector Retrofit | 0.5-1W | Current sensing circuitry |
| Scene Controller | 1-3W | WRGB LEDs dominate when active |
| Environmental Sensor | 0.5W | Low duty cycle sensing |
| Processor Module | 3-5W | Large e-ink, Ethernet PHY, Matter processing |

### 3.3 Module Power Supplies

Each module generates local 3.3V (and 5V if needed) from the 24V bus input. Isolated modules use DC-DC converters with isolation ratings suitable for 120VAC fault protection (minimum 2.5kV).

---

## 4. Module Specifications

### 4.1 Isolator/Power Booster

**Purpose**: Safe boundary between non-isolated upstream network and isolated downstream devices. Injects 24V power.

**Hardware**:
- USB-C PD input (45W minimum capability)
- 24V DC-DC converter (isolated)
- Dual CAN transceivers with opto-isolation between them
- 120Ω termination resistor (switchable/jumper)
- Reset button (restarts local power supply)
- Small e-ink display: device ID, power status, bus voltage

**Physical**:
- DIN rail mount
- 2-4 module widths (TBD based on power supply size)
- LED array: bus voltage indicator (8 levels)

**Protection**:
- Input: USB-C PD overcurrent, overvoltage
- Output: 24V current limit, short-circuit protection
- Isolation: 2.5kV minimum (suitable for 120VAC fault propagation prevention)

---

### 4.2 AC Load Controller

**Purpose**: AC load control with voltage/current monitoring. Supports three output modes: phase-control dimming (leading or trailing edge) or mechanical relay switching.

**Hardware**:
- STM32G4 MCU
- Three output stages (all present, software-selected):
  - TRIAC: Leading-edge phase control (resistive/inductive loads)
  - MOSFET pair: Trailing-edge phase control (capacitive/LED loads)
  - Mechanical relay: 120VAC @ 20A, hard switching
    - Service life: >20,000 cycles minimum (preferably 100,000+)
    - SPST or SPDT configuration
- AC input monitoring (always active, regardless of output mode):
  - ZMPT101B voltage transformer (waveform sampling)
  - CT clamp current sensor (current and power measurement)
- Small e-ink display: device ID, output mode, voltage/current/power readings
- LED array: dimming level (8 levels) or relay state indication
- Physical disconnect switch (guarantees load off, software cannot override)
- Reset button

**Output State Persistence Architecture**:
- **Memory Buffer Device**: TBD implementation
  - Dual-port RAM, small FPGA, or similar shared memory solution
  - Processor writes desired state to buffer
  - Independent "mem-to-GPIO" controller reads buffer and drives physical outputs
- **Benefits**:
  - Output state maintained during processor reboots
  - Output state maintained if processor offline
  - Processor only required for state *changes*, not state *maintenance*
- **Fail-Safe**:
  - Reset button clears memory buffer to "off" state
  - All outputs disabled on reset
- **State Tracking**:
  - Buffer contains: output mode (TRIAC/MOSFET/RELAY), target value
  - Mem-to-GPIO controller continuously applies buffered state to output circuits

**Physical**:
- DIN rail mount
- 4-6 module widths (accommodates all three output circuits)
- Mains terminal block (line, load, neutral): WAGO style connectors
- CAN passthrough (four pins in, four pins out)

**Configuration**:
- **Default mode**: Relay (on/off switching) - safe, predictable behavior out-of-box
- Software-configurable via HTTP interface:
  - Output mode: TRIAC / MOSFET / Relay
  - For TRIAC/MOSFET: dimming curves, minimum/maximum levels, fade rates
  - For relay: state only (on/off)
  - Load type: Resistive, Inductive (fan), LED, etc. (used for safety interlocks)

**Capabilities**:
- Continuous dimming curves (resistive/lighting)
- Discrete stepped modes (fan-safe operating regions)
- Voltage waveform sampling
- Current and power measurement
- Load verification: Detects mismatches between commanded state and actual power draw (fault detection)
- Load classification via waveform analysis (future)

**Safety**:
- Physical switch disconnects load regardless of output mode
- Opto-isolation between 24V CAN side and AC mains side
- Output mode can be locked (require password to change from relay mode)
- Clearance/creepage per mains voltage standards

---

### 4.3 0-10V Controller

**Purpose**: Analog 0-10V dimming control for external LED drivers and ballasts. Low-voltage module with no mains connection.

**Hardware**:
- STM32G4 MCU
- 0-10V analog output (isolated from CAN bus)
  - Sourcing or sinking configuration (TBD based on common driver standards)
  - Resolution: 8-bit minimum (256 steps)
- Small e-ink display: device ID, current output level (0-100% / 0-10V)
- LED array: output level indication (8 levels)
- Reset button

**Physical**:
- DIN rail mount
- 1-2 module widths
- Two-position terminal block: 0-10V and common
- CAN passthrough (four pins in, four pins out)
- Has mounting holes for mounting without a din rail

**Operation**:
- Receives brightness commands via CAN (0-100%)
- Maps to 0-10V analog output in software
- No current sensing (output only, no feedback)
- Used by external LED drivers, ballasts, or other 0-10V compatible equipment

**Isolation**:
- Output isolated from CAN bus (prevents ground loops with external equipment)
- 500V isolation minimum

---

### 4.4 Closed Contact Sensor (16-Channel)

**Purpose**: Monitor state of window/door sensors and other dry contacts.

**Hardware**:
- STM32G4 MCU
- 16 independent wet contact channels
  - Supply: 12V @ ~1-2mA per channel (current-limited)
  - Detection: Current presence/absence (open = no current, closed = current flows)
- Software-configurable: NO (normally open) or NC (normally closed) interpretation per channel
- Software debouncing (configurable duration per channel)
- Small e-ink display: device ID, channel states summary
- LED array: activity indicator (any channel active)
- Reset button

**Physical**:
- DIN rail mount
- 2-4 module widths
- 16 two-position terminal blocks (or 8 dual-channel blocks)
- CAN passthrough (four pins in, four pins out)

**Operation**:
- Each channel supplies current to external contact
- Reads voltage drop across sense resistor to detect current
- Reports state changes to processor module via CAN
- Supports end-of-line resistor supervision (future consideration)

**Note**: The Closed Contact Sensor can also function as a simple doorbell detector when configured for a single momentary contact (button press) rather than continuous monitoring.

---

### 4.5 Doorbell Detector Retrofit

**Purpose**: Detect doorbell button presses on traditional low-voltage doorbell circuits (16-24VAC).

**Hardware**:
- STM32G4 MCU
- Current sensing circuit:
  - Measures baseline current (doorbell lamp/transformer idle current)
  - Detects current spike when button pressed (doorbell solenoid activated)
  - Differential detection distinguishes lamp current from button press
- Two-wire connection to doorbell circuit (parallel to existing button)
- Small e-ink display: device ID, last press time
- LED array: activity indicator
- Reset button

**Physical**:
- DIN rail mount
- 1-2 module widths
- Two-position terminal block for doorbell wires
- CAN passthrough (four pins in, four pins out)

**Operation**:
- Non-invasive (parallel connection, does not replace button)
- Configurable current threshold and duration for "press" detection
- Reports press events to processor module

---

### 4.6 Scene Controller

**Purpose**: User input device with configurable button plates and per-button WRGB LED feedback.

**Hardware**:
- STM32G4 MCU
- **Front Plate Interface**: 16-pin pogo connector
  - 2 pins per button (up to 8 buttons)
  - Allows swappable button plates (rocker, pushbutton, touch, etc.)
- **Button Reading**: GPIO expander IC (I2C or SPI-based, e.g., MCP23017 or PCA9535)
- **WRGB LED Control**: LED driver IC (I2C, e.g., PCA9635 16-channel PWM)
  - One WRGB LED per button position (up to 8)
  - Each LED independently controllable (exposed as a "light" in the system)
- **Configuration Detection**: GPIO expander pins hardcoded with resistor dividers to identify:
  - Number of buttons installed (1-8)
  - Which positions have LEDs installed
  - Button-to-LED mapping
- Small e-ink display: device ID, active scene (optional, minimal use)
- Reset button

**Physical**:
- **NOT DIN rail** — designed for 1-gang electrical box
- Decorator/Decora wall plate compatible
- Low-voltage only (24V from CAN)
- Designed for four-conductor low-voltage cable (e.g., old phone wire, half Ethernet, Crestron cable)

**Pin Optimization**:
- MCU communicates with GPIO expander and LED driver via shared I2C/SPI bus
- Reduces pin count from 16+ (direct GPIO) to 4-6 (bus + interrupt)

**Operation**:
- Each button reports press/release events to processor module
- Each WRGB LED receives color/brightness commands from processor
- LEDs can indicate: scene active, load status, night light, etc.

---

### 4.7 Environmental Sensor

**Purpose**: Monitor temperature, humidity, and air quality (PM2.5 and CO2).

**Hardware**:
- STM32G4 MCU
- Temperature/humidity sensor (I2C, e.g., SHT30 or SHT40)
- PM2.5 sensor (particulate matter, TBD model)
- CO2 sensor (TBD model, e.g., SGP30, SCD40, or similar)
- Small e-ink display: device ID, current readings (temp, humidity, PM2.5, CO2)
- LED array: status indicator
- Reset button

**Physical**:
- **NOT DIN rail** — dual-gang decora style (two standard switch openings side by side)
- Provides adequate space for airflow and multiple sensors
- Low-voltage (24V from CAN)
- Wall plate mounting

**Operation**:
- Periodic sensor readings (configurable interval per sensor type)
- Reports to processor module via CAN
- Display updates on significant changes or periodically
- Sensor models selected based on Matter device type support and availability

---

### 4.8 Processor Module

**Purpose**: Central hub for configuration, Matter integration, automation logic, and firmware orchestration.

**Hardware**:
- STM32H7 MCU (high performance variant)
- 4" e-ink display (larger than other modules)
- Navigation buttons: Up, Down, Left, Right, Select
- Ethernet PHY (RJ45 connector, PoE not used — separate USB-C power)
- USB-C power input (separate from network power, for processor module only)
- 4x CAN ports (bridged internally, for convenience — not isolated switches)
  - All ports output 24V (powered)
- SD card slot (SPI interface) for configuration database storage
- Reset button
- Matter pairing QR code display button (via menu option)

**Physical**:
- DIN rail mount
- 4-6 module widths (accommodates large display and connectors)
- RJ45 Ethernet port
- 4x 4-pin CAN connectors
- USB-C power input
- SD card accessible from front

**Display Content**:
- Main screen: Status overview, IP address, CAN bus status
- Menu system (navigated via buttons):
  - Network status
  - Connected device list
  - Configuration password display
  - Configuration password rotation
  - Matter pairing QR code
  - Firmware update status
  - Diagnostic logs

**Connectivity**:
- Ethernet: HTTP configuration interface
- CAN: All device communication
- DHCP server: For Smert device network (isolated from home network)

---

## 5. Software Architecture

### 5.1 Operating System

- **All Modules**: FreeRTOS
  - Provides task scheduling, inter-task communication, and deterministic timing
  - Suitable for both resource-constrained device modules and complex processor module

### 5.2 Repository Structure

- **Monorepo**: All firmware in single repository
- **Build System**: Multiple build targets per module type
- **Shared Libraries**: Common code for CAN protocol, e-ink drivers, etc.

```
/smert-home-firmware
  /libs
    /can_protocol      # Shared CAN message definitions
    /eink_driver       # Common e-ink display driver
    /led_array         # LED array driver
    /matter_stack      # Matter SDK integration (processor only)
    /http_server       # HTTP server (processor only)
    /sd_card           # SD card/FAT filesystem (processor only)
    /wrgb_driver       # WRGB LED driver (scene controller)
    /gpio_expander     # I2C/SPI GPIO expander driver
  /apps
    /isolator          # Isolator/Power Booster firmware
    /ac_load_controller # AC Load Controller firmware (TRIAC/MOSFET/Relay)
    /0-10v_controller  # 0-10V Controller firmware
    /contact_sensor    # Closed Contact Sensor firmware
    /doorbell_detector # Doorbell Detector Retrofit firmware
    /scene_controller  # Scene Controller firmware
    /env_sensor        # Environmental Sensor firmware
    /processor         # Processor Module firmware
  /web_assets        # SPA HTML/CSS/JS (stored in external flash)
  /tools
    /programmer        # Programming utility
    /configurator      # HTTP config helper
```

---

## 6. CAN Protocol Specification

### 6.1 Addressing

- **CAN ID**: 11-bit standard identifier
- **Assignment**: 11 least significant bits of STM32 unique ID (UID)
- **Storage**: Written to EEPROM on first boot (if not already present)
- **Collision Handling**: User can reassign via configuration interface if collision detected

### 6.2 Message Types

**Common Message Types (all devices)**:

| Message Type | Direction | Description |
|--------------|-----------|-------------|
| `DEVICE_ID_GET` | Processor → Device | Request device unique ID |
| `DEVICE_ID_REPORT` | Device → Processor | Report 96-bit UID |
| `DEVICE_TYPE_GET` | Processor → Device | Request device type |
| `DEVICE_TYPE_REPORT` | Device → Processor | Report module type (byte code) |
| `FIRMWARE_VERSION_GET` | Processor → Device | Request firmware version |
| `FIRMWARE_VERSION_REPORT` | Device → Processor | Report version string |
| `PROTOCOL_VERSION_GET` | Processor → Device | Request protocol version |
| `PROTOCOL_VERSION_REPORT` | Device → Processor | Report protocol version |
| `EEPROM_UPDATE` | Processor → Device | Write data to EEPROM (address + data) |
| `FIRMWARE_PAGE_OFFER` | Processor → Device | Offer firmware page for update |
| `FIRMWARE_PAGE_ACCEPT` | Device → Processor | Accept page, ready to receive |
| `FIRMWARE_PAGE_DATA` | Processor → Device | Firmware page data |
| `FIRMWARE_PAGE_VERIFY` | Device → Processor | Page received, checksum OK |
| `FIRMWARE_COMMIT` | Processor → Device | Commit new firmware, reboot |

**Module-Specific Message Types**:

| Module | Message Type | Description |
|--------|--------------|-------------|
| AC Load Controller | `OUTPUT_MODE_GET` | Request current output mode |
| AC Load Controller | `OUTPUT_MODE_SET` | Set output mode (TRIAC/MOSFET/RELAY) |
| AC Load Controller | `VOLTAGE_GET` | Request voltage reading |
| AC Load Controller | `VOLTAGE_REPORT` | Report voltage waveform sample |
| AC Load Controller | `CURRENT_GET` | Request current reading |
| AC Load Controller | `CURRENT_REPORT` | Report current/power measurement |
| AC Load Controller | `DIMMER_SET` | Set dimming level (0-100%, TRIAC/MOSFET only) |
| AC Load Controller | `DIMMER_GET` | Request current dimming level |
| AC Load Controller | `RELAY_SET` | Set relay state (on/off, relay mode only) |
| AC Load Controller | `RELAY_GET` | Request relay state |
| AC Load Controller | `CURVE_SET` | Set dimming curve parameters |
| 0-10V Controller | `OUTPUT_SET` | Set 0-10V output level (0-100%) |
| 0-10V Controller | `OUTPUT_GET` | Request current output level |
| Contact Sensor | `CHANNEL_GET` | Request specific channel state |
| Contact Sensor | `CHANNEL_REPORT` | Report channel state change |
| Contact Sensor | `ALL_CHANNELS_GET` | Request all 16 channel states |
| Doorbell Detector Retrofit | `DOORBELL_GET` | Request last press timestamp |
| Doorbell Detector Retrofit | `DOORBELL_EVENT` | Doorbell press detected (async) |
| Scene Controller | `BUTTON_EVENT` | Button press/release (async) |
| Scene Controller | `LED_SET` | Set WRGB LED color/brightness |
| Scene Controller | `LED_GET` | Request LED state |
| Environmental | `TEMP_GET` | Request temperature |
| Environmental | `TEMP_REPORT` | Report temperature reading |
| Environmental | `HUMIDITY_GET` | Request humidity |
| Environmental | `HUMIDITY_REPORT` | Report humidity reading |

### 6.3 Event Reporting

Devices send asynchronous reports when:
- Input state changes (button press, contact open/close, doorbell ring)
- Sensor readings change significantly (configurable threshold)
- Periodic heartbeat (configurable interval)

---

## 7. Device Addressing & Discovery

### 7.1 Initial Addressing

1. On first boot, device reads STM32 UID
2. Takes 11 LSBs as proposed CAN ID
3. Checks EEPROM for stored ID:
   - If empty: uses proposed ID, stores in EEPROM
   - If present: uses stored ID (allows manual override)
4. Begins operation with this ID

### 7.2 Collision Detection

- Processor module maintains ID registry
- If duplicate ID detected (two devices responding to same address):
  - Flag in configuration interface
  - User prompted to resolve via HTTP interface
  - Resolution: assign new ID to one device, stored in its EEPROM

### 7.3 Discovery Protocol

1. Processor broadcasts `DEVICE_ID_GET` to all addresses
2. Devices respond with `DEVICE_ID_REPORT` (96-bit UID)
3. Processor builds device table: CAN ID, UID, Device Type, Firmware Version
4. New devices detected via periodic scans or manual trigger

---

## 8. Configuration & Commissioning

### 8.1 Web Configuration Interface

**Architecture**: Single Page Application (SPA)

**Asset Storage**:
- HTML, CSS, JavaScript, and image assets stored in external SPI flash (connected to processor)
- STM32H7 internal flash insufficient for full web application
- Assets updatable via firmware update system

**Technology Stack**:
- Frontend framework: React or similar functional-style framework
- Footprint optimization acceptable (alternative: Preact, Solid.js, or vanilla JS with reactive patterns)

**Core Design Principles**:
1. **JSON-Centric**: All configuration represented as JSON
2. **Offline-First**: UI functions without server connection after initial load
3. **Local-First**: User owns their configuration file

**Interface Modes**:

**Configuration Mode (Offline-Capable)**:
- **Download**: Browser downloads `smert-config.json` from processor
- **Edit**: User modifies JSON (via SPA UI or external editor)
- **Upload**: Browser file picker selects JSON file
  - Server validates: schema structure, required fields, device ID existence on CAN bus
- **Activate**: Validated config written to SD card and applied

**Status Mode (Live Connection Required)**:
- Real-time I/O state monitoring
- Manual override controls for testing
- Direct read/write of module inputs/outputs
- Requires active processor connection

**Security**:
- Password protection (configurable)
- Password display: Menu option on processor e-ink screen
- Password rotation: Menu option generates new password

### 8.2 Configuration Storage

- **Location**: SD card on processor module (SPI interface)
- **Format**: single JSON file
- **Contents**:
  - Device registry (ID, type, name, description, location)
  - Automation rules
  - Matter credentials (bridge and controller modes)
  - System settings

### 8.3 Matter Pairing

- **QR Code Display**: Menu option or dedicated button shows Matter pairing QR on e-ink display
- **Pairing Key**: Displayed alongside QR code for manual entry
- **Modes**:
  - **Bridge Mode**: Processor joins external Matter ecosystem, exposes all Smert devices
  - **Controller Mode**: Processor accepts 3rd-party Matter devices, stores credentials

---

## 9. Logic Engine

### 9.1 Rules-Based Automation

**Rule Structure**:
```
IF (condition) THEN (action)
```

**Inputs (Conditions)**:
- Digital inputs (button press, contact sensor, doorbell)
- Sensor thresholds (temperature > X, humidity < Y, light level < 50%)
- Time-of-day (dawn, dusk, night, day)
- Output state feedback (light is on, dimmer at 30%)
- Boolean combinations (AND, OR, NOT)
- Delayed signals (output of timer block)

**Outputs (Actions)**:
- Set light brightness (0-100%)
- Set relay state (on/off)
- Set WRGB LED color
- Trigger scene

### 9.2 State-Based Operation

- Rules are evaluated continuously based on current state
- No procedural execution (no "wait 5 minutes then...")
- Delays implemented as derived inputs (timer block outputs boolean after configured duration)
- Hysteresis supported for threshold inputs (e.g., light > 60% to activate, light < 40% to deactivate)

### 9.3 Composite Inputs

- Multiple conditions combined with boolean logic
- Example: "IF (motion detected AND night time) THEN (turn on light)"
- Example: "IF (light > 50% AND no occupancy for 10 minutes) THEN (dim to 20%)"

---

## 10. Matter Integration

### 10.1 Bridge Mode (Smert → External)

- Processor module connects to external Matter ecosystems (Apple Home, Google Home, etc.)
- All Smert devices appear as native Matter accessories:
  - AC Load Controller (TRIAC/MOSFET mode) → Dimmable Light or Fan
  - AC Load Controller (Relay mode) → On/Off Light or Switch
  - 0-10V Controller → Dimmable Light (external driver)
  - Contact Sensor → Contact Sensor
  - Doorbell Detector Retrofit → Doorbell (or generic button/switch)
  - Environmental Sensor → Temperature/Humidity Sensor, Air Quality Sensor
  - Scene Controller → Scene or Stateful Programmable Switch
- Device types not directly supported mapped to closest equivalent

### 10.2 Controller Mode (External → Smert)

- Processor acts as Matter Commissioner for 3rd-party devices
- Stores credentials for joined devices
- Surfaces 3rd-party devices through bridge:
  - External Matter light appears in HTTP interface alongside Smert devices
  - External sensors can trigger Smert automation rules
- Goal: Single integrated system regardless of device origin

### 10.3 Standalone Operation

- Full functionality without external Matter ecosystem
- Local HTTP interface provides complete control
- Matter protocol only required for external integration

### 10.4 Implementation

- **Language**: C++ (Matter SDK native support)
- **Alternative**: Rust (if Matter RS mature at implementation time)
- **SDK**: Connected Home over IP (CHIP) / Matter SDK
- **Initial Device Types**: Lights (on/off, dimmable, color), Sensors (contact, temp, humidity), Switches
- **Future Types**: Locks, thermostats, cameras (as capabilities expand)

---

## 11. Firmware Update System

### 11.1 Architecture

- **Orchestrator**: Processor module manages all updates
- **Storage**: Firmware pages stored in EEPROM of target device
- **Execution**: Device applies update and reboots autonomously

### 11.2 Update Flow

1. User uploads new firmware via HTTP interface (processor module)
2. Processor validates firmware (signature check, version compatibility)
3. Processor offers firmware to target device via `FIRMWARE_PAGE_OFFER`
4. Device accepts, receives pages via `FIRMWARE_PAGE_DATA`
5. Device verifies each page, stores in EEPROM
6. All pages received, device reports `FIRMWARE_PAGE_VERIFY`
7. Processor sends `FIRMWARE_COMMIT`
8. Device writes new firmware to flash, reboots
9. Device reports new version on startup

**Processor Firmware Update - State Preservation**:
- During processor firmware update, AC Load Controller modules maintain current output state via memory buffer architecture
- Mem-to-GPIO controller continues applying last-received state from buffer
- No interruption to controlled loads during processor reboot
- After processor restart:
  - Processor reads current output states from modules
  - Resumes normal operation without changing states

### 11.3 Safety

- Rollback capability: Previous version preserved until new version confirmed working
- Update verification: Checksum validation on each page
- Power failure recovery: Resume interrupted updates

---

## 12. Safety & Physical Design

### 12.1 Physical Disconnect Switches

- **AC Load Controller**: Hardware switch disconnects load (independent of relay/TRIAC/MOSFET output mode)
- **Purpose**: Guarantees circuit is not live for maintenance
- **Override**: Software cannot activate load when switch is in "off" position
- **Advantage**: Eliminates need for tag-out/lock-out procedures in most cases

### 12.2 Form Factors

| Module | Form Factor | Notes |
|--------|-------------|-------|
| Isolator/Power Booster | DIN rail | 2-4 module widths |
| AC Load Controller | DIN rail | 4-6 module widths (accommodates all three output circuits) |
| 0-10V Controller | DIN rail | 1-2 module widths |
| Closed Contact Sensor | DIN rail | 2-4 module widths |
| Doorbell Detector Retrofit | DIN rail | 1-2 module widths |
| Scene Controller | 1-gang box | Decorator plate compatible, low-voltage only |
| Environmental Sensor | Dual-gang box | Decora style with airflow for PM2.5/CO2 sensors |
| Processor Module | DIN rail | 4-6 module widths |

### 12.3 Isolation Ratings

- **Isolator/Power Booster**: 2.5kV minimum isolation
- **AC Load Controller**: 2.5kV between 24V CAN and AC mains
- **Purpose**: Prevent 120VAC fault propagation to low-voltage network

---

## 13. Manufacturing & Programming

### 13.1 Programming Interface

- **Method**: Pogo pin programming fixture
- **Interface**: SWD (Serial Wire Debug) for STM32
- **Connection**: 4-5 pins (VCC, GND, SWDIO, SWCLK, nRST)
- **Process**:
  1. Load PCB into programming fixture
  2. Automated/programmer-initiated flash of bootloader
  3. First-boot initialization of EEPROM with UID-derived CAN ID
  4. Functional test via CAN loopback
  5. Mark as programmed (ink mark or digital flag)

### 13.2 First Boot Sequence

1. Bootloader initializes hardware
2. Reads STM32 96-bit UID
3. Computes 11-bit CAN ID (LSBs of UID)
4. Checks EEPROM for existing ID
   - If blank: writes computed ID to EEPROM
   - If present: uses existing ID (allows factory override)
5. Starts main application with assigned ID

### 13.3 Testing

- Each module tested with loopback CAN message before shipment
- Verify e-ink display operation
- Verify LED array operation
- Verify reset button function

---

## 14. Future Considerations

### 14.1 Certification (TBD)

- **UL/ETL Listing**: Required for mains-connected devices sold in US
  - Investigate requirements for TRIAC/relay modules, isolator
  - Consider pre-certified power supply modules
- **FCC Part 15**: Unintentional radiator certification
  - CAN bus signaling may require testing
  - Processor module Ethernet may require testing

### 14.2 Load Classification

- Future firmware feature: Analyze voltage/current waveforms
- Automatically detect load type (resistive, inductive, capacitive)
- Suggest optimal control mode (leading vs trailing edge)

### 14.3 End-of-Line Supervision

- Security application: Detect tampering with contact sensor wiring
- Requires end-of-line resistors on sensors
- Firmware support for resistance measurement

### 14.4 Expansion Possibilities

- Additional sensor types (CO2, air quality, occupancy)
- Motor control modules (curtains, blinds)
- Audio integration (doorbell chime variants)

---

## 15. Summary

Smert Home is a thoughtfully designed distributed automation system that prioritizes:

1. **Reliability**: Physical disconnects, opto-isolation, state-based logic
2. **Flexibility**: Swappable button plates, configurable rules, Matter interoperability
3. **Installability**: CAN bus daisy-chaining, 24V power distribution, DIN rail and standard box mounting
4. **Maintainability**: Firmware updates over CAN, HTTP configuration, SD card portability
5. **Independence**: Full functionality without cloud or external ecosystems

The system balances sophisticated capabilities with practical installation concerns, making it suitable for both professional integrators and advanced DIY installations.

### 15.1 Module Summary

| # | Module | Function |
|---|--------|----------|
| 1 | Isolator/Power Booster | USB-C PD → 24V, opto-isolation boundary |
| 2 | AC Load Controller | TRIAC/MOSFET/Relay output, voltage/current sensing |
| 3 | 0-10V Controller | Analog dimming control for external drivers |
| 4 | Closed Contact Sensor | 16-channel wet contact monitoring |
| 5 | Doorbell Detector Retrofit | Current-based doorbell press detection (legacy circuits) |
| 6 | Scene Controller | Configurable buttons with WRGB LED feedback |
| 7 | Environmental Sensor | PM2.5, CO2, temperature, and humidity monitoring |
| 8 | Processor Module | Central hub, Matter bridge/controller, HTTP config |

---

*Document Version: 1.0*
*Date: 2026-04-23*
