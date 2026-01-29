# Logitech RS50 Protocol Specification

**Document Version**: 4.3
**Date**: 2026-01-28
**Author**: Verified from USB capture analysis
**Status**: Working specification for Linux driver development

---

## 1. Device Identification

| Property | Value |
|----------|-------|
| Vendor ID | `0x046D` (Logitech) |
| Product ID | `0xC276` |
| Device Name | RS50 Base for PlayStation/PC |
| USB Version | 2.0 |
| Max Packet Size | 64 bytes |
| Device Class | HID (Human Interface Device) |

---

## 2. USB Interface Structure

The RS50 presents **3 HID interfaces** to the host:

### Interface 0 - Game Controller (Joystick)
| Property | Value |
|----------|-------|
| Endpoint | `0x81 IN` (Interrupt) |
| Report Size | 30 bytes |
| Interval | 1ms |
| Purpose | Wheel position, pedals, buttons |

### Interface 1 - HID++ Protocol
| Property | Value |
|----------|-------|
| Endpoint IN | `0x82 IN` (Interrupt) |
| Endpoint OUT | `0x00` (Control, SET_REPORT) |
| Report Size | 64 bytes |
| Purpose | Configuration, settings, feature queries |

### Interface 2 - Force Feedback
| Property | Value |
|----------|-------|
| Endpoint IN | `0x83 IN` (Interrupt, 64 bytes) |
| Endpoint OUT | `0x03 OUT` (Interrupt, 64 bytes) |
| Purpose | Real-time force feedback |

**Important**: FFB uses dedicated endpoint `0x03 OUT`, NOT the HID++ protocol.

---

## 3. Joystick Input Reports (Endpoint 0x81)

### Report Format (30 bytes) - VERIFIED

```
Offset  Size  Type      Description
------  ----  ----      -----------
0-1     2     uint16    Button bitmask (little-endian)
2-3     2     -         Reserved (always 0x0000)
4-5     2     uint16    Wheel position (little-endian)
6-7     2     uint16    Accelerator pedal (little-endian)
8-9     2     uint16    Brake pedal (little-endian)
10-11   2     uint16    Clutch pedal (little-endian)
12-17   6     -         Reserved (zeros)
18      1     uint8     Always 0x01
19-29   11    -         Reserved (zeros)
```

### Button Bitmask (4 bytes, offset 0-3) - VERIFIED

Button state is encoded in bytes 0-3 of the input report.

**Byte 0:**
| Bit | Mask   | Button |
|-----|--------|--------|
| 3   | `0x08` | Baseline (normally set, cleared for D-pad) |
| 4   | `0x10` | **A** |
| 5   | `0x20` | **X** |
| 6   | `0x40` | **B** |
| 7   | `0x80` | **Y** |

**Byte 1:**
| Bit | Mask   | Button |
|-----|--------|--------|
| 0   | `0x01` | **Right Paddle** |
| 1   | `0x02` | **Left Paddle** |
| 2   | `0x04` | **LT** (Left Trigger Button) |
| 3   | `0x08` | **RT** (Right Trigger Button) |
| 4   | `0x10` | **View** (Back/Select) |
| 5   | `0x20` | **Menu** (Start) |
| 6   | `0x40` | **LB** (Left Bumper) |
| 7   | `0x80` | **RB** (Right Bumper) |

**Byte 3:**
| Bit | Mask   | Button |
|-----|--------|--------|
| 7   | `0x80` | **G Button** (Logitech logo) |

### D-pad Encoding (Byte 0, bits 1-2)

When D-pad is pressed, byte 0 bit 3 (0x08 baseline) is **cleared** and bits 1-2 encode direction:

| Byte 0 Value | Direction |
|--------------|-----------|
| `0x00` | D-Right |
| `0x02` | D-Left |
| `0x04` | D-Up |
| `0x06` | D-Down |

**Detection**: D-pad pressed when `(byte0 & 0x08) == 0`

**Example:**
- Idle: `08 00 00 00` (baseline set)
- D-Up: `04 00 00 00` (baseline cleared, direction = 0x04)
- D-Left: `02 00 00 00` (baseline cleared, direction = 0x02)

### Wheel Position Encoding

| Value | Position |
|-------|----------|
| `0x0000` | Full left |
| `0x8000` | Center |
| `0xFFFF` | Full right |

Resolution depends on configured rotation range (90° to 2700°).

### Pedal Value Encoding

| Value | Position |
|-------|----------|
| `0x0000` | Released |
| `0xFFFF` | Fully pressed |

### Example Reports

```
Centered, pedals released:
08 00 00 00 00 80 00 00 00 00 00 00 00 00 00 00 00 00 01 00 00 00 00 00 00 00 00 00 00 00

Throttle full:
08 00 00 00 3C 80 FF FF 00 00 00 00 00 00 00 00 00 00 01 00 00 00 00 00 00 00 00 00 00 00
                  ^^^^^ accelerator = 0xFFFF

Brake partial (0x4847):
08 00 00 00 10 80 00 00 47 48 00 00 00 00 00 00 00 00 01 00 00 00 00 00 00 00 00 00 00 00
                        ^^^^^ brake = 0x4847
```

---

## 4. Force Feedback Output Commands (Endpoint 0x03 OUT)

### Standard FFB Report (64 bytes) - VERIFIED

```
Offset  Size  Type      Description
------  ----  ----      -----------
0       1     uint8     Report ID (0x01)
1-3     3     -         Reserved (0x00 0x00 0x00)
4       1     uint8     Effect type (0x01 = constant force)
5       1     uint8     Sequence counter (0x00-0xFF, wraps)
6-7     2     uint16    Force value (little-endian)
8-9     2     uint16    Force value duplicate (little-endian)
10-63   54    -         Padding (zeros)
```

**CRITICAL**: Sequence counter is a **single byte** that wraps at 255.

### Force Value Encoding (Offset Binary)

| Hex Value | Decimal | Force Direction |
|-----------|---------|-----------------|
| `0x0000` | 0 | Maximum LEFT |
| `0x4000` | 16384 | Half LEFT |
| `0x8000` | 32768 | NEUTRAL (no force) |
| `0xC000` | 49152 | Half RIGHT |
| `0xFFFF` | 65535 | Maximum RIGHT |

**Conversion from signed to offset binary**:
```c
uint16_t offset_binary = (int16_t)signed_value + 0x8000;
```

### FFB Enable/Refresh Command (64 bytes)

Sent periodically during gameplay (approximately every 30-60 seconds):

```
Offset  Size  Description
------  ----  -----------
0       1     Report ID (0x05)
1       1     Command type (0x07)
2-6     5     Reserved (zeros)
7-8     2     Value (0xFFFF = enabled?)
9-63    55    Padding (zeros)
```

Example: `05 07 00 00 00 00 00 FF FF 00 00 00 00 ...`

This command appears to refresh/maintain FFB state during gameplay.

### Example FFB Commands

```
Neutral (no force), sequence 0x7F:
01 00 00 00 01 7F 00 80 00 80 00 00 00 00 ... (54 zeros)

Maximum force LEFT, sequence 0x00:
01 00 00 00 01 00 00 00 00 00 00 00 00 00 ... (54 zeros)

Maximum force RIGHT, sequence 0x01:
01 00 00 00 01 01 FF FF FF FF 00 00 00 00 ... (54 zeros)
```

### Observed Behavior

- FFB update rate: ~250-500 Hz during gameplay
- Sequence counter increments with each command
- Force value and duplicate must always match
- `05 07` command sent periodically (~3 times per minute)

---

## 5. HID++ Protocol (Interface 1)

The RS50 uses HID++ 4.2 protocol for configuration.

### Message Formats

**Short Report (0x10)** - 7 bytes:
```
Byte 0: Report ID (0x10)
Byte 1: Device Index (0xFF = wired)
Byte 2: Feature Index
Byte 3: Function (high nibble) | SW ID (low nibble)
Bytes 4-6: Parameters
```

**Long Report (0x11)** - 20 bytes:
```
Byte 0: Report ID (0x11)
Byte 1: Device Index (0xFF)
Byte 2: Feature Index
Byte 3: Function | SW ID
Bytes 4-19: Parameters
```

**Very Long Report (0x12)** - 64 bytes:
```
Byte 0: Report ID (0x12)
Byte 1: Device Index (0xFF)
Byte 2: Feature Index
Byte 3: Function | SW ID
Bytes 4-63: Parameters
```

### ⚠️ CRITICAL: RS50 HID++ Report ID Behavior

**The RS50 has non-standard HID++ report handling:**

1. **G Hub sends SHORT reports (0x10)** for all commands, NOT long reports (0x11)
2. **RS50 ALWAYS responds with VERY LONG reports (0x12)** regardless of the input report type
3. Responses are 64 bytes even for simple queries

This is different from other Logitech HID++ devices which typically respond with the same report type they receive.

**Implication for drivers:**
- When sending 0x10 (short) or 0x11 (long), expect response on 0x12 (very long)
- The kernel driver must handle this asymmetric report ID behavior
- Feature discovery and settings queries work correctly when using 0x10 requests

**Verified via USB capture (2026-01-28):**
```
Host → Device: 10 ff 00 0b 81 38 00  (SHORT: query feature 0x8138)
Device → Host: 12 ff 00 0b 18 00 00... (VERY LONG: feature at index 0x18)
```

### HID++ Communication Method

Commands are sent via **USB Control Transfer (SET_REPORT)** to endpoint 0, NOT via interrupt OUT.
Responses arrive via **Interrupt IN** on endpoint 0x82.

```
Host → Device: SET_REPORT (Control, endpoint 0x00)
Device → Host: Interrupt IN (endpoint 0x82)
```

### Complete Feature Table (Verified)

| Index | Feature ID | Name | G Hub Setting |
|-------|------------|------|---------------|
| 0x00 | `0x0000` | IRoot | Feature discovery |
| 0x01 | `0x0001` | IFeatureSet | List all features |
| 0x02 | `0x0002` | IFeatureInfo | Feature metadata |
| 0x03 | `0x0003` | DeviceInfo | Device name, firmware |
| 0x04 | `0x00C3` | SecureDFU | Firmware update |
| 0x0A | `0x8040` | AxisResponseCurve | Sensitivity + **Brightness** |
| **0x0B** | **`0x807A`** | **LIGHTSYNC** | LED effect selection |
| 0x0D | `0x80A4` | AxisCalibration | Response curves |
| 0x0F | `0x8120` | FFBExtended | Internal FFB config |
| 0x10 | `0x8123` | ForceFeedback | HID++ FFB (unused) |
| **0x14** | **`0x8133`** | **Damping** | Damping slider |
| **0x15** | **`0x8134`** | **BrakeForce** | Brake Force slider |
| **0x16** | **`0x8136`** | **FFBStrength** | Strength slider |
| **0x17** | **`0x8137`** | **Profile** | Profile switching |
| **0x18** | **`0x8138`** | **RotationRange** | Rotation Range slider |
| **0x19** | **`0x8139`** | **TRUEFORCE** | TRUEFORCE slider |
| **0x1A** | **`0x8140`** | **FFBFilter** | FFB Filter + Auto toggle |

### Setting Commands (All Verified)

All commands use Short HID++ (0x10) with Software ID 0xD.

#### FFB Strength (Feature 0x8136, Index 0x16)
```
Set: 10 FF 16 2D [Value_Hi] [Value_Lo] 00
```
| Value | Percentage |
|-------|------------|
| `0x1FFF` | ~12% |
| `0x7FFF` | ~50% |
| `0xFFFF` | 100% |

#### Damping (Feature 0x8133, Index 0x14)
```
Set: 10 FF 14 1D [Value_Hi] [Value_Lo] 00
```
| Value | Percentage |
|-------|------------|
| `0x0000` | 0% |
| `0x7FFF` | ~50% |
| `0xFFFF` | 100% |

#### FFB Filter (Feature 0x8140, Index 0x1A)
```
Set: 10 FF 1A 2D [AutoFlag] 00 [Level]
```

**Auto Flag (Byte 4):**
| Value | Meaning |
|-------|---------|
| `0x00` | Auto FFB Filter OFF |
| `0x04` | Auto FFB Filter ON |

**Filter Level (Byte 6):**
| Value | Level |
|-------|-------|
| `0x01` | Minimum |
| `0x07` | Low |
| `0x0B` | Medium |
| `0x0F` | Maximum |

**Examples:**
- `10 FF 1A 2D 04 00 0B` = Auto ON, level 11
- `10 FF 1A 2D 00 00 0B` = Auto OFF, level 11

#### Rotation Range (Feature 0x8138, Index 0x18)
```
Set: 10 FF 18 2D [Degrees_Hi] [Degrees_Lo] 00
```
| Value | Degrees |
|-------|---------|
| `0x005A` | 90° |
| `0x0168` | 360° |
| `0x0384` | 900° |
| `0x0438` | 1080° |
| `0x0A8C` | 2700° |

#### TRUEFORCE (Feature 0x8139, Index 0x19)
```
Set: 10 FF 19 3D [Value_Hi] [Value_Lo] 00
```
| Value | Percentage |
|-------|------------|
| `0x0001` | Minimum |
| `0x4CCC` | ~30% |
| `0xFFFF` | 100% |

#### Brake Force (Feature 0x8134, Index 0x15)
```
Set: 10 FF 15 2D [Value_Hi] [Value_Lo] 00
```
| Value | Percentage |
|-------|------------|
| `0x028F` | ~1% |
| `0x4CCC` | ~30% |
| `0xFFFF` | 100% |

#### Sensitivity (Feature 0x8040, Index 0x0A)
```
Set: 10 FF 0A 2D 00 [Value] 00
```
| Value | Setting |
|-------|---------|
| `0x64` | 100 (default) |

*Note: Only available in Desktop profiles.*

#### LED Brightness (Feature 0x8040, Index 0x0A) - Same as Sensitivity
```
Set: 10 FF 0A 2D 00 [Value] 00
```
| Value | Brightness |
|-------|------------|
| `0x00` | 0% (off) |
| `0x32` | 50% |
| `0x64` | 100% |

*Note: Shares the same Feature Index as Sensitivity.*

#### LIGHTSYNC Effect (Feature 0x807A, Index 0x0B) - VERIFIED
```
Set: 10 FF 0B 3C [Effect] 00 00
```
| Effect | Name |
|--------|------|
| `0x01` | Inside→Out |
| `0x02` | Outside→In |
| `0x03` | Right→Left |
| `0x04` | Left→Right |
| `0x05` | Static |

#### Profile Switch (Feature 0x8137, Index 0x17)
```
Set: 10 FF 17 2D [ProfileIndex] 00 00
```
| Index | Profile |
|-------|---------|
| `0x00` | Desktop |
| `0x01+` | Built-in profiles |

---

## 6. Initialization Sequence

Recommended initialization order:

1. **USB Enumeration** - Get device descriptors
2. **Claim Interfaces** - Claim interfaces 0, 1, 2
3. **Feature Discovery** - Use IRoot (index 0x00) to find feature indices
4. **Read Settings** - Query current rotation, FFB gain
5. **Start FFB Loop** - Begin sending force commands at ~250-500 Hz
6. **Periodic Refresh** - Send `05 07` command every ~30 seconds

---

## 7. Driver Implementation Notes

### Linux Kernel Driver

1. **Multi-Interface Device**: Claim interface 2 for FFB, interface 0 for input
2. **FFB via hid_hw_output_report()**: Send 64-byte reports to interface 2
3. **Workqueue**: FFB must be sent from process context
4. **Custom FF Handler**: Implements FF_CONSTANT plus condition effects (FF_SPRING, FF_DAMPER, FF_FRICTION, FF_INERTIA)

### Condition Effects Implementation

The driver implements condition effects using a 250Hz timer:

- **FF_SPRING**: Force = coefficient × (position - center). Centering force.
- **FF_DAMPER**: Force = coefficient × velocity. Resists movement.
- **FF_FRICTION**: Force = coefficient × sign(velocity). Constant resistance.
- **FF_INERTIA**: Force = coefficient × acceleration. Resists speed changes.

Position history tracks wheel position over 4 samples to calculate velocity and acceleration.

### Settings Query on Init

The driver queries current device settings via HID++ on initialization:
- Rotation range, FFB strength, damping, TRUEFORCE, brake force
- FFB filter level and auto mode
- LED brightness

This ensures sysfs values reflect actual device state.

### C Structure Definition

```c
#define RS50_FF_REPORT_ID       0x01
#define RS50_FF_EFFECT_CONSTANT 0x01
#define RS50_FF_REPORT_SIZE     64

struct rs50_ff_report {
    u8 report_id;       /* 0x01 */
    u8 reserved[3];     /* 0x00, 0x00, 0x00 */
    u8 effect_type;     /* 0x01 = constant force */
    u8 sequence;        /* 0x00-0xFF, wraps */
    __le16 force;       /* 0x0000=left, 0x8000=center, 0xFFFF=right */
    __le16 force_dup;   /* duplicate of force value */
    u8 padding[54];     /* zeros */
} __packed;

static_assert(sizeof(struct rs50_ff_report) == RS50_FF_REPORT_SIZE,
              "RS50 FFB report structure size mismatch");
```

### Force Value Conversion

```c
static inline u16 rs50_force_to_offset_binary(s16 signed_force)
{
    return (u16)((s32)signed_force + 0x8000);
}
```

### Linux Sysfs Interface

The driver exposes settings via sysfs under `/sys/class/hidraw/hidrawX/device/`:

#### Wheel/FFB Settings
| Attribute | Range | Description |
|-----------|-------|-------------|
| `rs50_range` | 90-2700 | Rotation range in degrees |
| `rs50_strength` | 0-100 | FFB strength percentage |
| `rs50_damping` | 0-100 | Damping level percentage |
| `rs50_trueforce` | 0-100 | TRUEFORCE audio-haptic level |
| `rs50_brake_force` | 0-100 | Brake pedal load cell threshold |
| `rs50_ffb_filter` | 1-15 | FFB smoothing/filtering level |
| `rs50_ffb_filter_auto` | 0-1 | Auto FFB filter (0=off, 1=on) |
| `rs50_led_effect` | 1-5 | LIGHTSYNC effect (1=In→Out, 2=Out→In, 3=R→L, 4=L→R, 5=Static) |
| `rs50_led_brightness` | 0-100 | LED brightness percentage |

#### Pedal Response Curves & Combined Mode
| Attribute | Range/Format | Description |
|-----------|--------------|-------------|
| `rs50_combined_pedals` | 0-1 | Combined pedals mode (0=off, 1=on). When enabled, throttle axis outputs (throttle-brake) |
| `rs50_throttle_curve` | 0-2 | Throttle response curve: 0=linear, 1=low sensitivity, 2=high sensitivity |
| `rs50_brake_curve` | 0-2 | Brake response curve: 0=linear, 1=low sensitivity, 2=high sensitivity |
| `rs50_clutch_curve` | 0-2 | Clutch response curve: 0=linear, 1=low sensitivity, 2=high sensitivity |
| `rs50_throttle_deadzone` | "L U" | Throttle deadzone: L=lower%, U=upper% (e.g., "5 3" = 5% bottom, 3% top) |
| `rs50_brake_deadzone` | "L U" | Brake deadzone: L=lower%, U=upper% |
| `rs50_clutch_deadzone` | "L U" | Clutch deadzone: L=lower%, U=upper% |

**Response Curve Types:**
- **Linear (0)**: Direct 1:1 mapping, no transformation
- **Low Sensitivity (1)**: Less responsive at start, more responsive near full travel (quadratic)
- **High Sensitivity (2)**: More responsive at start, less responsive near full travel (square root)

**Usage Examples:**
```bash
# Read current rotation range
cat /sys/class/hidraw/hidraw*/device/rs50_range

# Set rotation to 900 degrees
echo 900 > /sys/class/hidraw/hidraw*/device/rs50_range

# Set FFB strength to 50%
echo 50 > /sys/class/hidraw/hidraw*/device/rs50_strength

# Set TRUEFORCE to 30%
echo 30 > /sys/class/hidraw/hidraw*/device/rs50_trueforce

# Set LED effect to Static (5)
echo 5 > /sys/class/hidraw/hidraw*/device/rs50_led_effect

# Enable auto FFB filter
echo 1 > /sys/class/hidraw/hidraw*/device/rs50_ffb_filter_auto

# Enable combined pedals mode (throttle - brake on single axis)
echo 1 > /sys/class/hidraw/hidraw*/device/rs50_combined_pedals

# Set throttle to high sensitivity curve
echo 2 > /sys/class/hidraw/hidraw*/device/rs50_throttle_curve

# Set brake to low sensitivity curve
echo 1 > /sys/class/hidraw/hidraw*/device/rs50_brake_curve

# Set throttle deadzone: 5% lower, 2% upper
echo "5 2" > /sys/class/hidraw/hidraw*/device/rs50_throttle_deadzone

# Set brake deadzone: 3% lower, 0% upper
echo "3 0" > /sys/class/hidraw/hidraw*/device/rs50_brake_deadzone
```

---

## 8. Differences from G920/G923

| Feature | G920/G923 | RS50 |
|---------|-----------|------|
| FFB Method | HID++ Feature 0x8123 | Dedicated endpoint 0x03 |
| Report ID | 0x11/0x12 | 0x01 (custom) |
| Sequence Field | 2 bytes | 1 byte |
| Max Rotation | 900° | 2700° |
| Motor Type | Belt/Gear | Direct drive |
| FFB Refresh | Not needed | `05 07` periodic |

---

## 9. Unknown/TODO

1. ~~**Auto FFB Filter toggle**~~ - VERIFIED: Byte 4 of FFB Filter command (0x04=on, 0x00=off)
2. ~~**LIGHTSYNC/LED settings**~~ - VERIFIED: Feature 0x0B (0x807A), brightness via 0x0A
3. ~~**Complete button mapping**~~ - VERIFIED: All 17 buttons mapped
4. ~~**D-pad encoding**~~ - VERIFIED: Byte 0 bits 1-2 when baseline cleared
5. ~~**Axis calibration curve format**~~ - IMPLEMENTED: Software-side response curves (linear/low/high sensitivity + deadzones)
6. ~~**Diagonal D-pad**~~ - IMPLEMENTED: Driver decodes all 8 directions
7. ~~**Combined pedals mode**~~ - IMPLEMENTED: Software-side pedal combining (throttle - brake)

### Diagonal D-pad Encoding (Implemented)

| Byte 0 Value | Direction |
|--------------|-----------|
| `0x00` | Right |
| `0x01` | Up-Right |
| `0x02` | Left |
| `0x03` | Up-Left |
| `0x04` | Up |
| `0x05` | Down-Right |
| `0x06` | Down |
| `0x07` | Down-Left |

---

## 10. Capture Files Reference

| File | Description |
|------|-------------|
| `rs50_ffb_game3.pcapng` | Gameplay FFB (~320k commands, includes `05 07`) |
| `2026-01-26_ghub_startup.pcapng` | G Hub init, feature enumeration |
| `2026-01-26_ffb_strength_sweep.pcapng` | FFB Strength slider |
| `2026-01-26_damping_sweep.pcapng` | Damping slider |
| `2026-01-26_ffb_filter_sweep.pcapng` | FFB Filter slider |
| `2026-01-26_rotation_sweep.pcapng` | Rotation Range slider |
| `2026-01-26_trueforce_sweep.pcapng` | TRUEFORCE slider |
| `2026-01-26_brake_force_sweep.pcapng` | Brake Force slider |
| `2026-01-26_profile_desktop.pcapng` | Profile switch + Sensitivity |
| `2026-01-26_wheel_input.pcapng` | Wheel position input |
| `2026-01-26_pedal_throttle.pcapng` | Throttle pedal input |
| `2026-01-26_pedal_brake.pcapng` | Brake pedal input |
| `2026-01-26_button_mapping.pcapng` | Button press bitmask mapping |
| `2026-01-26_lightsync.pcapng` | LED effect + brightness |
| `2026-01-26_auto_ffb_filter.pcapng` | Auto FFB Filter toggle |
| `2026-01-28_boot_no_ghub.pcapng` | RS50 boot WITHOUT G Hub - raw USB init |
| `2026-01-28_boot_with_ghub.pcapng` | RS50 boot WITH G Hub - full init sequence |
| `2026-01-28_ghub_init_wheel_on.pcapng` | G Hub init with wheel already powered |

---

## 11. Kernel Driver Implementation Notes

### RESOLVED: HID++ Feature Discovery Timeouts

**Problem (now fixed):** The Linux kernel HID++ driver experienced timeouts when querying RS50 feature indices during deferred initialization.

**Root Cause (identified):** The interrupt IN endpoint being disabled:
1. `hidpp_probe()` calls `hid_hw_open()` to enable the interrupt endpoint for protocol detection
2. After probe completes, it calls `hid_hw_close()` which **stops the interrupt IN endpoint**
3. Deferred init work (`rs50_ff_init_work`) runs 1 second later and sends HID++ commands
4. Commands go out via control endpoint (works), but responses arrive on the **stopped** interrupt endpoint
5. Driver never receives responses → 5-second timeout per query

**Solution (implemented):**
1. In `rs50_ff_init_work()`: Call `hid_hw_open()` to re-enable the interrupt endpoint before HID++ communication
2. Do NOT call `hid_hw_close()` after init - keep it open for runtime sysfs HID++ commands
3. In `rs50_ff_destroy()`: Call `hid_hw_close()` to balance the open count during cleanup
4. Track open state with `ff->hid_open` flag to ensure balanced open/close

**Why this works:** `hid_hw_open()` increments an open counter that keeps the interrupt IN endpoint active. By not closing after init, the endpoint stays open for any sysfs attribute writes that need to communicate with the device.

### RESOLVED: Module Unload Crash (kfree issue)

**Problem (now fixed):** Module unload (`rmmod`) crashed with use-after-free or memory corruption.

**Root Cause:** Dangling `input->ff->private` pointer:
1. The FF subsystem stores a back-reference in `input->ff->private` pointing to our `rs50_ff_data`
2. Interface 0 (joystick) owns the input device, interface 1 (HID++) owns our FF data
3. During removal, we freed our `ff` struct before the input device was fully cleaned up
4. If FF callbacks fired in the race window, they accessed freed memory

**Solution (implemented):**
```c
/* In rs50_ff_destroy(), BEFORE freeing ff: */
if (ff->input && ff->input->ff) {
    WRITE_ONCE(ff->input->ff->private, NULL);
}
```

This clears the back-reference before freeing, so any racing FF callbacks see NULL and exit safely.

### Historical Note: Report ID Behavior

The initial investigation suspected report ID mismatch (driver sending 0x11, expecting 0x11, getting 0x12). This was addressed but was NOT the root cause. The actual issue was the endpoint being stopped as described above.

### RESOLVED: HID Descriptor Button Overflow ("Invalid code 768")

**Problem (now fixed):** Kernel logged "Invalid code 768 type 1" through "Invalid code 777 type 1" messages during driver load.

**Root Cause:** The RS50 HID descriptor declares buttons 29-92 (64 buttons) but only ~20 physically exist on the wheel:
- Paddles: LB, RB
- Face buttons: A, B, X, Y, LT, RT, LSB, RSB, GL, GR
- Functional: Menu, Camera, G1
- Rotary encoder clicks: Left, Right
- D-pad: 4 directions

Linux input codes max out at 767 (0x2FF). Button 81+ overflow to codes 768+, which don't exist.

**Solution (implemented):** HID descriptor fixup in `rs50_report_fixup()`:
```c
/* Change Usage Maximum from 92 (0x5C) to 80 (0x50) */
if (rdesc[i+4] == 0x29 && rdesc[i+5] == 0x5c)
    rdesc[i+5] = 0x50;  /* Button max 92->80 */
```

**Note:** "Invalid code" messages may still appear briefly during manual `rmmod`/`insmod` cycles when hid-generic temporarily binds. This is cosmetic and doesn't affect functionality.

### Feature Discovery Sequence (from G Hub)

G Hub uses this sequence to discover features:
1. Protocol ping: `10 ff 00 1X 00 00 5A` → Response includes protocol version (4.2)
2. Feature query: `10 ff 00 0X HH LL 00` → Response byte 4 = feature index

Where `HH LL` is the 16-bit Page ID (e.g., 0x8138 for rotation range).

### Software ID (SW_ID) Behavior

The HID++ protocol uses a Software ID (SW_ID) in the lower nibble of the function byte to correlate requests with responses. G Hub uses SW_ID `0xB` (11), while the Linux driver uses SW_ID `0x01`.

**Observed RS50 behavior:**
- G Hub sends: `10 ff 00 0b 81 38 00` (function 0, SW_ID 11)
- RS50 responds: `12 ff 00 0c 18 00 00...` (function 0, SW_ID echoed as 12?)

The RS50 appears to echo the SW_ID in responses, though some HID++ 2.0 devices may leave it as 0. The driver should handle both cases by comparing only the function index (upper nibble) when SW_ID is 0.

### Alternative: FeatureSet Enumeration

Instead of querying ROOT.getFeatureIndex for each PAGE ID, G Hub uses FeatureSet (index 0x01) to enumerate all features:

```
Query FeatureSet.getFeatureID(index):
  10 ff 01 1X [index] 00 00
Response:
  12 ff 01 1X [pageID_hi] [pageID_lo] [type] ...
```

This returns the PAGE ID at each index. G Hub queries indices 0x00 through ~0x1F to build the complete feature map. This is more efficient than individual ROOT queries because:
1. Single function call per index (vs. searching for each PAGE)
2. Discovers all features, including unknown/undocumented ones
3. Faster overall enumeration

---

## 12. Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-01-26 | Initial specification |
| 1.1 | 2026-01-26 | Added HID++ feature table |
| 2.0 | 2026-01-26 | Complete rewrite: verified input format (30 bytes), documented `05 07` command, removed Swedish translations, added all setting commands |
| 2.1 | 2026-01-26 | Added button mapping (Y/X/A/B/LT/RT/LB/RB), LIGHTSYNC commands (Feature 0x0B), Auto FFB Filter toggle, LED brightness |
| 2.2 | 2026-01-26 | Complete button mapping: all 17 buttons (paddles, View, Menu, G button), D-pad encoding (byte 0 bits 1-2) |
| 3.1 | 2026-01-26 | Condition effects (FF_SPRING, FF_DAMPER, FF_FRICTION, FF_INERTIA), settings query on init, diagonal D-pad |
| 3.2 | 2026-01-26 | Combined pedals mode, pedal response curves (linear/low/high sensitivity), pedal deadzones |
| 4.0 | 2026-01-28 | **CRITICAL**: Documented RS50 HID++ report ID behavior - device sends 0x12 responses regardless of input report type. Added boot capture analysis. Documented kernel driver timeout root cause. |
| 4.1 | 2026-01-28 | Added SW_ID behavior documentation, FeatureSet enumeration method. Verified feature indices from USB capture analysis. |
| 4.2 | 2026-01-28 | **DRIVER WORKING**: Identified and fixed HID++ timeout root cause (hid_hw_close disabling endpoint). Fixed module unload crash (input->ff->private dangling pointer). Sysfs read/write fully functional. |
| 4.3 | 2026-01-28 | Fixed "Invalid code 768-777" kernel messages by patching HID descriptor (button max 92→80). Added complete PC mode button documentation (18 buttons + D-pad from PCGamingWiki). |
| 4.4 | 2026-01-29 | **RELEASE READY**: Fixed -1 refcount crash on rmmod (added get_device/put_device for cross-interface references). Fixed sysfs TOCTOU race (added stopping checks). Fixed pending_work counter leak. Fixed G920 memory leak and use-after-free. |
