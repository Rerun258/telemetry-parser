# Function Tree

Design rule: bad input is expected, not exceptional. The parse path returns
status codes and never throws.

## parser/ (static library, no file or console I/O)

### constants.h
- `SYNC_WORD = 0xEB90`: fixed 2-byte marker at the start of every frame.
- `PROTOCOL_VERSION = 1`: the only frame format version this parser accepts.
- `MAX_PAYLOAD = 256`: largest allowed payload; a larger LENGTH is treated as corruption.
- `HEADER_SIZE = 16`: bytes from SYNC through LENGTH.
- `CRC_SIZE = 2`: bytes in the trailing checksum.
- `MAX_FRAME_SIZE = 274`: header + max payload + CRC; the size of the parser's fixed buffer.

### status.h
- `enum class Status`: result codes returned instead of exceptions.
  - `Ok`: frame decoded successfully.
  - `BadSync`: the first two bytes were not `0xEB90`.
  - `BadVersion`: VERSION is not the supported value.
  - `BadLength`: LENGTH exceeds `MAX_PAYLOAD`, or does not match the bytes supplied.
  - `BadCrc`: the CRC computed by the receiver differs from the one in the frame.
  - `Truncated`: fewer bytes were supplied than the frame needs.

### byte_io.h / byte_io.cpp (step 2): big-endian helpers
- `writeU8 / writeU16 / writeU32 / writeU64(std::vector<uint8_t>&, value)`: append an integer to the end of a buffer, most significant byte first.
- `writeFloat32(std::vector<uint8_t>&, float)`: convert a float to its 32-bit pattern with `std::bit_cast`, then write it with `writeU32`.
- `readU8 / readU16 / readU32 / readU64(const uint8_t*) -> value`: decode an integer from big-endian bytes; the caller guarantees enough bytes exist.
- `readFloat32(const uint8_t*) -> float`: read 4 bytes with `readU32`, then `std::bit_cast` back to a float.

### crc16.h / crc16.cpp (step 3)
- `uint16_t crc16(const uint8_t* data, size_t len)`: compute CRC-16/CCITT-FALSE (poly `0x1021`, init `0xFFFF`) over a byte range.

### frame.h / frame.cpp (step 2; CRC wired in at step 3)
- `struct Frame { version, sourceId, seq, timestampUs, payload }`: one decoded frame, with the payload left as raw bytes.
- `struct SensorReading { sensorId, temperatureC, pressureKpa }`: the decoded meaning of a version-1 payload.
- `serializeFrame(const Frame&, std::vector<uint8_t>& out) -> Status`: turn a Frame into wire bytes (sync, header, payload, CRC). Fails if the payload exceeds `MAX_PAYLOAD`.
  - calls: byte_io writers, `crc16`
- `deserializeFrame(const uint8_t*, size_t, Frame& out) -> Status`: validate and decode one complete candidate frame, checking sync, version, length, and CRC in that order.
  - calls: byte_io readers, `crc16`
- `encodeReading(const SensorReading&) -> std::vector<uint8_t>`: pack a reading into its 9-byte payload (sensor ID, temperature, pressure).
- `decodeReading(const uint8_t*, size_t, SensorReading& out) -> Status`: unpack a payload into a reading; fails if the length is not 9.

### sequence_tracker.h / .cpp (step 7)
- `SequenceTracker::observe(sourceId, seq) -> SeqResult`: compare this frame's sequence number with the last one seen from the same source.
  - `InOrder`: exactly one higher than the last (65535 wraps to 0).
  - `Gap(n)`: n frames were skipped, meaning frames were lost.
  - `Repeat`: same or older number, meaning a duplicate or out-of-order frame.

### parser.h / parser.cpp (step 4; resync at step 7)
- `class FrameParser`
  - `feed(const uint8_t* data, size_t len)`: public entry point; push any number of bytes in, split across frames however they arrive.
    - calls: `step(byte)` for each byte
  - `reset()`: return to the starting state and clear the buffer, keeping the statistics.
  - `stats() -> ParserStats`: return the running counters.
  - `setEventHandler(std::function<void(const ParserEvent&)>)`: register the callback that receives every result.
  - private, one handler per state:
    - `step(byte)`: send one byte to the handler for the current state.
    - `huntSync(byte)`: scan for `0xEB` followed by `0x90`; on a match, begin collecting a candidate frame.
    - `readHeader(byte)`: collect the 14 header bytes after SYNC, then check VERSION and LENGTH before waiting for the payload.
    - `readPayload(byte)`: collect exactly LENGTH payload bytes.
    - `readCrc(byte)`: collect the 2 CRC bytes, then call `validateFrame`.
    - `validateFrame()`: run `deserializeFrame` on the collected bytes, update sequence tracking, emit an event, and return to hunting.
    - `resync()`: after a failed candidate, drop only its first byte and replay the remaining buffered bytes through `huntSync`, since a real frame may start inside them.
- `struct ParserEvent`: what the parser reports.
  - `FrameAccepted`: a frame passed every check, and carries the decoded data.
  - `FrameDropped(reason)`: a candidate failed, with the `Status` that explains why.
  - `SeqGap / SeqRepeat`: frames were lost or duplicated.
  - `SensorFault`: the frame is valid, but a reading is NaN or infinite.
- `struct ParserStats`: counters for bytes fed, bytes skipped, frames accepted, frames dropped (per reason), sequence gaps, and repeats.

### validator.h / validator.cpp (step 8)
- `struct Thresholds { tempMaxC, tempMinC, pressureMinKpa, pressureMaxKpa }`: the safe operating limits (values to be decided).
- `checkReading(const SensorReading&, const Thresholds&) -> ReadingStatus`: classify a reading from an accepted frame.
  - `SensorFault`: a value is NaN or infinite (`!std::isfinite`).
  - `Ok / TempHigh / TempLow / PressureLow / PressureHigh`: finite values compared against the limits.

## telemetry-parser/ (app, thin shell) (steps 4, 9, 10)
- `main(argc, argv)`: read the input