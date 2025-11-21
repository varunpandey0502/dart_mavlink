# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

dart_mavlink is a Dart library for parsing and serializing MAVLink v1/v2 packets. MAVLink is a lightweight messaging protocol used for communicating with drones and other unmanned vehicles.

## Key Concepts

### Architecture

The library has three main layers:

1. **Protocol Layer** (`lib/mavlink_parser.dart`, `lib/mavlink_frame.dart`):
   - `MavlinkParser`: State machine that parses incoming byte streams into `MavlinkFrame` objects
   - `MavlinkFrame`: Container for MAVLink packets (supports both v1 and v2 protocol versions)
   - Handles CRC validation, byte ordering, and protocol version differences

2. **Dialect Layer** (`lib/dialects/*.dart`, `lib/mavlink_dialect.dart`):
   - Each dialect XML file (from MAVLink message definitions) generates a Dart file
   - Dialects define available messages and enums for specific vehicle types
   - Common dialects: `common`, `ardupilotmega`, `minimal`
   - Each dialect implements the `MavlinkDialect` interface for parsing message IDs

3. **Message Layer** (`lib/mavlink_message.dart`, generated classes):
   - Each message type is an immutable Dart class
   - Messages implement `MavlinkMessage` interface with `serialize()` method
   - Field ordering follows MAVLink serialization rules (sorted by size, extensions last)

### Code Generation

The core of this library is **code generation** from MAVLink XML definitions:

- **Generator**: `tool/generate.dart` parses MAVLink XML files and generates Dart dialect files
- **Input**: MAVLink XML definitions from `mavlink/message_definitions/v1.0/`
- **Output**: Dart files in `lib/dialects/` containing message classes, enums, and dialect implementations
- **Key classes in generator**:
  - `DialectDocument`: Parses XML and handles include directives
  - `DialectMessage`: Represents a message definition with CRC calculation
  - `DialectField`: Represents message fields with type parsing
  - `ParsedMavlinkType`: Handles conversion from MAVLink types to Dart types

### Special Handling

- **Entry value parsing** (`tool/generate.dart:129-141`): Handles multiple value formats in XML:
  - Raw integers (e.g., `8`)
  - Exponentiation (e.g., `2**4`)
  - Hex values (e.g., `0xFE`)
  - Binary values (e.g., `0b001000`)

- **Field reordering** (`DialectMessage.orderedFields`): MAVLink requires fields ordered by size (largest first) for wire format, but extensions always come last

- **CRC Extra**: Each message has a CRC extra byte calculated from message definition to detect version mismatches

## Development Commands

### Running Tests
```bash
dart test                                    # Run all tests
dart test test/mavlink_parser_v1_test.dart  # Run specific test file
```

### Code Generation
```bash
dart run tool/generate.dart                  # Regenerate all dialect files from XML
```

The generator:
- Reads XML files from `mavlink/message_definitions/v1.0/`
- Generates Dart files in `lib/dialects/`
- Automatically formats generated code with `dart format`

### Formatting
```bash
dart format .                                # Format all Dart files
```

### Running Examples
```bash
dart run example/sitl_test.dart              # Run SITL test (requires Ardupilot SITL)
dart run example/parser.dart                 # Run parser example
```

## Message Immutability

All message classes are immutable. To modify a message, use the `copyWith()` method that is generated for each message class:

```dart
CommandLong modified = original.copyWith(param2: 10);
```

## Message Signing (MAVLink 2 Security)

The library implements MAVLink 2 message signing for authentication and replay attack prevention.

### Architecture

**Signature Manager** (`lib/mavlink_signature.dart`):
- `MavlinkSignatureConfig`: Holds 32-byte secret key, link ID, and accept policy
- `MavlinkSignatureManager`: Core signing functionality
  - Generates monotonically increasing 48-bit timestamps (10µs units since Jan 1, 2015)
  - Calculates SHA-256 signatures (first 48 bits used)
  - Verifies signatures and timestamp validity
  - Tracks last timestamp per link (systemId:componentId:linkId) for replay prevention
  - Enforces accept policy (signedOnly, acceptUnsigned, acceptAll)

**Signing Process**:
1. `MavlinkFrame.v2()` accepts optional `signatureManager` parameter
2. When serializing, if manager present:
   - Sets incompatibility flag 0x01 (signed bit)
   - Generates timestamp via manager
   - Calculates signature: SHA-256(secretKey + header + payload + CRC + linkId + timestamp)
   - Appends 13 bytes: linkId(1) + timestamp(6) + signature(6)

**Verification Process**:
1. `MavlinkParser` accepts optional `signatureManager` parameter
2. Parser state machine includes signature states after CRC
3. When incompatibility flag 0x01 detected, reads 13 signature bytes
4. Verifies signature matches expected SHA-256 hash
5. Validates timestamp (not old, not future, monotonically increasing)
6. Applies accept policy to determine if packet is emitted to stream

### Usage

To enable signing, pass `MavlinkSignatureManager` to both frame creation and parser:
```dart
// Send
MavlinkFrame.v2(seq, sysId, compId, message, signatureManager: manager)

// Receive
MavlinkParser(dialect, signatureManager: manager)
```

Without a signature manager, frames are unsigned and parser accepts all packets.

## Testing Strategy

- `test/generate_test.dart`: Tests XML parsing and code generation logic
- `test/mavlink_parser_v1_test.dart`: Tests MAVLink v1 protocol parsing
- `test/mavlink_parser_v2_test.dart`: Tests MAVLink v2 protocol parsing
- `test/mavlink_serialize_v1_test.dart`: Tests message serialization
- `test/mavlink_signature_test.dart`: Tests signature generation, verification, and policies
- `test/mavlink_signing_integration_test.dart`: Tests end-to-end signing with real messages

Test files in `test/mavlink_dialect/` contain minimal XML definitions for testing the generator.
