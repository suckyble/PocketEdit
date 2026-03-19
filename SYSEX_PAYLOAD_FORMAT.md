# Pocket Master SysEx Payload Format — Decoded

## 1. Full Message Structure (with BLE Header)

Every SysEx message over BLE is prefixed with a 2-byte MIDI timestamp (`8080`). Over USB, this header is stripped. The hex string is **nibble-expanded**: each logical byte `0xAB` is transmitted as two hex byte-pairs `0A 0B`.

```
[80] [80] [F0] [CRC_H] [CRC_L] [TP_H] [TP_L] [PN_H] [PN_L] [LEN_H] [LEN_L] [PAYLOAD...] [F7]
 ↑    ↑    ↑     ↑       ↑       ↑       ↑       ↑       ↑      ↑       ↑         ↑          ↑
 BLE timestamp  SysEx  CRC-8/SMBus   Total    Current   Payload    Command/      SysEx
              start   (nibble-exp.)  Packets   Packet    Length    Status Data     end
```

| Hex Position | Byte Index | Field             | Description                                                      |
|--------------|------------|-------------------|------------------------------------------------------------------|
| 0–3          | 0–1        | BLE Header        | Always `8080`. Stripped for USB transport.                       |
| 4–5          | 2          | SysEx Start       | Always `F0`.                                                     |
| 6–9          | 3–4        | CRC-8/SMBus       | Nibble-expanded CRC. Covers bytes from [5] to end of payload.   |
| 10–13        | 5–6        | Total Packets     | Nibble-expanded. `0001` = single packet. Decoded: `(b5 << 4) \| b6`. |
| 14–17        | 7–8        | Current Packet    | Nibble-expanded. `0000` = first (0-indexed).                    |
| 18–21        | 9–10       | Payload Length     | Nibble-expanded. Number of payload bytes that follow.            |
| 22+          | 11+        | Payload Data       | Variable length. Contains the command/response body.             |
| Last 2       | Last       | SysEx End          | Always `F7`.                                                     |

### CRC Calculation

- **Algorithm**: CRC-8/SMBus PEC (polynomial `0x07`, seed `0x00`)
- **Input**: The nibble-expanded hex string from byte [5] through end of payload, after removing the extra zero-nibbles (via regex `0(.)/g → '$1'`)
- **Result**: Single byte, then nibble-expanded back (e.g., `0x3A` → `030A`)

---

## 2. Payload Structure

The payload (starting at hex position 22) is itself structured. Based on exhaustive analysis of `COMMAND_LIBRARY_DATA`:

```
Payload = [FIXED] [FIXED] [CMD_TYPE] [MODULE_DATA...]
```

| Payload Byte Offset (hex chars from pos 22) | Field           | Description                         |
|----------------------------------------------|-----------------|-------------------------------------|
| 0–3 (chars 22–25)                            | Fixed prefix    | Always `0101`                       |
| 4–7 (chars 26–29)                            | Command Type    | `0407`, `0408`, `0409`, `0201`, etc. |
| 8+ (chars 30+)                               | Command Body    | Variable, depends on command type   |

---

## 3. Command Types

### 3.1 `0409` — Module State (On/Off) + Amp Mode

Controls the enable/disable state of effect modules and the factory/clone amp mode toggle.

**Full payload layout:**

```
0101 0409 [MOD] [00] [00] [00] [00] [00] [00] [00] [VAL] [00] [00] [00] [00] [00] [00]
│         │                                          │
│         └─ Module ID (nibble-expanded)             └─ State: 01=ON, 00=OFF
└─ Fixed prefix + command type
```

| Payload Char Offset | Nibble-Expanded Bytes | Field        | Values                                      |
|---------------------|-----------------------|--------------|---------------------------------------------|
| 0–3                 | `0101`                | Fixed        | Always `0101`                               |
| 4–7                 | `0409`                | Command      | Module State command                        |
| 8–11                | `00XX`                | Module ID    | 00–08 for NR/FX1/DRV/AMP/IR/EQ/FX2/DLY/RVB; 09 for Clone/AmpMode |
| 12–25               | 7 × `00`             | Reserved     | Always zero                                 |
| 26–29               | `00XX`                | Value        | `01` = ON/Clone mode, `00` = OFF/Factory mode |
| 30–41               | 6 × `00`             | Padding      | Always zero                                 |

**Example**: Module 0 (NR) ON:
```
8080 f0 01 0c 00 01 00 00 00 0a 0101 0409 0000 0000000000000000 01 000000000000 f7
       CRC       TP    PN   LEN  fix  cmd  mod=0  (7 zero bytes)  ON   (padding)
```

**Module ID Reference:**

| ID | Module        | Description       |
|----|---------------|-------------------|
| 0  | NR            | Noise Reduction   |
| 1  | FX1           | Effects 1         |
| 2  | DRV           | Drive             |
| 3  | AMP           | Amplifier         |
| 4  | IR            | Impulse Response  |
| 5  | EQ            | Equalizer         |
| 6  | FX2           | Effects 2         |
| 7  | DLY           | Delay             |
| 8  | RVB           | Reverb            |
| 9  | (Amp Mode)    | Factory/Clone toggle |

---

### 3.2 `0407` — Effect Type Selection

Selects which effect algorithm is active within a module slot.

**Full payload layout:**

```
0101 0407 [MOD] [00] [00] [00] [00] [00] [00] [00] [MOD2] [00] [00] [00] [00] [00] [00] [SIG_B1] [SIG_B2] [00] [00] [00] [00] [00] [CAT]
│         │                                          │                                     │                                           │
│         └─ Module ID                               └─ Module ID (repeated)               └─ Effect Signature (2-byte ID)             └─ Category/Variant byte
└─ Fixed prefix + command type
```

| Payload Char Offset | Field               | Description                                                    |
|---------------------|---------------------|----------------------------------------------------------------|
| 0–3                 | `0101`              | Fixed prefix                                                  |
| 4–7                 | `0407`              | Effect Type command                                           |
| 8–11                | Module ID           | Target module (nibble-expanded)                               |
| 12–25               | Reserved            | 7 × zero bytes                                                |
| 26–29               | Module ID (echo)    | Same module ID repeated                                       |
| 30–43               | Reserved            | 7 × zero bytes                                                |
| 44–47               | Effect Signature H  | High byte of effect identifier (nibble-expanded)              |
| 48–51               | Effect Signature L  | Low byte of effect identifier                                 |
| 52–63               | Reserved            | Zero bytes                                                    |
| 64–67               | Category Byte       | Effect category/variant indicator                             |

**Length note**: `0407` commands use payload length `0E` (14 bytes) vs `0A` (10 bytes) for `0409`.

**Verification — Module 2 (DRV), Effect 201 (Scream):**
```
8080f0 0f05 0001 0000 000A 0101 0407 0002 00000000000000 00 000000000003 f7
                            cmd  mod=2                                   cat=03
```
The short payload (length `0A`) with `0003` at the end indicates Scream (DRV category=03, no extra signature bytes needed since the default effect is selected).

**Verification — Module 1 (FX1), Effect 101 (COMP 1):**
```
8080f0 0c05 0001 0000 000e 0101 0407 0001 00000000000000 0100000000000000 0000000000 00 f7
                            cmd  mod=1                    mod=1(echo)       sig=0000    cat=00
```

**Effect Signature Patterns (derived from `COMMAND_LIBRARY_DATA.effectTypes`):**

The effect identity within a module is encoded as a combination of the "signature bytes" and "category byte" at the end of the payload. Each effect type has a unique combination. The exact encoding varies by module type:

| Module | Signature bytes | Category byte | Encoding style |
|--------|----------------|---------------|----------------|
| DRV (2)  | 2 bytes at offset 44–47 | 1 byte at end | `{sig_h}{sig_l}...{cat}` |
| AMP (3)  | 4 bytes at offset 44–51 | 1 byte at end | Multi-byte signature |
| IR (4)   | 4 bytes at offset 44–51 + extra | 1 byte at end | Multi-byte with variant |
| FX1/FX2  | 4 bytes at offset 44–51 | 1 byte at end | Includes sub-variant |
| DLY (7)  | 2 bytes at offset 44–47 | 1 byte at end | Simple 2-byte ID |
| RVB (8)  | 2 bytes at offset 44–47 | 1 byte at end | Simple 2-byte ID |

---

### 3.3 `0408` — Parameter Value Set

Sets a parameter value for the currently active effect in a module.

**Full payload layout:**

```
0101 0408 [MOD] [00] [00] [00] [00] [00] [00] [00] [ALGID] [00] [00] [00] [00] [00] [00] [VALUE_BYTES (6 nibble-expanded bytes)]
│         │                                          │                                      │
│         └─ Module ID                               └─ Algorithm/Parameter ID              └─ Encoded parameter value
└─ Fixed prefix + command type
```

| Payload Char Offset | Field               | Description                                                     |
|---------------------|---------------------|-----------------------------------------------------------------|
| 0–3                 | `0101`              | Fixed prefix                                                   |
| 4–7                 | `0408`              | Parameter Value command                                        |
| 8–11                | Module ID           | Target module (nibble-expanded)                                |
| 12–25               | Reserved            | 7 × zero bytes                                                 |
| 26–29               | Algorithm ID (AlgID)| Which parameter within the effect (nibble-expanded)            |
| 30–43               | Reserved            | 7 × zero bytes                                                 |
| 44–55               | Value encoding      | 6 nibble-expanded bytes representing the parameter value       |

**Parameter Value Encoding:**

Parameter values are NOT stored as plain integers. They use a **lookup-table encoding** where each value (0, 0.1, 0.2, ... 100) maps to a specific 6-byte pattern. Examples from Module 0 (NR), AlgID 0:

| Param Value | Encoded 6 bytes (at payload offset 44–55) | Notes                    |
|-------------|-------------------------------------------|--------------------------|
| 0           | `000000000000`                            | Zero = all zeros         |
| 1           | `000000000800030F`                        | Integer value 1          |
| 2           | `000000000000040 0`                       | Integer value 2          |
| 10          | `000000000000040 1` (approx)              | Uses 0401 suffix series  |
| 100         | `00000000000C080402`                      | Maximum value (for some) |

For Module 1 (FX1), AlgID 0, sub-integer values exist (e.g., `0.1`, `0.5`, `1.3`), and each has a unique 12-char hex encoding in the last portion of the message. The 6-byte value block appears to use a non-linear encoding scheme decoded via `eqReverseLookup` table in the code.

**Verification — Module 0, AlgID 0, Value 0:**
```
8080F0 0D02 0001 0000 000E 0101 0408 0000 00000000000000 0000 00000000000000 0000000000 0000 F7
                             cmd  mod=0                   aid=0               value=all zeros
```

**Verification — Module 1, AlgID 0, Value 0:**
```
8080f0 080F 0001 0000 000E 0101 0408 0001 00000000000000 0000 00000000000000 0000000000 0000 f7
                             cmd  mod=1                   aid=0               value=all zeros
```

**AlgID meanings per module (from `ALGID_LOCATION_MAP` and effect library):**

| Module | AlgID | Common Parameter Name |
|--------|-------|----------------------|
| NR (0) | 0     | Threshold (THRE)     |
| FX1 (1)| 0–5   | Varies by active effect (e.g., Sense, Range, Q, Mix, Mode, Level) |
| DRV (2)| 0–4   | Gain, Tone, Volume, etc. |
| AMP (3)| 0–6   | Gain, Bass, Mid, Treble, Presence, Master, etc. |
| IR (4) | 0     | Volume (VOL)         |
| EQ (5) | 0–5   | 6-band EQ frequencies |
| FX2 (6)| 0–4   | Same as FX1 param structure |
| DLY (7)| 0–4   | Mix, Time, Feedback, Tone, (Mod) |
| RVB (8)| 0–5   | Mix, Decay, Tone, Pre-delay, etc. |
| Clone(9)| 0–4  | Clone-specific params |

---

### 3.4 `0201` / `0202` — Request Commands (Device Queries)

These are sent TO the device to request data. They use a different command prefix:

```
0101 [PAYLOAD_AREA...] → for data commands (0407/0408/0409)
0201 [REQUEST_TYPE]    → for query commands
```

| Full SysEx Message                                   | Request Type | Purpose                            |
|------------------------------------------------------|--------------|------------------------------------|
| `8080F0000E00010000000201020400F7`                   | `020400`     | Request all preset names (full dump)|
| `8080F0000700010000000201020403F7`                   | `020403`     | Request current preset number      |
| `8080F0000900010000000201020401F7`                   | `020401`     | Request current preset state (5-packet dump) |
| `8080F0020900010000000201020200F7`                   | `020200`     | Request User IR names              |
| `8080F0030500010000000201020204F7`                   | `020204`     | Request Clone Amp names            |
| `8080F00B0900010000000201020100F7`                   | `020100`     | Request global settings            |
| `8080F0010500010000000201020405F7`                   | `020405`     | Request preset modified status     |

**Response routing**: Responses don't echo the `0201` prefix. Instead, the firmware replies with content-specific packets, and the app routes them by matching signatures like `01020204` (Clone names) or `01020200` (IR names), or by using state flags set before the request was sent.

---

### 3.5 `0301` — Response/ACK Commands (from Device)

| Full SysEx Message                                    | Meaning                    |
|-------------------------------------------------------|----------------------------|
| `8080F00B02000100000003010400080000F7`                | Success ACK                |
| `8080F0070A000100000003010204050001F7`                | Preset Modified notification|
| `8080F0070D000100000003010204050000F7`                | Preset Saved notification  |

---

### 3.6 `040A` — Save Preset Command

Used in the save flow to write current preset state to a slot:

```
Payload template: 0001000001030101040a{PRESET_BYTES}000000000000{ENCODED_NAME}000000000000
```

| Field          | Description                                         |
|----------------|-----------------------------------------------------|
| `0101040A`     | Save preset command type                            |
| Preset Bytes   | Nibble-expanded preset index (0-49)                 |
| Encoded Name   | 10 characters × 4 hex chars each = 40 chars        |

---

## 4. User's Example Message — Full Decode

```
8080f0080100010000000a0101040900090000000000000000000000000000f7
```

Parsed byte-by-byte:

| Hex Chars  | Value    | Field             | Decoded Meaning                   |
|------------|----------|-------------------|-----------------------------------|
| `8080`     | —        | BLE Header        | MIDI timestamp (ignored)          |
| `f0`       | 0xF0     | SysEx Start       | —                                 |
| `0801`     | 0x81     | CRC-8/SMBus       | Checksum (nibble-expanded 0x81)   |
| `0001`     | 1        | Total Packets     | 1 packet total                    |
| `0000`     | 0        | Current Packet    | Packet 0 (first)                  |
| `000a`     | 10       | Payload Length     | 10 bytes of payload               |
| `0101`     | —        | Fixed Prefix      | Always `0101`                     |
| `0409`     | —        | Command Type      | **Module State** command          |
| `0009`     | 9        | Module ID         | **Amp Mode** (module 9)           |
| `0000000000000000` | — | Reserved      | 7 zero bytes                      |
| `00`       | 0        | Value             | **0 = Factory mode**              |
| `000000000000` | —   | Padding           | 6 zero bytes                      |
| `f7`       | 0xF7     | SysEx End         | —                                 |

**Result**: This message **sets the Amp Mode to Factory** (as opposed to Clone mode). This matches `COMMAND_LIBRARY_DATA.ampModes.factory`.

---

## 5. Summary Table — Command Type Quick Reference

All payload offsets below are **hex character offsets** from position 22 (start of payload) in the full BLE hex string. Payload Length (PL) is the **logical** byte count found in the header (transmitted bytes = PL × 2 due to nibble expansion).

### 5.1 Outgoing Data Commands (App → Device)

All prefixed with `0101` at payload offset 0–3. Command type at offset 4–7.

| Command | PL       | Purpose              | Key Payload Fields                                            |
|---------|----------|----------------------|---------------------------------------------------------------|
| `0101`  | `0A` (10)| Global Setting       | ParamGroup @ +8, ParamSubID @ +12, Value @ +24               |
| `0402`  | `07` (7) | Preset Volume        | Fixed `00010200 00010000` @ +8, Value(NE) @ +24              |
| `0403`  | `06` (6) | Preset Select        | PresetIndex(NE) @ +8, Padding                                |
| `0404`  | `0C` (12)| Chain Order          | 9 × ModuleID(NE) @ +8 through +44, fixed `0009` suffix      |
| `0407`  | `0E` (14)| Effect Type Select   | Module ID @ +8, Effect Signature @ +44, Category @ end       |
| `0408`  | `0E` (14)| Parameter Value      | Module ID @ +8, AlgID @ +26, Value @ +44                     |
| `0409`  | `0A` (10)| Module On/Off        | Module ID @ +8, Value (01/00) @ +26                          |
| `040A`  | `13` (19)| Save Preset          | PresetIndex(NE) @ +8, EncodedName(40ch) @ +20                |

### 5.2 Outgoing Request Commands (App → Device)

All prefixed with `0102` at payload offset 0–3. PL = `02` (2 logical bytes → 4 transmitted). Request type encoded in the remaining 4 bytes.

| Full Payload (hex) | Purpose                        | Trigger                              |
|---------------------|-------------------------------|--------------------------------------|
| `0102 0100`         | Request Global Settings       | On connect / settings panel open     |
| `0102 0200`         | Request User IR Names         | On connect                           |
| `0102 0204`         | Request Clone Amp Names       | On connect                           |
| `0102 0400`         | Request All Preset Names      | Sync button / on connect             |
| `0102 0401`         | Request Current Preset State  | After preset select / on connect     |
| `0102 0403`         | Request Current Preset Number | After name sync completes            |
| `0102 0405`         | Request Preset Modified Status| Periodic check                       |

### 5.3 Incoming Responses & Notifications (Device → App)

PL = `03` (3 logical bytes → 6 transmitted).

| Full Payload (hex)   | Purpose                       | Routing                                   |
|-----------------------|-------------------------------|--------------------------------------------|
| `0104 0008 0000`     | Success ACK                   | Resolves `ackPromise`                      |
| `0102 0405 0001`     | Preset Modified Notification  | Matched by exact hex string                |
| `0102 0405 0000`     | Preset Saved Notification     | Matched by exact hex string                |

Multi-packet responses (preset state dump, name dump, global settings) are routed by state flags (`isWaitingForPresetDump`, `isSyncing`, `isWaitingForGlobals`), not by payload signature.

### 5.4 Protocol-Level Signals

| Full SysEx Message (hex)                        | Purpose                     |
|-------------------------------------------------|-----------------------------|
| `8080F0030601050104000200000000F7`              | Name dump terminator (last of ~21 packets, empty payload) |

---

## 6. Detailed Decode — New Command Types

### 6.1 `0404` — Chain Order

Sets the signal chain order of the 9 effect modules.

```
0101 0404 [M0] [M1] [M2] [M3] [M4] [M5] [M6] [M7] [M8] [0009]
│         │                                                 │
│         └─ 9 Module IDs in chain order (NE)               └─ Fixed suffix (module 9 / amp-mode)
└─ Fixed prefix + command type
```

| Payload Offset | Field              | Description                                              |
|----------------|--------------------|----------------------------------------------------------|
| 0–3            | `0101`             | Fixed prefix                                             |
| 4–7            | `0404`             | Chain Order command                                      |
| 8–11           | Module slot 1      | First module in chain (nibble-expanded module ID)        |
| 12–15          | Module slot 2      | Second module in chain                                   |
| 16–19          | Module slot 3      | Third module in chain                                    |
| 20–23          | Module slot 4      | Fourth module in chain                                   |
| 24–27          | Module slot 5      | Fifth module in chain                                    |
| 28–31          | Module slot 6      | Sixth module in chain                                    |
| 32–35          | Module slot 7      | Seventh module in chain                                  |
| 36–39          | Module slot 8      | Eighth module in chain                                   |
| 40–43          | Module slot 9      | Ninth module in chain                                    |
| 44–47          | `0009`             | Fixed suffix (always module 9)                           |

**Example**: Chain order NR→FX1→FX2→DLY→RVB→DRV→AMP→IR→EQ:
```
8080f0 000C 0001 0000 000c 0101 0404 0000 0001 0006 0007 0008 0002 0003 0004 0005 0009 f7
                             fix  cmd  NR   FX1  FX2  DLY  RVB  DRV  AMP  IR   EQ   (9)
```

### 6.2 `0101` — Global Setting

Sets a global device parameter (not tied to a specific effect/module).

```
0101 0101 [GRP] [SUB] [0000] [0000] [VALUE (8 hex chars)]
│         │      │                    │
│         │      └─ Parameter Sub-ID  └─ Nibble-expanded value (encoding varies by param)
│         └─ Parameter Group
└─ Fixed prefix + command type
```

| Payload Offset | Field          | Description                                          |
|----------------|----------------|------------------------------------------------------|
| 0–3            | `0101`         | Fixed prefix                                         |
| 4–7            | `0101`         | Global Setting command                               |
| 8–11           | Param Group    | Nibble-expanded group ID                             |
| 12–15          | Param Sub-ID   | Nibble-expanded sub-parameter ID                     |
| 16–23          | Reserved       | 4 × zero bytes                                       |
| 24–39          | Value          | Nibble-expanded value (8 bytes, encoding depends on param) |

**Global Parameter Addressing:**

| Group | Sub-ID | Parameter        | Value Range   | Unit |
|-------|--------|------------------|---------------|------|
| `02`  | `02`   | Global Volume    | 0–100         | —    |
| `01`  | `03`   | Input Level      | -20 to +20    | dB   |
| `01`  | `04`   | FX Rec Level     | -20 to +20    | dB   |
| `02`  | `04`   | Monitor Level    | -20 to +20    | dB   |
| `05`  | `04`   | BT Rec Level     | -20 to +20    | dB   |

**Example**: Global Volume = 0:
```
8080f0 0c0b 0001 0000 000a 0101 0101 0002 0002 0000 0000 0000 0000 0000 0000 f7
                             fix  cmd  grp  sub  (reserved)  (value = 0)
```

**dB value encoding**: Negative values use a special encoding with `0F0F` patterns (e.g., inputLevel = -1 → `0f0f 0f0f 0f0f 0f0f`). Positive and zero values use standard nibble-expanded integers in the value field.

### 6.3 `0402` — Preset Volume

Sets the volume level for the currently loaded preset (independent of global volume).

```
0101 0402 0001 0200 0001 0000 [VALUE (4 hex chars)]
│         │                    │
│         └─ Fixed routing     └─ Nibble-expanded volume value (0–100)
└─ Fixed prefix + command type
```

| Payload Offset | Field          | Description                                          |
|----------------|----------------|------------------------------------------------------|
| 0–3            | `0101`         | Fixed prefix                                         |
| 4–7            | `0402`         | Preset Volume command                                |
| 8–15           | `00010200`     | Fixed routing bytes                                  |
| 16–23          | `00010000`     | Fixed routing bytes                                  |
| 24–27          | Value          | Nibble-expanded volume (0x00–0x64 → `0000`–`0604`)  |

**Example**: Preset Volume = 100 (0x64):
```
8080f0 ... 0001 0000 0007 0101 0402 0001 0200 0001 0000 0604 f7
                                cmd              (routing)     val=0x64=100
```

### 6.4 `0403` — Preset Select

Loads a user or factory preset by index.

```
0101 0403 [IDX_H] [IDX_L] [0000] [0000] [0000] [00]
│         │
│         └─ Nibble-expanded preset index
└─ Fixed prefix + command type
```

| Payload Offset | Field          | Description                                          |
|----------------|----------------|------------------------------------------------------|
| 0–3            | `0101`         | Fixed prefix                                         |
| 4–7            | `0403`         | Preset Select command                                |
| 8–11           | Preset Index   | Nibble-expanded index (see table below)              |
| 12–23          | Padding        | Zero bytes                                           |

**Preset Index Mapping:**

| Preset   | Index (dec) | Index (hex) | Nibble-Expanded |
|----------|-------------|-------------|-----------------|
| P01      | 0           | 0x00        | `0000`          |
| P10      | 9           | 0x09        | `0009`          |
| P17      | 16          | 0x10        | `0100`          |
| P50      | 49          | 0x31        | `0301`          |
| F01      | 50          | 0x32        | `0302`          |
| F50      | 99          | 0x63        | `0603`          |

User presets P01–P50 = indices 0–49. Factory presets F01–F50 = indices 50–99.

### 6.5 `040A` — Save Preset

Writes the current device state to a preset slot with a name.

```
0101 040A [IDX] 000000000000 [ENCODED_NAME (40 chars)] 000000000000
│         │     │              │                         │
│         │     └─ Reserved    └─ 10 chars × 4 hex each  └─ Padding
│         └─ Nibble-expanded preset index (0–49)
└─ Fixed prefix + command type
```

Name encoding uses `CHARACTER_MAP`: each character → 4 hex chars (e.g., `A` → `0401`, `0` → `0300`, space → `0200`, null/pad → `0000`).

---

## 7. Device Response Format — Short `0407` Messages

### Observation

When the device **sends back** `0407` (Effect Type Selection) messages (e.g. after a preset change or when reporting current state), it uses a **short format** (payload length `0A` = 10 bytes, 40 payload hex chars) instead of the **long format** (payload length `0E` = 14 bytes, 56 payload hex chars) used in the stored `COMMAND_LIBRARY_DATA`.

The short format omits the "module ID echo" block (the repeated module ID and its surrounding reserved bytes at payload offsets 26–43), keeping only the module ID, the effect signature/category tail, and the fixed prefix/command fields.

### Long format (sent to device / stored in COMMAND_LIBRARY_DATA)

```
0101 0407 [MOD] [00×7] [MOD2] [00×7] [SIG_H] [SIG_L] [00×5] [CAT]
          ^^^^          ^^^^^^         ^^^^^^^^^^^^^^^^        ^^^^
          mod ID        mod echo       effect signature        category
offset:   8             26             44                      64
payload length = 0E (56 hex chars)
```

### Short format (received from device)

```
0101 0407 [MOD] [00×7] [SIG_H] [SIG_L] [00×5] [CAT]
          ^^^^         ^^^^^^^^^^^^^^^^        ^^^^
          mod ID       effect signature        category
offset:   8            24                      -
payload length = 0A (40 hex chars)
```

The short format collapses the layout by removing the 16-char block at offsets 26–43 (module echo + padding). The effect-identifying "tail" — the signature bytes plus category — starts at offset **24** in the short format vs offset **40** in the long format.

### Prefix difference

The device also uses prefix `0102` in responses instead of `0101` used in stored commands. Both prefixes are valid and functionally equivalent for payload decoding.

### Comparison example — DRV module, Effect 208

**Stored (long, `0E`):**
```
8080f0 0805 0001 0000 000E 0101 0407 0002 00000000000000 0200000000000000 0400000000000003 F7
                             prefix    mod=2              mod2=2(echo)      sig=0400 cat=03
                                       ↑ offset 8        ↑ offset 26       ↑ offset 40
                                                                            tail = 0400000000000003
```

**Received (short, `0A`):**
```
8080F0 0D06 0001 0000 000A 0102 0407 0002 00000000000000 0400000000000003 F7
                             prefix    mod=2              sig=0400 cat=03
                                       ↑ offset 8        ↑ offset 24
                                                          tail = 0400000000000003
```

The tail `0400000000000003` is identical in both formats.

### Matching solution (fingerprint-based lookup)

Since the tail portion is identical between formats, construct a fingerprint key from:
- **Module ID** (payload chars 8–11) — same in both formats
- **Tail** — `payload.substring(40)` for long format, `payload.substring(24)` for short format

The fingerprint key `moduleId:tail` uniquely identifies each effect regardless of message length. When building the lookup table from stored commands, extract the tail based on payload length (>40 chars → long, ≤40 → short). When matching received messages, extract the tail the same way.

---

## 8. Nibble-Expansion Encoding

All values in this protocol use **nibble expansion**: every logical byte `0xAB` is transmitted as two bytes `0x0A 0x0B`. This means:

- Module ID `9` → transmitted as `00 09` (hex string `0009`)
- Payload length `14` (0x0E) → transmitted as `00 0E` (hex string `000E`) 
- CRC `0x81` → transmitted as `08 01` (hex string `0801`)

To decode: `(high_byte << 4) | low_byte`
To encode: `upper_nibble = byte >> 4`, `lower_nibble = byte & 0x0F`, each prefixed with `0`
