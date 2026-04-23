# Autonomous Wildlife Imaging System (AWIS) Product Requirements Document

## 1. Executive Summary

### 1.1 Product Overview

AWIS is a portable, autonomous camera system designed to detect, track, and photograph wildlife in outdoor environments without human intervention.

### 1.2 Core Philosophy

- **"Image is truth"**: Vision-driven control, not precision robotics
- **Bias toward action**: Too many photos is better than missed opportunities
- **Iterative development**: Build manual → assisted → automatic in phases

### 1.3 Primary Goal

Capture at least one usable image per wildlife encounter, prioritizing:
- Subject in frame at user-specified position
- Sharp focus (preferably on eye(s))
- Proper exposure
- The indescribable "magic" of a compelling wildlife photograph

### 1.4 Target Wildlife

Initially optimized for large birds (eagles), but designed to detect any animal including:
- Bears, elk, moose, deer
- Foxes, wolves, coyotes
- Walrus, seals, sea lions
- Birds of prey (eagles, osprey, hawks)
- Any detectable wildlife (priority system handles preferences)

---

## 2. System Architecture

### 2.1 Physical System

```
┌─────────────────────────────────────────────────────────┐
│                    AWIS Physical Layout                  │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  [Bipod]──[Rail 6-8ft]──[Bipod]                         │
│       │                   │                              │
│       └────[Carriage]─────┘                              │
│              │                                           │
│       ┌──────┴──────┐                                    │
│       │   Gimbal    │                                    │
│       │  Pan + Tilt │                                    │
│       └──────┬──────┘                                    │
│         │         │                                      │
│    [GoPro]   [Nikon Z9 + Lens]                          │
│    Context    Primary (600mm f/5.6 typical)               │
│                                                          │
│  [Jetson Orin Nano] ←── Compute                          │
│  [Motor Controller] ←── MCU for steppers                 │
│  [SSD + HDD] ←── Storage                                 │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### 2.2 Components

| Component | Purpose | Key Spec |
|-----------|---------|----------|
| Linear Rail | Position carriage along 6-8ft track | Bi-pod supported at each end |
| Pan Motor | Rotate gimbal horizontally | 720°+ or continuous rotation with slip ring |
| Tilt Motor | Rotate gimbal vertically | ±90° typical |
| Carriage Motor | Move along rail | Speed TBD based on subject tracking |
| Nikon Z9 | Primary telephoto camera | 45.7MP, 1000BASE-T ethernet, USB PTP |
| GoPro | Context/wide-angle camera | ~120° FOV, HDMI or USB output |
| Jetson Orin Nano | AI detection, system control | GPU acceleration, NVMe storage |
| Motor Controller MCU | Real-time stepper control | STM32 or Arduino with motor hats |
| Slip Ring | Continuous pan power/data | Commercial off-the-shelf (TBD model) |

### 2.3 Logical System Components

| Component | Responsibility |
|-----------|----------------|
| Vision | Detect subjects from context camera |
| Control Loop | Convert detections → motor commands |
| Motion System | Execute pan/tilt/rail movement |
| Camera Control | Trigger + configure primary camera |
| Overlay/UI | Display video + framing info |
| Input | Manual controller override |
| Storage | Persist images + metadata |
| Networking | Local access + remote sync via Tailscale |

---

## 3. Phased Development Roadmap

### 3.1 Version 0 (MVP) — Manual Operation

**Features**:
- Fully manual control via game controller
- Live HDMI view with overlays
- Full camera control (trigger + settings)
- Motor control for pan/tilt/rail
- Basic storage and metadata

**Success Criteria**:
- Operator can manually track and photograph moving subjects
- System operates for 40 hours on battery
- Images saved with basic metadata

### 3.2 Version 1 (Fast Follow) — Assisted Mode

**Features**:
- Assisted mode: Manual framing + automatic shutter triggering
- Detection overlay (visualization only in v1.0)
- Confidence scoring display
- Calibration procedure implemented

**Success Criteria**:
- Auto-trigger fires when subject meets criteria
- ≥50% usable shots per encounter (operator framing)

### 3.3 Version 2 (Inflection Point) — Fully Automatic

**Features**:
- Detection → automatic framing → shooting
- Priority-based animal classification
- Auto on/off based on light conditions
- Background upload to remote storage

**Success Criteria**:
- ≥1 usable image per encounter without human intervention
- System runs unattended for full 40-hour target

---

## 4. Hardware Specifications

### 4.1 Compute: NVIDIA Jetson Orin Nano

**Rationale**:
- GPU acceleration for object detection
- Low power consumption
- NVMe SSD support for fast storage
- Linux-based (treat as regular computer)

**Requirements**:
- NVMe SSD (500GB-1TB) for hot storage
- USB ports for camera connections
- Ethernet for networking
- GPIO/I2C/UART for motor controller communication

### 4.2 Primary Camera: Nikon Z9

**Specifications**:
- 45.7MP full-frame mirrorless
- 1000BASE-T ethernet for fast file transfer
- USB PTP for settings control
- Mechanical shutter trigger (fallback)
- Built-in subject detection AF (animal/bird/people modes)

**Lens Support**:
- Primary: 600mm f/5.6 (bird photography)
- Alternative: 400mm f/2.8
- Wide-angle: 24mm f/2.8 or 50mm f/0.95 (different scenarios)
- Manual lens swap with recalibration per deployment

**Control Methods** (in order of preference):
1. Ethernet: Fast image transfer, some control
2. USB PTP: Settings, exposure compensation
3. Mechanical trigger: Shutter (fallback)

### 4.3 Context Camera: GoPro

**Purpose**:
- Wide FOV (~120°) for subject detection
- Mounted on gimbal with primary camera
- HDMI or USB output to Jetson

**Future Expansion**:
- Stationary context cameras (punted to P2)
- Multiple angles for better coverage

### 4.4 Motion System

#### 4.4.1 Axes

| Axis | Range | Purpose |
|------|-------|---------|
| Pan | 720°+ or continuous | Horizontal tracking |
| Tilt | ±90° | Vertical tracking |
| Rail | 6-8ft (1.8-2.4m) | Linear positioning |

#### 4.4.2 Motors

**Type**: Closed-loop stepper motors

**Benefits**:
- Missed step correction
- Position + velocity feedback
- Simpler integration than open-loop + separate encoders

**Integration**:
- Jetson sends target positions/commands to MCU
- MCU handles real-time stepping with closed-loop feedback
- Communication via UART/SPI/I2C

#### 4.4.3 Slip Ring (for continuous pan)

**Purpose**: Pass power and data through rotating joint

**Requirements**:
- Commercial off-the-shelf (TBD specific model)
- Minimum 6 circuits (power, data, signal)
- Rated for outdoor use
- Low friction

**Note**: Specific model to be determined in mechanical design phase. Alternative: wireless power + data if slip rings prove problematic.

#### 4.4.4 Motor Controller MCU

**Requirements**:
- Real-time stepper control
- Closed-loop feedback integration
- Communication with Jetson
- Endstop handling

**Options** (optimize for manufacturing ease):
- Arduino + motor driver shields (TMC2209, TMC5160)
- STM32 with integrated drivers
- Off-the-shelf CNC/motion control board

**Recommendation**: Evaluate existing "motor control hats" for Jetson or Arduino shields to minimize custom PCB work for MVP.

#### 4.4.5 Endstops

**Required for all axes**:
- Hard limit switches (safety)
- Optional "near limit" warnings (software)
- Position homing on startup

### 4.5 Storage Architecture

#### 4.5.1 Tiered Storage Strategy

```
┌─────────────────────────────────────────┐
│           Storage Hierarchy            │
├─────────────────────────────────────────┤
│                                         │
│  L1: NVMe SSD (500GB-1TB)              │
│      - Fast offload from Z9 (ethernet)  │
│      - Active session storage           │
│      - Quick access for processing      │
│                                         │
│  L2: HDD (2-4TB)                       │
│      - Archive storage                  │
│      - rsync'd from SSD during idle     │
│      - Low-cost, high capacity          │
│                                         │
│  L3: Remote Upload (background)          │
│      - Internet available → upload       │
│      - Resume capability                │
│      - Don't delete from HDD after      │
│                                         │
└─────────────────────────────────────────┘
```

#### 4.5.2 File Naming Convention

Format: `<root>/<YYYY-MM-DD>/<ulid>-<camera-id>.<ext>`

Examples:
- `/media/awis/2026-04-23/01J8PQK8Z5Y1B2C3D4E5F6G7H8-primary.nef`
- `/media/awis/2026-04-23/01J8PQK8Z5Y1B2C3D4E5F6G7H9-context.jpg`

**Rationale**:
- ULID: Lexicographically sortable, collision-free across cameras
- Date directory: Easy browsing by day
- Camera ID: Distinguishes primary vs context
- Extension: .nef (Nikon RAW), .jpg (compressed)

#### 4.5.3 File Types

- **Primary (Z9)**: RAW (.nef) + optional low-quality JPEG preview
- **Context (GoPro)**: JPEG or H264 video segments

**Note**: High burst rates expected. Automated culling is a separate future project.

### 4.6 Power System

#### 4.6.1 Power Budget

| Component | Average Draw | Peak Draw | Notes |
|-----------|--------------|-----------|-------|
| Jetson Orin Nano | 15-25W | 40W | GPU inference dependent |
| Nikon Z9 | 5-10W | 20W | Burst shooting, AF, VR |
| GoPro | 2-3W | 5W | Recording dependent |
| Motors (all 3) | 10-20W | 50W | Movement dependent |
| Motor Controller MCU | 2-5W | 5W | - |
| Storage (SSD+HDD) | 5-10W | 15W | Spin-up for HDD |
| Travel Router | 3-5W | 8W | WiFi + WAN |
| Cooling/Heating | 5-15W | 30W | Environmental dependent |
| **Total** | **47-113W** | **173W** | - |

**Target Average**: 30-100W (closer to 50W typical)

#### 4.6.2 User-Provided Power Station

**Requirements** (for user selection):
- Minimum capacity: 1.5-2kWh for 40-hour runtime at 50W average
- Recommended: 2-4kWh for safety margin and cold weather
- Output: 120V AC or 12V/24V DC (system will need appropriate PSU)
- Portability: Suitable for outdoor field deployment

**Note**: AWIS design specifies power draw. User selects appropriate Goal Zero/Jackery/Bluetti/etc. unit based on capacity needs.

### 4.7 Networking

#### 4.7.1 Local Network

**Travel Router**: GL.iNet or similar
- Creates local WiFi AP
- Wired ethernet for Jetson
- WAN port for internet connection

#### 4.7.2 Internet Connectivity

**Primary**: Starlink (user-provided, optional)
- Connects to travel router WAN
- Provides internet in remote locations

**Fallback**: No internet = local-only operation
- Images stored locally
- Upload resumes when connection available

#### 4.7.3 Remote Access

**Tailscale VPN**:
- Configured on Jetson
- User can access AWIS from anywhere
- Secure, no port forwarding needed

**RTSP Stream**:
- Context camera feed with overlays
- Accessible via Tailscale for remote monitoring

### 4.8 Environmental Specifications

#### 4.8.1 Weatherproofing

**Target**: IP65
- Dust protected
- Protected against water jets

**Enclosure**: Off-the-shelf weatherproof case for electronics
- Jetson, storage, motor controller housed inside
- Cable glands for wire entry/exit
- Ventilation/heating as needed for temperature management

**Camera Protection**:
- Nikon Z9 is weather-sealed (professional grade)
- Optional external housing acceptable for additional protection
- GoPro inherently weatherproof

**Pragmatic Note**: Well-placed umbrella acceptable for prototype/testing. Production version should be self-sufficient.

#### 4.8.2 Operating Conditions

- Temperature: 0°C to 40°C (extended range with heating/cooling)
- Humidity: 0-90% non-condensing
- Wind: System tolerant with closed-loop motor control and damping
- Vibration: Lens VR handles most; software damping for movements

---

## 5. Software Architecture

### 5.1 System Processes

```
┌─────────────────────────────────────────┐
│         AWIS Software Stack            │
├─────────────────────────────────────────┤
│                                         │
│  ┌─────────────┐  ┌──────────────┐      │
│  │   Vision    │  │  Control     │      │
│  │  (Detect)   │──│   Loop       │      │
│  └──────┬──────┘  └──────┬───────┘      │
│         │                │              │
│         │    ┌───────────┘              │
│         │    │                          │
│         │    ▼                          │
│  ┌──────┴────┐  ┌──────────────┐      │
│  │   Motor   │  │    Camera    │      │
│  │ Controller│  │   Controller │      │
│  └───────────┘  └──────────────┘      │
│                                         │
│  ┌─────────────┐  ┌──────────────┐      │
│  │   Overlay   │  │   Input      │      │
│  │    /UI      │  │   Handler    │      │
│  └─────────────┘  └──────────────┘      │
│                                         │
│  ┌─────────────┐  ┌──────────────┐      │
│  │   Storage   │  │   Upload     │      │
│  │   Manager   │  │  (background)│      │
│  └─────────────┘  └──────────────┘      │
│                                         │
└─────────────────────────────────────────┘
```

### 5.2 Core Processes

#### 5.2.1 Vision Process

**Input**: Context camera frames (GoPro via HDMI/USB)

**Processing**:
- Object detection using off-the-shelf SOTA model
- Animal classification with priority ranking
- Bounding box output
- Confidence scoring

**Output**:
- Bounding boxes (x, y, width, height)
- Detection confidence (0.0-1.0)
- Animal class (bear, eagle, unknown animal, etc.)
- Priority level (based on config)
- Timestamp

**Detection Model Strategy**:
- Start with best available pre-trained model (YOLOv8/v9/v10, Detectron2, etc.)
- Fine-tune on wildlife dataset if needed
- Plan for model swap in future (corresponds to compute upgrade cycle)

**Priority System**:
```json
{
  "detection_priority": [
    {"class": "eagle", "priority": 10},
    {"class": "bear", "priority": 9},
    {"class": "moose", "priority": 8},
    {"class": "fox", "priority": 7},
    {"class": "unknown_animal", "priority": 1}
  ]
}
```

- Configurable via JSON file
- Requires process restart to apply changes
- "unknown_animal" at bottom captures anything detectable

#### 5.2.2 Control Loop Process (Core)

**Rate**: 10-30 Hz

**Pipeline**:
1. **Read Detection**: Get latest from Vision Process
2. **Select Target**: Highest priority detection
3. **Compute Target Position**: Map context pixel → pan/tilt angles
4. **Apply Smoothing**: Velocity/acceleration limits
5. **Command Motors**: Send targets to Motor Controller MCU
6. **Decide Trigger**: Check if shooting criteria met

**State Machine**:
```
IDLE → DETECTED → TRACKING → SHOOTING → COOLDOWN → IDLE
```

#### 5.2.3 Motor Controller Process (on MCU)

**Responsibilities**:
- Real-time stepper control with closed-loop feedback
- Endstop monitoring
- Position/velocity reporting to Jetson
- Safety stops

**Communication**:
- Receives: Target positions, velocity limits, mode commands
- Sends: Current position, velocity, status, errors

#### 5.2.4 Camera Controller Process

**Nikon Z9 Control**:
- Ethernet: Image transfer, some settings
- USB PTP: Detailed settings, exposure compensation
- Mechanical: Shutter trigger (GPIO)

**Functions**:
- Configure settings (aperture, ISO, shutter mode)
- Trigger capture (via best available method)
- Monitor buffer status
- Periodic calibration shots (for auto on/off)

**Auto On/Off Logic**:
- Periodic calibration images assess light level
- If calibration image underexposed below threshold → sleep/detection off
- Resume when light returns
- Manual override always available

#### 5.2.5 Overlay/UI Process

**Input**: Context camera feed + detection data

**Output**:
- HDMI: Local display with overlays (low latency)
- RTSP: Remote stream (H264)

**Overlays**:
- Bounding boxes around detections
- Target frame (where primary camera is aimed)
- Confidence scores
- System status (mode, storage, battery)
- Pan/tilt/rail position indicators

#### 5.2.6 Input Handler Process

**Game Controller** (Bluetooth or USB):
- Left stick: Pan/Tilt speed
- Right stick: Rail movement
- Buttons: Trigger, mode change, settings
- Always overrides automatic control

**Web Interface**:
- Tailscale-secured access
- Mode switching
- Configuration
- Monitoring

#### 5.2.7 Storage Manager Process

**Responsibilities**:
- Manage file naming (ULID format)
- Embed XMP metadata in image files (EXIF/XMP packet injection)
- Tiered storage management (SSD → HDD)
- Cleanup/purge policies (TBD, don't delete without confirmation)

**Metadata Embedding**:
- Writes AWIS XMP namespace data directly into NEF and JPG files immediately after capture
- Uses Exiv2 or similar library for reliable XMP manipulation
- Preserves all existing camera EXIF data (Nikon, GoPro native metadata untouched)

#### 5.2.8 Upload Process (Background)

**Trigger**: Internet connection available (via Starlink/WAN)

**Behavior**:
- Background rsync from HDD to remote storage
- Resume interrupted uploads
- Don't delete local copies after upload
- Configurable remote endpoint (S3, NAS, etc.)

---

## 6. Control System

### 6.1 Coordinate Mapping: Context → Telephoto

**Problem**: Context camera (120° FOV) and telephoto (2-3° FOV) have radically different perspectives.

**Solution**: Calibration-based linear mapping

#### 6.1.1 Field Calibration Procedure

**Goal**: Map context camera pixels to pan/tilt angles

**Process**:
1. Place visible target in scene (person, calibration card, or distinct object)
2. For 5-10 positions across the frame:
   - Manually center target in primary camera (telephoto)
   - Record: Context pixel (x, y) and Pan/Tilt angles
3. Fit linear regression:
   ```
   pan  = a1*x + b1*y + c1
   tilt = a2*x + b2*y + c2
   ```

**Advantages**:
- Avoids complex camera calibration
- Handles distortion and misalignment naturally
- Specific to each lens/setup deployment

#### 6.1.2 Runtime Coordinate Conversion

```python
# Detection from context camera
nx = (detection.x - context_width/2) / context_width   # Normalized -0.5 to 0.5
ny = (detection.y - context_height/2) / context_height

# Convert to angular offset (approximate)
theta_pan = nx * context_hfov
theta_tilt = ny * context_vfov

# Apply calibration model
target_pan = current_pan + theta_pan + pan_offset
target_tilt = current_tilt + theta_tilt + tilt_offset
```

#### 6.1.3 Offsets and User Preferences

- **pan_offset, tilt_offset**: User-configurable (e.g., "put bird at left third, not center")
- **Composition**: Rule of thirds, centered, or custom position

### 6.2 Axis Control

#### 6.2.1 Axis Abstraction

```python
class Axis:
    target_position: float  # degrees (pan/tilt) or mm (rail)
    current_position: float
    velocity: float
    is_moving: bool
    max_velocity: float
    max_acceleration: float
```

#### 6.2.2 Smoothing and Constraints

- **Max Velocity Limits**: Prevent jerky motion, protect lens
- **Acceleration Limits**: Smooth starts/stops
- **Deadband**: ~0.2° (don't hunt around target)

#### 6.2.3 Rail Coordination Policy

**Primary**: Pan/Tilt do most tracking

**Rail moves when**:
- Pan approaches limits (~70-80% of range)
- Subject consistently poorly framed despite pan/tilt
- Hysteresis to avoid oscillation

### 6.3 Motor Control Loop

**Frequency**: 10-30 Hz on Jetson, higher on MCU

**Steps**:
1. Receive target position from Control Loop
2. Compute trajectory (S-curve or trapezoidal)
3. Send step commands to closed-loop drivers
4. Monitor encoder feedback
5. Report actual position/velocity back to Jetson

---

## 7. Detection, Framing, and Shooting

### 7.1 Detection Priority System

**Configuration** (JSON file, restart to apply):

```json
{
  "detection_config": {
    "enabled_classes": ["eagle", "bear", "moose", "deer", "fox", "unknown_animal"],
    "priority": {
      "eagle": 10,
      "bear": 9,
      "moose": 8,
      "deer": 5,
      "fox": 7,
      "unknown_animal": 1
    },
    "min_confidence": 0.3,
    "priority_boost_for_size": true,
    "tracking_timeout_seconds": 5
  }
}
```

**Behavior**:
- Always track highest priority detection
- Unknown animals get lowest priority but still captured
- Size can boost priority (large bear > small bird)

### 7.2 Confidence Scoring

**Confidence Formula**:
```
confidence_score = 
  w1 * detection_confidence +
  w2 * subject_size_score +
  w3 * centering_score +
  w4 * priority_boost
```

Where:
- `detection_confidence`: From vision model (0-1)
- `subject_size_score`: bbox_area / frame_area (0-1)
- `centering_score`: 1 - (distance_from_target_position / max_distance) (0-1)
- `priority_boost`: Scaled priority class (0-1)

**Default Weights** (configurable):
- w1: 0.3 (detection confidence)
- w2: 0.2 (size)
- w3: 0.3 (centering)
- w4: 0.2 (priority)

**Range**: 0.0 - 1.0

### 7.3 Triggering Logic

**Shoot When**:
```
IF confidence_score > threshold_low (0.5)
AND subject_size > min_threshold (configurable per animal)
THEN trigger_shutter()
```

**Additional Criteria**:
- Ignore motion state (still shoot even if moving)
- Add cooldown: 100-300ms between shots
- Burst mode: Multiple shots per trigger if configured

**Focus Strategy**:
1. Pre-focus to zone (manual or auto)
2. When subject detected, use camera's built-in AF
3. Prefer eye detection (camera feature)
4. Trigger when AF confirms (or immediately if AF fast enough)

### 7.4 Auto On/Off

**Strategy**: Light-based

**Process**:
1. Periodic calibration shots (every 5-10 minutes)
2. Analyze exposure (histogram or brightness)
3. If underexposed below threshold → enter sleep mode
   - Stop detection
   - Stop motor tracking
   - Minimal power consumption
4. Resume when calibration shot shows adequate light

**Threshold**: Configurable, based on "too dark for meaningful wildlife photography"

**Future Expansions** (P2/P3):
- Motion trigger (PIR or optical flow)
- Laser trip beam
- Scheduled active periods (dawn/dusk)

---

## 8. Metadata and Storage

### 8.1 Metadata Format

**Strategy**: XMP (Extensible Metadata Platform) embedded directly in image files

**Rationale**:
- Metadata travels with image (no separate files to manage)
- Lightroom compatible
- Supports custom namespaces for AWIS-specific data
- Works across NEF (Nikon Z9) and JPG (GoPro) formats

**Namespace**: `http://awis.example.org/ns/1.0/` (TBD official URI)

**XMP Schema**:

```xml
<rdf:RDF xmlns:rdf="http://www.w3.org/1999/02/22-rdf-syntax-ns#"
         xmlns:awis="http://awis.example.org/ns/1.0/"
         xmlns:xmp="http://ns.adobe.com/xap/1.0/"
         xmlns:dc="http://purl.org/dc/elements/1.1/"
         xmlns:photoshop="http://ns.adobe.com/photoshop/1.0/">
  
  <!-- AWIS Custom Fields -->
  <awis:CameraId>primary</awis:CameraId>
  <awis:CameraType>nikon_z9</awis:CameraType>
  <awis:Lens>600mm_f5.6</awis:Lens>
  
  <awis:PanDegrees>123.4</awis:PanDegrees>
  <awis:TiltDegrees>-12.3</awis:TiltDegrees>
  <awis:RailMm>450.0</awis:RailMm>
  
  <awis:PanVelocityDps>5.2</awis:PanVelocityDps>
  <awis:TiltVelocityDps>-1.1</awis:TiltVelocityDps>
  <awis:IsMoving>true</awis:IsMoving>
  
  <awis:DetectionConfidence>0.82</awis:DetectionConfidence>
  <awis:AnimalClass>eagle</awis:AnimalClass>
  <awis:Priority>10</awis:Priority>
  <awis:Bbox>640,480,120,80</awis:Bbox>
  <awis:ContextPixelX>640</awis:ContextPixelX>
  <awis:ContextPixelY>480</awis:ContextPixelY>
  
  <awis:FramingConfidence>0.67</awis:FramingConfidence>
  <awis:TargetComposition>left_third</awis:TargetComposition>
  <awis:CalibrationVersion>2026-04-23-field-001</awis:CalibrationVersion>
  
  <awis:ExposureMode>aperture_priority</awis:ExposureMode>
  <awis:AfMode>animal_detection</awis:AfMode>
  <awis:AfConfirmed>true</awis:AfConfirmed>
  
  <awis:AwisVersion>v2.1.0</awis:AwisVersion>
  <awis:DetectionModel>yolov8x-wildlife-v3</awis:DetectionModel>
  <awis:ControlLoopHz>20</awis:ControlLoopHz>
  
</rdf:RDF>
```

**Implementation Notes**:
- **Nikon Z9 (NEF)**: XMP embedded in standard location; Nikon EXIF data preserved and not modified
- **GoPro (JPG)**: XMP embedded in JPEG APP1 segment
- **No sidecar files**: If image is lost, metadata is lost (acceptable tradeoff for simplicity)
- **Lightroom compatibility**: Custom fields appear in Lightroom's metadata panel (under "AWIS" section)
- **Orthogonal to camera data**: AWIS namespace is separate from manufacturer EXIF; no conflicts with Z9's native metadata

**Tools**:
- Python: `python-xmp-toolkit` or `exiftool`
- C++: Adobe XMP SDK or Exiv2 library (recommended for cross-platform support)
- Command line: `exiftool` for debugging/verification
- Validation: Adobe XMP Schema validator

### 8.2 Directory Structure

```
/media/awis/
├── 2026-04-23/
│   ├── 01J8PQK8Z5Y1B2C3D4E5F6G7H8-primary.nef     (XMP metadata embedded)
│   ├── 01J8PQK8Z5Y1B2C3D4E5F6G7H9-context.jpg     (XMP metadata embedded)
│   ├── 01J8PQK8Z5Y1B2C3D4E5F6G7I0-primary.nef
│   └── ...
├── 2026-04-24/
│   └── ...
├── calibration/
│   ├── 2026-04-23-field-001.json                 (calibration data, not image metadata)
│   └── ...
├── config/
│   ├── detection_priority.json                   (system configuration)
│   ├── camera_settings.json                      (camera presets)
│   └── system.json                               (general settings)
└── logs/
    ├── 2026-04-23-control.log
    └── ...
```

**Note**: All image metadata is embedded via XMP. JSON files exist only for:
- Calibration profiles (separate from captured images)
- System configuration (not per-image data)
- Application logs

---

## 9. Web Interface (SPA)

### 9.1 Architecture

- **Framework**: React or similar functional-style framework
- **Deployment**: Served from Jetson (treat as regular Linux computer)
- **Access**: Via Tailscale VPN (secure, no port forwarding)
- **Features**:
  - Live view with controls
  - Configuration management
  - Calibration assistant
  - Gallery review
  - System monitoring

### 9.2 Key Features

**Live View**:
- RTSP stream display
- Detection overlay
- Manual control override
- Mode switching

**Configuration**:
- Detection priority editor
- Camera settings
- Motor limits
- Threshold adjustments

**Calibration Assistant**:
- Step-by-step walkthrough
- Point collection UI
- Model fitting display
- Save/restore calibration profiles

**Gallery**:
- Browse by date
- Filter by animal class
- Confidence score display
- Metadata viewer

**System**:
- Storage usage
- Power/battery status
- Temperature monitoring
- Log viewer

---

## 10. Operating Modes

### 10.1 Mode Priority (Highest to Lowest)

1. **Manual**: Controller input always overrides
2. **Assisted**: Manual framing, auto trigger
3. **Automatic**: Full auto (detection → framing → shooting)

### 10.2 Mode Details

#### Manual Mode
- Operator controls all movement
- Operator triggers shutter
- Detection overlay visible but doesn't control
- Use case: Setup, calibration, specific composition needs

#### Assisted Mode (v1)
- Operator manually frames subject
- System auto-triggers when criteria met
- Detection drives trigger decision only
- Use case: Operator wants control but doesn't want to miss shots

#### Automatic Mode (v2)
- System detects subject
- System automatically frames and tracks
- System triggers when criteria met
- Operator can override anytime
- Use case: Unattended operation

### 10.3 Mode Switching

- Physical button on system (LED indicator)
- Web interface
- Controller button combo
- Always revert to Manual on startup (safety)

---

## 11. Failure Handling

| Failure | Behavior |
|---------|----------|
| Endstop hit | Stop axis immediately, flag error, notify user |
| No detection | Optional slow scan (pan back/forth), or sleep |
| Camera disconnect | Continue tracking, attempt reconnect, alert user |
| Detection flicker | No special handling (v1), smoothing in v2 |
| Motor stall | Closed-loop driver attempts recovery, alert if persistent |
| Power low | Save state, finish current operation, safe shutdown |
| Storage full | Stop shooting, alert user, preserve existing data |
| Network disconnect | Continue local operation, queue uploads |

---

## 12. Build Order and Milestones

### Phase 0: Foundation (Weeks 1-4)
- [ ] Mechanical assembly (rail, gimbal, carriage)
- [ ] Motor installation with closed-loop drivers
- [ ] Endstop installation
- [ ] Basic motor controller firmware (position control)

### Phase 1: Manual Control (Weeks 5-8) — **v0 MVP**
- [ ] Jetson integration
- [ ] Game controller → motor control
- [ ] Nikon Z9 control (ethernet + trigger)
- [ ] GoPro capture
- [ ] HDMI overlay system
- [ ] Basic storage and metadata
- [ ] **Milestone**: Can manually track and shoot moving subjects

### Phase 2: Detection Integration (Weeks 9-12) — **v0.5**
- [ ] Install and configure detection model
- [ ] Vision process → overlay display
- [ ] Detection visualization only
- [ ] **Milestone**: Can see what system detects in real-time

### Phase 3: Calibration System (Weeks 13-14)
- [ ] Field calibration procedure
- [ ] Context → telephoto mapping
- [ ] Calibration storage and recall
- [ ] **Milestone**: Can calibrate new lens/setup in <10 minutes

### Phase 4: Assisted Mode (Weeks 15-18) — **v1**
- [ ] Assisted mode logic
- [ ] Triggering system
- [ ] Confidence scoring
- [ ] Priority system
- [ ] **Milestone**: ≥50% usable shots with operator framing

### Phase 5: Automatic Mode (Weeks 19-24) — **v2**
- [ ] Automatic framing logic
- [ ] Smoothing and prediction
- [ ] Auto on/off based on light
- [ ] Background upload
- [ ] **Milestone**: ≥1 usable image per encounter, 40-hour unattended operation

### Phase 6: Polish and Optimization (Weeks 25-30)
- [ ] Web SPA interface
- [ ] Remote monitoring via Tailscale
- [ ] Power optimization
- [ ] Environmental testing
- [ ] Documentation
- [ ] **Milestone**: Production-ready system

---

## 13. Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| Control loop rate | 10-30 Hz | Sufficient for smooth tracking |
| Detection latency | 50-150 ms | Model dependent |
| End-to-end latency | 200-700 ms | Detection → trigger decision |
| Runtime | ~40 hours | On user-provided power station |
| Shot success | ≥1 usable image per encounter | Primary goal |
| Usable shot rate (assisted) | ≥50% | v1 target |
| Calibration time | <10 minutes | Field procedure |

---

## 14. Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Context → telephoto mapping accuracy | Medium | High | Good calibration procedure; accept "good enough" over perfect |
| Stepper reliability under wind/load | Medium | High | Closed-loop steppers; over-specified torque; software damping |
| Detection latency vs motion speed | Medium | Medium | Fast model (YOLO); predict motion; accept some motion blur |
| Wide-angle distortion effects | Low | Medium | Calibration handles distortion; simple linear model sufficient |
| Mechanical vibration affecting image quality | Medium | High | Lens VR; software smoothing; vibration isolation (if needed) |
| Power consumption exceeding budget | Low | High | Power monitoring; adjustable sleep modes; user selects adequate battery |
| Storage filling up unexpectedly | Low | Medium | Monitor storage; alert user; large HDD; aggressive compression for context |
| Network unavailable for extended periods | Medium | Low | Local storage first; upload queue; don't depend on connectivity |

---

## 15. Design Principles Recap

1. **Bias toward capturing something over missing**: When in doubt, shoot
2. **Keep math/simple models over complex CV early**: Linear calibration > photogrammetry for MVP
3. **Build observability first**: Overlays, metadata, live view before automation
4. **Defer sophistication until system is proven**: Manual → Assisted → Automatic
5. **Treat calibration as first-class feature**: Make it easy, document it well
6. **Accept "good enough" over perfect**: Image quality matters more than geometric precision
7. **Plan for evolution**: Model upgrades, compute upgrades, feature additions

---

## 16. Future Enhancements (P2/P3)

### P2 (Near-term)

- **Primary camera HDMI ingestion**: Fine framing feedback from telephoto
- **Multi-context camera fusion**: Stationary wide cameras for better coverage
- **Painting-based semantic regions**: Mask sky vs trees vs ground for smarter tracking
- **Improved AF preconditioning**: Pre-focus based on expected subject distance
- **Motion trigger alternatives**: PIR sensor, optical flow detection
- **Laser trip beam**: Specific entry point monitoring

### P3 (Long-term)

- **Photogrammetry-based 3D scene model**: Better depth-aware tracking
- **Predictive tracking in world space**: Anticipate subject movement
- **Occlusion-aware tracking**: Handle subjects moving behind obstacles
- **Advanced framing heuristics**: Leading room, rule of thirds automatically
- **Automated culling**: AI-based selection of best shots
- **Species-specific behaviors**: Different strategies for birds vs mammals

---

## 17. Appendices

### Appendix A: Component Shopping List (MVP)

**Compute**:
- [ ] NVIDIA Jetson Orin Nano Developer Kit
- [ ] NVMe SSD 500GB-1TB (M.2 2280)
- [ ] SATA HDD 2-4TB (2.5" or 3.5" with USB adapter)

**Cameras**:
- [ ] Nikon Z9 (user-provided likely)
- [ ] Nikon lens: 600mm f/5.6 or 400mm f/2.8
- [ ] GoPro Hero 11/12 or similar

**Motion**:
- [ ] Linear rail system (6-8ft)
- [ ] Bi-pod stands (2x)
- [ ] Pan/tilt gimbal mechanism
- [ ] Closed-loop stepper motors (3x)
- [ ] Stepper drivers with encoder feedback (TMC2209/TMC5160 class)
- [ ] Slip ring (commercial, TBD model)
- [ ] Endstop switches (6x, 2 per axis)
- [ ] Motor controller MCU board (Arduino/STM32 with shields)

**Power**:
- [ ] DC-DC converters (12V/24V to 5V, 19V for Jetson)
- [ ] Power distribution block
- [ ] User-provided: Power station (2-4kWh)

**Networking**:
- [ ] Travel router (GL.iNet or similar)
- [ ] Starlink terminal (optional, user-provided)

**Environmental**:
- [ ] Weatherproof enclosure (IP65)
- [ ] Cable glands
- [ ] Ventilation/heating (if needed)

**Misc**:
- [ ] Game controller (Xbox/PlayStation compatible)
- [ ] HDMI display (portable monitor)
- [ ] Cables, connectors, mounting hardware

### Appendix B: Power Budget Calculator

```
System Average Power Draw: _____ W
Target Runtime: _____ hours
Required Battery Capacity: _____ Wh (W × hours × 1.5 safety factor)
Recommended Power Station: _____ kWh
```

### Appendix C: Calibration Worksheet

```
Calibration ID: _______________
Date: _______________
Location: _______________
Lens: _______________

Calibration Points:
| Point | Context X | Context Y | Pan Angle | Tilt Angle |
|-------|-----------|-----------|-----------|------------|
| 1     |           |           |           |            |
| 2     |           |           |           |            |
| 3     |           |           |           |            |
| 4     |           |           |           |            |
| 5     |           |           |           |            |

Fitted Model:
pan  = ___*x + ___*y + ___
tilt = ___*x + ___*y + ___

Validation:
Test point error: ___ degrees
Acceptable? [ ] Yes [ ] No
```

### Appendix D: XMP Metadata Implementation

**Libraries**:

**C++** (Recommended for AWIS Core):
- **Exiv2**: `https://github.com/Exiv2/exiv2`
  - Cross-platform, mature, supports NEF and JPG
  - XMP read/write with custom namespaces
  - Preserves existing EXIF data
  - Example:
  ```cpp
  Exiv2::Image::AutoPtr image = Exiv2::ImageFactory::open("image.nef");
  image->readMetadata();
  Exiv2::XmpData& xmpData = image->xmpData();
  xmpData["Xmp.awis.PanDegrees"] = 123.4;
  xmpData["Xmp.awis.AnimalClass"] = "eagle";
  image->writeMetadata();
  ```

**Python** (Utilities, Testing):
- **python-xmp-toolkit**: `https://github.com/python-xmp-toolkit/python-xmp-toolkit`
- **ExifTool wrapper**: `pyexiftool`

**Command Line** (Debugging):
- **ExifTool**: `exiftool -X image.nef` (dump all metadata)
- **ExifTool XMP write**: `exiftool -Xmp-awis:PanDegrees=123.4 image.nef`

**XMP Schema Definition**:

Register custom namespace with Adobe XMP Toolkit or Exiv2:
- Namespace URI: `http://awis.example.org/ns/1.0/`
- Preferred prefix: `awis`

**Validation**:
- Use Adobe XMP Schema Validator
- Lightroom: Import image → Metadata panel shows AWIS fields

### Appendix E: Configuration File Templates

See `/config/` directory in repository for JSON schemas.

**Note**: JSON configuration files are for system settings only. Per-image metadata is embedded via XMP (Appendix D).

---

*Document Version: 1.0*  
*Date: 2026-04-23*  
*Product: Autonomous Wildlife Imaging System (AWIS)*
