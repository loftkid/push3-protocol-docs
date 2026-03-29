# Push 2 / Push 3 Protocol Compatibility

Push 2 has official Ableton documentation. Push 3 does not - the information here is based on reverse engineering.

This document covers where the two devices are compatible and where they differ. Useful if you have Push 2 code and want to port it, or want to support both devices in the same codebase.

---

## 1. USB Display

Display protocol is largely identical. The only meaningful difference is transfer chunk size.

### Hardware IDs

| | Push 2 | Push 3 |
|---|---|---|
| Vendor ID | `0x2982` | `0x2982` |
| Product ID | `0x1967` | `0x1969` |

### Display specs

Both devices are identical: 960x160px, RGB565 little-endian, 327,680 bytes per frame (16 header + 327,664 framebuffer), 2,048 bytes per line (1,920px data + 128 padding).

Frame header is the same on both:

```
FF CC AA 88 00 00 00 00 00 00 00 00 00 00 00 00
```

### Transfer performance

Push 3 uses a much larger chunk size, which cuts USB overhead significantly.

| | Push 2 | Push 3 |
|---|---|---|
| Chunk size | 512 bytes | 16,384 bytes |
| Chunks per frame | 640 | 20 |
| Transfer time | 50-100ms | 20-30ms |
| Bandwidth | ~3-6 MB/s | ~10-15 MB/s |

```python
PUSH2 = {'chunk_size': 512, 'target_fps': 15}
PUSH3 = {'chunk_size': 16384, 'target_fps': 30}
```

### Encryption

Both use the same XOR pattern on the framebuffer (header is unencrypted on both):

```python
XOR_PATTERN = (0xE7, 0xF3, 0xE7, 0xFF)

def encrypt_push_frame(data: bytes) -> bytes:
    buf = bytearray(data)
    for i in range(len(buf)):
        buf[i] ^= XOR_PATTERN[i % 4]
    return bytes(buf)
```

---

## 2. MIDI

Almost everything is identical. Same manufacturer ID, same SysEx format, same CC mappings for transport and encoders, same pad note numbers.

### Device identification

| | Push 2 | Push 3 |
|---|---|---|
| Manufacturer ID | `00 21 1D` | `00 21 1D` |
| Device family | `01 01` | `01 01` |
| SysEx format | `F0 00 21 1D 01 01 [CMD] [DATA] F7` | same |

### Pad mapping

Identical on both devices:

```python
PAD_MAPPING = {
    'bottom_left': 36,   # C1
    'bottom_right': 43,  # G1
    'top_left':    92,   # E6
    'top_right':   99,   # D#6
}
```

### Push 3 SysEx extensions

These commands exist on Push 3 but not Push 2. Not fully documented yet.

```python
PUSH3_EXTENSIONS = {
    0x38: 'RGB LED Control',
    0x3A: 'Aftertouch Mode',   # 0 = poly, 1 = channel
    0x3E: 'Unknown',
    0x43: 'Pad Velocity Curve (bulk)',
}
```

---

## 3. Pad Sensitivity Curves

This is the most significant protocol difference between the two devices.

Both use a 128-entry lookup table (LUT) to map pad pressure to MIDI velocity. Each entry is a 7-bit value (0-127). The LUT must be monotonic and plateau at 127.

### How they differ

| | Push 2 | Push 3 |
|---|---|---|
| Send method | 8 messages of 16 values each (`0x20`) | 1 bulk message, all 128 values (`0x43`) |
| Calibration | Separate `0x1B` message with `cpmin`/`cpmax` | Not needed, built into LUT |
| Complexity | Higher | Lower |

Push 2 needs explicit threshold calibration because its LUT does not encode the pressure range directly. Push 3 encodes everything into the LUT through its four parameters.

### Push 3 curve parameters

| Parameter | Range | Meaning |
|---|---|---|
| Threshold | 0-100 | Pads below this are always velocity 1 |
| Drive | -50 to +50 | Skews the curve (negative = softer, positive = harder) |
| Compand | -50 to +50 | S-curve shaping |
| Range | 0-100 | Pads above this are always velocity 127 |

### Push 3 LUT builder

```python
import math

def clamp(x, lo, hi):
    return max(lo, min(hi, x))

def bias(x, b):
    # Schlick bias
    return x / ((1/b - 2) * (1 - x) + 1)

def gain(x, g):
    # Schlick gain
    if x < 0.5:
        return 0.5 * bias(2*x, 1-g)
    return 1 - 0.5 * bias(2-2*x, 1-g)

def build_curve_push3(threshold, drive, compand, range_):
    Nt = round(clamp(threshold, 0, 100) * 16 / 100)   # leading 1s
    Nr = round(clamp(range_,    0, 100) * 103 / 100)  # trailing 127s

    Nt = min(Nt, 127)
    Nr = min(Nr, 127 - Nt)
    num_curve = max(2, 128 - Nt - Nr + 2)

    b = clamp(0.5 + drive   / 100, 0.01, 0.99)
    g = clamp(0.5 + compand / 100, 0.01, 0.99)

    curve = [0] * 128

    for i in range(Nt):
        curve[i] = 1

    for i in range(num_curve):
        x = i / (num_curve - 1)
        y = bias(gain(x, g), b)
        v = int(round(1 + y * 126))
        curve[max(0, Nt + i - 1)] = min(127, max(1, v))

    for i in range(128 - Nr, 128):
        curve[i] = 127

    for i in range(1, 128):
        if curve[i] < curve[i-1]:
            curve[i] = curve[i-1]

    return curve

def validate_curve(curve):
    assert len(curve) == 128
    assert all(0 <= v <= 127 for v in curve)
    for i in range(1, 128):
        assert curve[i] >= curve[i-1]
    idx = next((i for i, v in enumerate(curve) if v == 127), None)
    if idx is not None:
        assert all(v == 127 for v in curve[idx:])
```

### Sending the curve

**Push 3** - single bulk SysEx:

```python
def send_curve_push3(curve, midi_out):
    msg = [0xF0, 0x00, 0x21, 0x1D, 0x01, 0x01, 0x43, *curve, 0xF7]
    midi_out.send(mido.Message('sysex', data=msg[1:-1]))
```

**Push 2** - 8 messages + calibration:

```python
def split_14bit(v):
    return v & 0x7F, (v >> 7) & 0x1F

def send_curve_push2(curve, midi_out, cpmin=1650, cpmax=2050):
    # Curve in 8 chunks of 16
    for start in range(0, 128, 16):
        payload = [0x20, start, *curve[start:start+16]]
        msg = [0xF0, 0x00, 0x21, 0x1D, 0x01, 0x01, *payload, 0xF7]
        midi_out.send(mido.Message('sysex', data=msg[1:-1]))

    # Calibration message
    data = [0x1B]
    for val in (33, 31, cpmin, cpmax):
        L, H = split_14bit(val)
        data += [L, H]
    msg = [0xF0, 0x00, 0x21, 0x1D, 0x01, 0x01, *data, 0xF7]
    midi_out.send(mido.Message('sysex', data=msg[1:-1]))
```

---

## 4. Porting Push 2 Code to Push 3

In most cases: swap the chunk size, done. MIDI mappings, SysEx format, pad notes, encoder CCs - all identical.

Things that actually need changes:

- Chunk size: 512 -> 16,384
- Pad curves: replace the 8-message curve + calibration flow with a single `0x43` bulk message
- Push 3-specific features (RGB LEDs, aftertouch mode) require new SysEx commands - see extensions table above

```python
import time
import mido

FRAME_HEADER = bytes.fromhex('FF CC AA 88 00 00 00 00 00 00 00 00 00 00 00 00')

DEVICE_CONFIG = {
    'push2': {'chunk_size': 512,   'target_fps': 15},
    'push3': {'chunk_size': 16384, 'target_fps': 30},
}

def send_frame(device, frame_data, device_type='push3'):
    cfg = DEVICE_CONFIG[device_type]
    frame_budget = 1.0 / cfg['target_fps']
    t_start = time.perf_counter()

    device.write(0x01, FRAME_HEADER)
    data = encrypt_push_frame(frame_data)
    chunk = cfg['chunk_size']
    for i in range(0, len(data), chunk):
        device.write(0x01, data[i:i+chunk])

    elapsed = time.perf_counter() - t_start
    remaining = frame_budget - elapsed
    if remaining > 0:
        time.sleep(remaining)
```

Cheers.
