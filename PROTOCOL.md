# ZCAN USB Protocol Documentation

**Device:** NXP USB CANFD DEBUG / Zhiyuan Electronics USBCANFD-200U
**USB ID:** 04d8:0053
**Firmware tested:** fw=0x0200 hw=0x0212

> **Note:** This documentation was entirely reverse-engineered from USB traffic
> captures and static analysis of `libcontrolcanfd.a`. No official protocol
> documentation was available.

---

## USB Endpoints

| Endpoint | Direction | Purpose |
|----------|-----------|---------|
| EP1 OUT (0x01) | Host → Device | Commands (BEEF-framed) + CH0 TX frames |
| EP1 IN  (0x81) | Device → Host | Command responses (BEEF-framed) + 2-byte polling |
| EP2 OUT (0x02) | Host → Device | CH1 TX frames |
| EP2 IN  (0x82) | Device → Host | RX frames received by **CH0** |
| EP3 IN  (0x83) | Device → Host | RX frames received by **CH1** |

> **Important:** EP1 IN continuously sends 2-byte polling packets every ~16ms.
> When reading command responses, these must be discarded until a real
> BEEF-framed response is received.

---

## Command Packet Format

All commands use BEEF/DEAD framing:

```
+--------+--------+--------+--------+--------+--------+--------+--------+
|  0xBE  |  0xEF  | datalen (16-bit BE)      | pkgall | pkgcur |
+--------+--------+--------+--------+--------+--------+--------+--------+
| cmd (16-bit BE) | data[datalen]                                        |
+--------+--------+----...----+--------+--------+--------+--------+
| check (16-bit BE)           |  0xDE  |  0xAD  |
+--------+--------+--------+--------+
```

| Field    | Size       | Description                            |
|----------|------------|----------------------------------------|
| head     | 2 bytes    | Fixed: `0xBEEF`                        |
| datalen  | 2 bytes    | Big-endian, length of data field       |
| pkgall   | 1 byte     | Total packet count (always `0x01`)     |
| pkgcur   | 1 byte     | Current packet number (always `0x01`)  |
| cmd      | 2 bytes    | Command code, big-endian               |
| data     | datalen    | Command-specific payload               |
| check    | 2 bytes    | Checksum                               |
| tail     | 2 bytes    | Fixed: `0xDEAD`                        |

### Checksum Algorithm

Reverse-engineered from `gen_package()` in `gvar.o` (libcontrolcanfd.a):

```c
check = datalen
      + 2 * cmd
      + (pkgall << 8)
      + (pkgcur << 8)
      + sum_of_16bit_bigendian_words(data)
      + 0xBEAD   /* PKG_MAGIC */
```

Result masked to 16 bits.

---

## Initialization Sequence

The following sequence must be followed exactly (verified from USB captures).
Both channels must be fully initialized even when only one is used — otherwise
the device stops delivering RX frames after a TX.

```
1. Wait 1500ms after USB enumeration
2. CMD 0x8001  OPEN_DEVICE    AES-128 challenge-response
3. CMD 0x8005  GET_INFO       Read firmware/hardware version
4. CMD 0x8002  INIT_CAN (ch0) Configure channel 0
5. CMD 0x8002  INIT_CAN (ch1) Configure channel 1
6. CMD 0x800B  SET_BAUD (ch0) Set bitrate
7. CMD 0x8003  START_CAN (ch0) Enable bus
8. CMD 0x800B  SET_BAUD (ch1)
9. CMD 0x8003  START_CAN (ch1)
```

---

## Command Reference

### CMD 0x8001 – OPEN_DEVICE (AES-128 Challenge-Response)

**Direction:** Host → Device (challenge), Device → Host (encrypted response)

This is an authentication handshake. The host sends a 32-byte challenge
plaintext and the device responds with AES-128 ECB encrypted output.

**Host sends 32-byte challenge:**
```
[0..1]  = 0xDE 0xFF  (magic)
[2..5]  = 0x00 0x00 0x00 0x00  (MUST be zero - device rejects non-zero)
[6..7]  = 0x43 0x01  (fixed)
[8..11] = 0xF2 0x89 0x82 0xEE  (fixed)
[12..15]= timestamp / random value
[16..21]= 0xD0 0x1F 0x07 0x11 0x5F 0x68  (fixed)
[22..25]= random bytes
[26..31]= 0xC2 0x3E 0xC8 0x26 0x52 0x36  (fixed tail)
```

**Device responds with 32-byte encrypted output:**
```
response = AES_128_ECB_encrypt(key, challenge[0:16])
         + AES_128_ECB_encrypt(key, challenge[16:32])
```

**AES-128 Key** (extracted from `controlcanfd.o` via GDB):
```
61 62 0B 1A 65 74 63 70 3B 40 75 00 38 22 71 65
```

---

### CMD 0x8005 – GET_INFO

**Direction:** Host → Device (empty payload), Device → Host (info)

**Response payload:**
```
[0..1]  = hw_version (BE)
[2..3]  = fw_version (BE)
[4..5]  = dr_version (BE)
[6..7]  = in_version (BE)
[8..9]  = irq_num
[10..11]= can_num (number of channels)
[12..23]= serial number (ASCII)
[24..39]= hardware type string (ASCII)
```

---

### CMD 0x8002 – INIT_CAN

**Direction:** Host → Device (32 bytes), Device → Host (ACK)

**Payload (verified from USB captures):**
```
[0]     = 0x55  magic
[1]     = 0x02  fixed
[2]     = channel index (0 or 1)
[3]     = 0x01  type (must be 0x01, even for classic CAN)
[4..5]  = 0x00 0x00
[6..9]  = acc_code = 0x00000001
[10..13]= acc_mask = 0xFFFFFFFF
[14..17]= abit_timing = 0x00000000
[18..21]= dbit_timing (bitrate-dependent, see table below)
[22..25]= 0x03 0x01 0x0A 0x02  (fixed, from capture)
[26..29]= 0x00000000
[30]    = 0x00  (termination disabled — hardware has fixed 120Ω)
[31]    = 0x00
```

**dbit_timing values:**

| Bitrate     | dbit_timing  |
|-------------|--------------|
| 125 kbit/s  | `0x0114BE2F` |
| 250 kbit/s  | `0x0014BE2F` |
| 500 kbit/s  | `0x00025E17` |
| 1 Mbit/s    | `0x00022E0B` |

---

### CMD 0x800B – SET_BAUD

**Direction:** Host → Device, Device → Host (ACK)

**4-byte variant (set bitrate):**
```
[0]    = 0x7E  magic
[1]    = channel index
[2..3] = baud_code (BE)
```

**Baud codes:**

| Bitrate     | baud_code |
|-------------|-----------|
| 125 kbit/s  | `0x3FFF`  |
| 250 kbit/s  | `0x1FFF`  |
| 500 kbit/s  | `0x0FFF`  |
| 1 Mbit/s    | `0x007F`  |

**12-byte variant (set hardware ID filter):**
```
[0]     = 0x7E  magic
[1]     = channel index
[2]     = filter mode (0xA0 = ID range, 0xB0 = single ID)
[3]     = 0x01  fixed
[4..7]  = start_id (BE uint32)
[8..11] = end_id   (BE uint32)
```

---

### CMD 0x8003 – START_CAN

**Direction:** Host → Device (4 bytes), Device → Host (ACK)

```
[0] = 0x55  magic
[1] = 0x80
[2] = 0x03
[3] = channel index
```

---

### CMD 0x8008 – RESET_CAN

**Direction:** Host → Device (4 bytes)

```
[0] = 0x55
[1] = 0x80
[2] = 0x03
[3] = channel index
```

---

### CMD 0x8004 – TRANSMIT (TX Frame)

**Direction:** Host → Device via EP1 OUT (CH0) or EP2 OUT (CH1)

TX frames are BEEF-wrapped commands with `cmd = 0x8004`.

**Classic CAN payload (26 bytes, verified from captures):**
```
[0]     = 0x55  magic
[1]     = 0xF1  classic CAN marker (0xF2 = CAN FD)
[2..7]  = 0x00  reserved
[8..9]  = CAN ID, big-endian 16-bit (11-bit SFF)
[10]    = DLC (0..8)
[11..11+dlc-1] = CAN frame data
[12+dlc..21]   = 0x00 padding
[21]    = transmit_type:
          0x00 = normal (auto-retry on arbitration loss or error)
          0x01 = single send (no retry)
[22..25]= 0x00 padding
```

**Example** (ID=0x111, DLC=4, data=AA BB CC 00, normal send):
```
55 F1 00 00 00 00 00 00  01 11 04  AA BB CC 00  00 00 00 00 00 00 00  00 00 00 00
magic type  [reserved ]  ID   DLC  [data      ]  [padding             ] txtype [pad]
```

---

## RX Frame Format (EP2 IN / EP3 IN)

RX frames arrive as **raw 21-byte packets** without BEEF framing.

**Endpoint routing:**
- **EP2 IN (0x82):** Frames received by CH0
- **EP3 IN (0x83):** Frames received by CH1

> **Warning:** Both channels must be started (INIT + BAUD + START) for both
> endpoints to deliver data. If CH1 is not initialized, EP3 stays silent and
> the device stops sending frames on EP2 after any TX is performed.

**Frame structure (verified from live USB captures):**
```
[0..1]  = flags / partial timestamp (little-endian)
[2..3]  = CAN_ID << 2, little-endian
[4..5]  = 0x00 0x00
[6]     = lower nibble = DLC  (upper nibble = frame flags)
[7]     = 0xFF  (fixed marker)
[8..8+dlc-1] = CAN frame data
[8+dlc..20]  = 0x00 padding
```

**Decoding:**
```c
u32 can_id = (((u32)frame[3] << 8) | frame[2]) >> 2;
u8  dlc    = frame[6] & 0x0f;   /* lower nibble */
u8 *data   = frame + 8;
```

**Example** (ID=0x111, DLC=4, data=AA BB CC 00):
```
xx xx  44 04  00 00  04 FF  AA BB CC 00  00 00 00 00 00 00 00 00 00 00 00
       ↑↑↑↑               ↑↑
       ID=0x111            DLC=4 (lower nibble of 0x04)
```
CAN ID calculation: `(0x0444) >> 2 = 0x111` ✓

---

## Notes

- The 1500ms startup delay is required — the device firmware needs time to
  initialize after USB enumeration before it will accept the AES handshake.
- The AES key was extracted from `controlcanfd.o` using GDB on a
  statically linked test binary.
- The `PKG_MAGIC = 0xBEAD` constant was found in the disassembly of
  `gen_package()` in `gvar.o`.
- `transmit_type = 0x00` (auto-retry) is required for reliable transmission;
  `0x01` (single send) causes ~50% packet loss in practice.
- The `type = 0x01` field in INIT_CAN must always be set to `0x01` even when
  using classic CAN — the device ignores the distinction at the frame level.
