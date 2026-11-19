# Glossary

Plain-language definitions of the terms used in this project, for readers who
don't work with embedded or aerospace software.

## The big picture

**Telemetry**: Measurements that a remote machine (a rocket, satellite, drone,
or factory sensor) sends automatically to the people or computers monitoring it.
Think of a car dashboard whose readings are transmitted over a radio or network
link.

**Downlink**: The radio link carrying data from a vehicle down to the ground.

**Ground station**: The receiving side of that link. In this project, the parser
plays this role.

**Sensor**: A device that measures something physical, such as temperature or
pressure.

## Data format

**Byte**: A unit of data holding a value from 0 to 255. Files and network
traffic are sequences of bytes.

**Binary data**: Data stored as raw bytes instead of readable text. It is compact
and fast, but looks like gibberish in a text editor.

**Hexadecimal (hex)**: A base-16 number system using the digits 0-9 and A-F.
Programmers use it because two hex digits equal exactly one byte. The prefix
`0x` marks a hex number, so `0xEB90` is the two bytes `EB` and `90`.

**Frame (packet)**: One self-contained message with a fixed structure.

**Header and payload**: The header is the bookkeeping at the start of a frame
(what it is, when it was sent, how long it is). The payload is the actual data,
such as the sensor readings. Think of an envelope and the letter inside it.

**Sync word**: A fixed pattern of bytes at the start of every frame that means
"a message begins here." The parser searches for it, especially after something
goes wrong.

**Sequence number**: A counter that increases by one with each frame, so the
receiver can notice missing or duplicated frames. It wraps back to 0 after
65,535.

**Timestamp / Unix epoch**: When a reading was taken, stored as the number of
microseconds since January 1, 1970 (UTC). A microsecond is one millionth of a
second.

**Endianness (big-endian / little-endian)**: The order in which the bytes of a
multi-byte number are stored or sent. Big-endian sends the most significant
byte first, so the number `0x1234` is sent as `12 34`. Little-endian sends
`34 12`. Both sides must agree, or numbers are read wrongly. (The names come from
a satirical dispute in *Gulliver's Travels* over which end of an egg to crack.)
*In this project:* big-endian.

**Floating-point number (float32, IEEE 754)**: The standard way computers store
decimal numbers such as 71.2. A `float32` uses 4 bytes.

**Serialization**: Converting values in memory into an exact sequence of bytes
for sending, and converting them back (deserialization). This project does it
field by field so the result never depends on the compiler.

## Detecting errors

**Bit flip**: A single 0 becoming a 1, or the reverse, usually because of radio
noise or interference during transmission.

**Checksum / CRC-16**: A small number calculated from a message's contents and
attached to it. The receiver recalculates it, and a mismatch means the message
was damaged. CRC stands for cyclic redundancy check, and 16 is the size of the
result in bits. It catches accidental corruption, not deliberate tampering.
*In this project:* CRC-16/CCITT-FALSE, a specific standard variant. The text
`123456789` must produce `0x29B1`, which confirms the implementation is correct.

**Malformed frame**: A frame that breaks the format rules, such as a bad
checksum, an impossible length, or an unknown version.

## How the parser behaves

**Parser**: A program that reads raw data and turns it into structured,
meaningful records.

**State machine**: A design in which the program is always in one named state
(for example "looking for sync word" or "reading payload") with clear rules for
moving between states. It makes behavior predictable and easy to test.

**Streaming**: Data arrives in chunks of arbitrary size, so one frame may be
split across two chunks. The parser remembers where it left off.

**Resynchronization (resync)**: Recovering after corrupted data by scanning for
the start of the next valid frame.

**Anomaly detection / threshold**: Flagging readings outside a safe range, such
as a temperature above a set limit. It is applied only to frames whose checksum
passed.

**Deterministic**: The same input always produces exactly the same output.

**Logging (JSON Lines)**: Recording every result in a file, one structured
record per line, so it can be searched and analyzed later.

## Testing and quality

**Unit test**: A small automated check of one piece of code.

**GoogleTest**: A widely used open-source library for writing C++ tests.

**Known-answer test**: A test against an independently published correct
answer, such as the CRC value above.

**Round-trip test**: Converting data one way and back, and checking that the
original comes out unchanged.

**Fault injection**: Deliberately damaging data (flipped bits, cut-off frames,
dropped bytes) to prove the software handles it safely.

**Fuzzing**: Feeding a program huge amounts of random or mutated input to find
crashes and hidden bugs.

**Static library**: A bundle of compiled code that other programs reuse. Here
the parsing logic lives in one, so the app and the tests run identical code.

**Sanitizer (AddressSanitizer)**: A compiler tool that watches a running program
for memory errors, such as reading past the end of an array.

**Static analysis**: Tools that scan source code for likely bugs without running
it.

**Test coverage**: The fraction of code lines that the tests actually execute.

**Continuous integration (CI)**: Automatically building and testing the project
every time code is pushed to the repository.

**End-to-end test**: A test that runs the whole pipeline, from data generation
to the final log, and checks the result.