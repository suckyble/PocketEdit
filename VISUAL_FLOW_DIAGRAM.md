# Visual Flow: USB Multi-Packet SysEx Handling

## Message Flow Comparison

### BLE (Single Message - Original Working Approach)
```
Device sends:  [Large SysEx Message with all data]
                        ↓
           handleMidiMessage()
                        ↓
        Pattern matching on full data
                        ↓
          Decode names/parameters ✅
```

### USB (Fragmented - Now Fixed)
```
Device sends:  [SysEx Msg 1/5] → [SysEx Msg 2/5] → [SysEx Msg 3/5] → [SysEx Msg 4/5] → [SysEx Msg 5/5]
                        ↓            ↓              ↓              ↓              ↓
              handleMidiMessage() (5x called)
                        ↓
              Detect multi-packet (bytes [3:4])
                        ↓
         Buffer by responseId with packets[] map
                        ↓
       Check: all packets (0..4) received?
                    ↙             ↖
                 NO              YES
                 ↓                ↓
          Set 2s timeout    Extract payloads
                 ↓            [bytes 9..end-1]
            Timeout? →          from all packets
            Log warn           Concatenate
            Clean up              ↓
                          Create synthetic
                        single-packet message
                             [0xF0] [ID] [00 01] [00 00] [00 00] [PAYLOAD...] [0xF7]
                                           ↑ Single-packet indicator
                                    ↓
                          Process with 8080 header
                                    ↓
                         handleNotification()
                                    ↓
                        Pattern matching on payload
                                    ↓
                    Decode names/parameters ✅
```

## State Machine: Multi-Packet Response Buffering

```
                    ┌─────────────────────────────┐
                    │   Receive Complete SysEx    │
                    │  (F0...F7, not partial)     │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    │                             │
            ┌───────▼───────┐          ┌─────────▼─────────┐
            │ Single Packet │          │ Multi-Packet      │
            │  (bytes[3:4]  │          │ (bytes[3:4] > 1)  │
            │    == 0001)   │          │                   │
            └───────┬───────┘          └────────┬──────────┘
                    │                           │
            ┌───────▼─────────┐         ┌───────▼────────────────────┐
            │ Process via     │         │ Create/Get Response Entry   │
            │ handleNotification          responseid =                │
            │ IMMEDIATE ✓     │         │ ${msg[1]}_${msg[2]}_...    │
            └─────────────────┘         └───────┬────────────────────┘
                                                │
                                        ┌───────▼──────────┐
                                        │ Buffer packet at │
                                        │ packets[N]       │
                                        │ where N =        │
                                        │ currentPacket    │
                                        └───────┬──────────┘
                                                │
                                    ┌───────────▼──────────────┐
                                    │ Check completion:        │
                                    │ ∀i ∈ [0..total):         │
                                    │   i in packets?          │
                                    └───────────┬──────────────┘
                                                │
                        ┌───────────────────────┴──────────────────────┐
                        │                                              │
                  ┌─────▼────────┐                          ┌─────────▼──────┐
                  │ Not Complete  │                         │   Complete!    │
                  │ Set 2s Timer  │                         │  All packets   │
                  │ (then cleanup)│                         │   received     │
                  └───────────────┘                         └────────┬───────┘
                                                                     │
                                            ┌────────────────────────▼────────────────┐
                                            │ Reassemble:                            │
                                            │ For each packet i=[0..total):          │
                                            │   Extract bytes [9] to [len-1]        │
                                            │   Append to reassembledPayload        │
                                            └────────────────────┬───────────────────┘
                                                                 │
                                            ┌────────────────────▼────────────────┐
                                            │ Build single-packet message:       │
                                            │ [F0] [ID...] [0001] [0000] [0000] │
                                            │ [REASSEMBLED_PAYLOAD] [F7]         │
                                            └────────────────────┬───────────────┘
                                                                 │
                                            ┌────────────────────▼────────────────┐
                                            │ Process with 8080 header           │
                                            │ [8080] + [synthetic message]       │
                                            │ → handleNotification()             │
                                            └────────────────────┬───────────────┘
                                                                 │
                                                    ┌────────────▼──────────┐
                                                    │ Pattern matching      │
                                                    │ on reassembled        │
                                                    │ payload ✓             │
                                                    └───────────────────────┘
                                                    
                                            ┌────────────────────┐
                                            │ Clean up response   │
                                            │ delete responses[]  │
                                            └────────────────────┘
```

## Byte Layout: Multi-Packet SysEx Message

```
USB Message N (packet number N):
┌─────┬────┬────┬────┬────┬────┬────┬────┬────┬─────────────────────────┬─────┐
│ F0  │CRC1│CRC2│TP1│TP2│PN1│PN2│RSV1│RSV2│   PAYLOAD (variable)     │ F7  │
├─────┼────┼────┼────┼────┼────┼────┼────┼────┼─────────────────────────┼─────┤
│ [0] │[1] │[2] │[3] │[4] │[5] │[6] │[7] │[8] │[9] ... [len-2]         │[len-1]
└─────┴────┴────┴────┴────┴────┴────┴────┴────┴─────────────────────────┴─────┘

Key fields:
[0]:     SysEx start marker (0xF0)
[1:2]:   CRC (ignored)
[3:4]:   Total Packets (16-bit big-endian) ← KEY for multi-packet detection
[5:6]:   Current Packet (16-bit big-endian) ← 0-indexed packet number
[7:8]:   Reserved (not used)
[9..]:   Actual payload data
[end]:   SysEx end marker (0xF7)

Example - Message 1 of 5:
┌────┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬── ────┬──┐
│F0  │07│00│00│05│00│00│01│03│01│02│02│... ...│F7│
│    │  │  │  │  │     │     │
│    │CRC  │      │Total=5  │
│         │Current=0
│    Synthetic MFG ID
└─ Start of SysEx

This indicates:
- Manufacturer ID: (0x07, 0x00) [synthetic/non-standard]
- Total packets in response: 5
- This is packet 0 (first of 5)
- Payload: starting at byte [9]
```

## Reassembly Process (Detailed)

```
Input: 5 complete SysEx messages with packets F0...F7

Step 1: Buffer individual packets
┌─────────────────┬─────────────────┬─────────────────┬─────────────────┬─────────────────┐
│ Message 1       │ Message 2       │ Message 3       │ Message 4       │ Message 5       │
│ [00 00]         │ [00 01]         │ [00 02]         │ [00 03]         │ [00 04]         │
│ Extract [9..]:  │ Extract [9..]:  │ Extract [9..]:  │ Extract [9..]:  │ Extract [9..]:  │
│ Payload 1       │ Payload 2       │ Payload 3       │ Payload 4       │ Payload 5       │
└────────┬────────┴────────┬────────┴────────┬────────┴────────┬────────┴────────┬────────┘
         │                │                │                │                │
         └────────────────┴────────────────┴────────────────┴────────────────┘
                                    │
                           Step 2: Concatenate
                                    ↓
                    ┌───────────────────────────┐
                    │  REASSEMBLED_PAYLOAD      │
                    │  = Payload1 +             │
                    │    Payload2 +             │
                    │    Payload3 +             │
                    │    Payload4 +             │
                    │    Payload5               │
                    │  (all bytes concatenated) │
                    └───────────────┬───────────┘
                                    │
                           Step 3: Wrap in SysEx
                                    ↓
        ┌─────────────────────────────────────────────┐
        │ [F0] [MFG] [0001] [0000] [0000]             │
        │ [REASSEMBLED_PAYLOAD...] [F7]               │
        │         ↑ Single-packet wrapper
        └─────────────────────────────────────────────┘
                                    │
                           Step 4: Add BLE header
                                    ↓
        ┌──────────────────────────────────────┐
        │ [80] [80]                             │
        │ [F0] [MFG] [0001] [0000] [0000]       │
        │ [REASSEMBLED_PAYLOAD...] [F7]         │
        │  ↑ BLE compatibility header
        └──────────────────────────────────────┘
                                    │
                           Step 5: Process
                                    ↓
              handleNotification(synthetic_message)
                                    ↓
                  Pattern matching works on reassembled
                  payload as if it were BLE ✓
```

## Response ID Generation

```
responseId = "${message[1]}_${message[2]}_${message[3]}_${message[4]}"

Example:
If message[1] = 0x07
   message[2] = 0x00
   message[3] = 0x00
   message[4] = 0x05

responseId = "7_0_0_5"

This ensures:
- Different message types have different IDs
- Same type with same packet count uses same ID
- Multiple concurrent responses don't collide
```

## Timeline: Complete Example

```
T=0ms:    Device sends SysEx message 1/5
          handleMidiMessage() → Detect multi-packet
          → Create buffer entry with responseId
          → packets[0] = message1
          → Wait for packets 1-4

T=50ms:   Device sends SysEx message 2/5
          → Append to packets[1]
          → Check: have packets [0..4]? NO
          → Continue waiting

T=100ms:  Device sends SysEx message 3/5
          → Append to packets[2]
          → Check: have packets [0..4]? NO
          → Continue waiting

T=150ms:  Device sends SysEx message 4/5
          → Append to packets[3]
          → Check: have packets [0..4]? NO
          → Continue waiting

T=200ms:  Device sends SysEx message 5/5
          → Append to packets[4]
          → Check: have packets [0..4]? YES ✓
          → Reassemble payload
          → Create synthetic single-packet
          → Call handleNotification()
          → Pattern matching succeeds
          → Decode data ✓
          → Clean up: delete responseId entry

T=205ms:  Response processing complete, ready for next request
```

## Timeout Scenario

```
T=0ms:    Receive packet 0/5 → Buffer, set 2s timer
T=50ms:   Receive packet 1/5 → Update
T=100ms:  Receive packet 2/5 → Update
T=150ms:   NO packet 3/5 arrives...
T=1500ms:  Still waiting...
T=2050ms:  TIMEOUT TRIGGERED
           → Log warning: "Timeout waiting for multi-packet response..."
           → Log detail: "received 3/5"
           → Delete partial response
           → Ready for next message
           → NO DEADLOCK ✓
```

---

**Key Insight**: The synthetic single-packet message makes the reassembled USB response look exactly like a BLE response, allowing all downstream processing to work unchanged.
