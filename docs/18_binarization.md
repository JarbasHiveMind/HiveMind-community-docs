# Binarization Protocol

> **Experimental**: subject to change without warning.

The HiveMind Binarization Protocol serializes and deserializes structured messages into compact binary formats for network transmission. This page describes the protocol's structure, encoding rules, and the reasoning behind key design decisions. The binary format carries a protocol version, to support backward compatibility and future extensions.

## Protocol versions

The protocol uses an integer version number to signal supported features and ensure compatibility between clients and servers. The current protocol version is `1`. Any change in functionality or structure requires a new version number.

Version-specific functionality:

- **Version 0**: the original protocol design. No binarization, no handshake, only a pre-shared `crypto_key`.
- **Version 1**: adds support for handshakes and binary payloads.

### Message types

The HiveMind protocol groups messages into types, each with a specific role. These types are encoded as 5-bit unsigned integers, which allows up to 32 distinct types. Examples include:

| Value | Type            | Description                      |
|-------|-----------------|----------------------------------|
| 0     | HANDSHAKE       | Initial connection handshake.   |
| 1     | BUS             | Standard message bus.           |
| 2     | SHARED_BUS      | Shared bus for multiple nodes.  |
| 3     | BROADCAST       | Global message broadcast.       |
| 4     | PROPAGATE       | Directed message propagation.   |
| 12    | BINARY          | Raw binary payload.             |

### Compression

Payloads can optionally use zlib compression. A single bit in the header shows whether compression is applied. Compressed payloads reduce transmission size but add a small computational cost during encoding and decoding.

### Metadata (HiveMeta)

HiveMeta is a reserved field for attaching arbitrary metadata to a message. The metadata is encoded as a byte array, prefixed by its size in bytes. This allows extensible features such as routing hints or debug information.

## Binary message structure

The serialized binary message consists of a header and a payload. All fields are packed to save space. The structure is as follows.

### Header

The header holds the protocol version, message type, compression flag, and metadata length.

| Field           | Size (bits) | Description                                    |
|------------------|-------------|------------------------------------------------|
| Start Marker     | 1           | Always `1`. Helps align message boundaries.   |
| Versioned Flag   | 1           | Indicates if protocol version is specified.   |
| Protocol Version | 8           | Protocol version (if `Versioned Flag` is `1`).|
| Message Type     | 5           | Encoded message type.                         |
| Compressed Flag  | 1           | Indicates if payload is compressed.           |
| Metadata Length  | 8           | Length of metadata in bytes.                  |

### Metadata

Metadata is optional and encodes key-value pairs or other information. If present, it follows the header. Its length is set in the header.

### Payload

The payload holds the core message data. Its format depends on the message type:

- **Standard messages**: encoded as a UTF-8 JSON string.
- **Binary messages**: encoded as raw bytes, with an additional 4-bit unsigned integer that indicates the binary payload type.

### Padding

To keep byte alignment, the protocol adds padding bits (`0`) as needed. The total message length must be a multiple of 8 bits.

## Encoding process

1. Add a single bit set to `1` for the start marker.
2. Add the header fields:
      - a 1-bit flag for whether the protocol version is included.
      - if included, the 8-bit protocol version number.
      - a 5-bit message type field.
      - a 1-bit compression flag.
      - an 8-bit metadata length field.
3. Add the metadata:
     - serialize metadata as a JSON object, if any.
     - compress the metadata if compression is enabled.
     - append the serialized metadata.
4. Add the payload:
     - serialize the payload according to the message type.
     - compress the payload if compression is enabled.
     - append the serialized payload.
5. Add `0` padding bits as needed, so the total length is a multiple of 8 bits.

## Decoding process

1. Read bits until you reach the start marker (`1`).
2. Read the header fields:
     - read the `Versioned Flag` to check if the protocol version is specified.
     - if specified, read the 8-bit protocol version number.
     - read the 5-bit message type field.
     - read the `Compressed Flag`.
     - read the 8-bit metadata length field.
3. Read the metadata:
     - read the specified number of bytes for metadata.
     - decompress it if the `Compressed Flag` is set.
     - deserialize the metadata.
4. Read the payload:
     - read the remaining bits as the payload.
     - decompress it if the `Compressed Flag` is set.
     - deserialize the payload based on the message type.

## Binary payloads

The protocol supports binary payloads, so it can transmit non-textual data. It handles binary payloads based on their designated types, which tell HiveMind how to process the binary content.

The header carries the binary payload type as a 4-bit unsigned integer, after the metadata and before the payload.

| Value | Type                   | Description                                                                     |
|-------|------------------------|-----------------------------------------------------------------------------------|
| 0     | UNDEFINED              | No information provided about the binary contents.                              |
| 1     | RAW_AUDIO              | Binary content is raw audio.                                                    |
| 2     | NUMPY_IMAGE            | Binary content is an image represented as a numpy array (e.g., webcam picture). |
| 3     | FILE                   | Binary is a file to be saved; additional metadata is provided elsewhere.        |
| 4     | STT_AUDIO_TRANSCRIBE   | Full audio sentence to perform Speech-to-Text (STT) and return transcripts.     |
| 5     | STT_AUDIO_HANDLE       | Full audio sentence to perform STT and handle transcription immediately.        |
| 6     | TTS_AUDIO              | Synthesized Text-to-Speech (TTS) audio to be played.                            |

> This is how the microphone satellite streams audio to `hivemind-listener`.

## Examples

### Serialized message

For a simple message with:

- protocol version: 1
- message type: `BUS`
- no compression
- metadata: `{}`
- payload: `{"type": "speak", "data":{"utterance": "Hello"}}`

The binary representation looks like this, in bit groups:
```
1 | 1 | 00000001 | 00001 | 0 | 00000000 | <metadata> | <payload>
```
Where:

- `1` (Start Marker)
- `1` (Versioned Flag)
- `00000001` (Protocol Version)
- `00001` (Message Type: `BUS`)
- `0` (Compressed Flag)
- `00000000` (Metadata Length: 0 bytes)
- `<metadata>`: serialized metadata bytes.
- `<payload>`: serialized payload bytes.

### Binary data

For a binary payload with:

- protocol version: 1
- message type: `BINARY`
- no compression
- metadata: `{}`
- a binary payload

The binary representation looks like this, in bit groups:
```
1 | 1 | 00000001 | 00001 | 0 | 00000000 | <metadata> | 0001 | <binary_payload>
```
Where:

- `1` (Start Marker)
- `1` (Versioned Flag)
- `00000001` (Protocol Version)
- `01100` (Message Type: `BINARY`)
- `0` (Compressed Flag)
- `00000000` (Metadata Length: 0 bytes)
- `<metadata>`: serialized metadata bytes.
- `0001` (Binary Type: Raw audio)
- `<payload>`: audio bytes.

### More examples

```
<uint:1=start_marker> | <uint:1=versioned_bit> | <uint:8=protocol_version> | <uint:5=msg_type> | <uint:1=compression_bit> | <uint:8=metadata_len> | <metadata> | <payload>
```

A binarized, uncompressed message:

```
1 | 1 | XXXXXXXX | XXXXX | 0 | XXXXXXXX | <metadata> | <payload>
```

An unversioned, compressed binarized message:
```
1 | 0 | XXXXX | 1 | XXXXXXXX | <metadata> | <payload>
```

A binary, uncompressed payload message:
```
1 | 1 | XXXXXXXX | 01100 | 0 | XXXXXXXX | <metadata> | XXXX | <payload>
```

An unversioned, compressed binary payload message:
```
1 | 0 | 01100 | 1 | XXXXXXXX | <metadata> | XXXX | <payload>
```

> The binarization scheme is simple enough that the protocol could, in principle, be implemented by flashing a light.

## Compression metrics

Compression cuts payload size significantly for larger messages, but is not always efficient for small ones. Benchmarks show a reduction of up to 50% for text-heavy payloads, while small payloads see little benefit.

---
[← Transport](04_protocol.md) · [Home](index.md) · [Pairing →](03_pairing.md)
