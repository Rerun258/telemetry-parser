# Telemetry Frame Specification (v1)

## Conventions
- All multi-byte integers are **big-endian** (most significant byte first).
- Floats are IEEE 754 big-endian (`float32`).
- Offsets are in bytes from the start of the frame.

## Frame layout

| Offset | Field     | Size | Type   | Description |
|--------|-----------|------|--------|-------------|
| 0      | SYNC      | 2    | u16    | Constant `0xEB90`. Marks a candidate frame start. |
| 2      | VERSION   | 1    | u8     | Format version. Currently `1`. |
| 3      | SEQ       | 2    | u16    | Increments by 1 per frame, wraps from 65535 to 0. |
| 5      | TIMESTAMP | 8    | u64    | Microseconds since the Unix epoch. |
| 13     | LENGTH    | 2    | u16    | Number of payload bytes. Maximum: 256. |
| 15     | PAYLOAD   | LENGTH | bytes | See payload layout below. |
| 15+LENGTH | CRC16  | 2    | u16    | CRC-16/CCITT-FALSE over VERSION through PAYLOAD. |

Total frame size = 17 + LENGTH bytes.

## CRC
- Algorithm: CRC-16/CCITT-FALSE (poly `0x1021`, init `0xFFFF`, no reflection, no final XOR).
- Covers: every byte from VERSION through the last PAYLOAD byte. It does **not** cover SYNC or the CRC itself.
- Check value: the ASCII bytes `123456789` produce `0x29B1`.

## Payload (version 1, LENGTH = 9)

| Offset | Field       | Size | Type    | Unit |
|--------|-------------|------|---------|------|
| 0      | SENSOR_ID   | 1    | u8      | n/a  |
| 1      | TEMPERATURE | 4    | float32 | °C   |
| 5      | PRESSURE    | 4    | float32 | kPa  |

## Validity rules
A frame is accepted only if all of these hold:
1. SYNC matches `0xEB90`.
2. LENGTH is at most 256.
3. VERSION is 1.
4. The CRC computed by the receiver equals the CRC in the frame.

Sequence gaps and repeats do not cause rejection. They are counted and reported.

## Resynchronization
When a candidate frame fails any check, discard only its first byte and resume
searching for SYNC from the next byte. A real frame may begin inside the bytes
that were just examined.

## Anomaly thresholds (applied only to accepted frames)
TODO (step 8): temperature and pressure limits.