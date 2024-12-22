# Binarization Protocol

The HiveMind Binarization Protocol is designed to efficiently serialize and deserialize structured messages into compact binary formats for network transmission. This document provides a high-level description of the protocol, including its structure, encoding rules, and the rationale behind key design decisions. The binary format is protocol-versioned to support backward compatibility and future extensions.

> 💡 the binarization scheme allows the hivemind protocol to be implemented by just flashing a light

## Protocol Versions

The protocol uses an integer version number to indicate supported features and ensure compatibility between clients and servers. The current protocol version is `1`. Any change in functionality or structure requires incrementing the version number.

Version-specific functionality:

- **Version 0**: Original protocol design. No binarization, no handshake, only pre-shared `crypto_key` supported

- **Version 1**: Introduces support for handshakes and binary payloads.

## Core Concepts

### Message Types
Messages in the HiveMind protocol are categorized into various types, each serving a specific role. These types are encoded as 5-bit unsigned integers, enabling up to 32 distinct types. Examples include:

| Value | Type            | Description                      |
|-------|-----------------|----------------------------------|
| 0     | HANDSHAKE       | Initial connection handshake.   |
| 1     | BUS             | Standard message bus.           |
| 2     | SHARED_BUS      | Shared bus for multiple nodes.  |
| 3     | BROADCAST       | Global message broadcast.       |
| 4     | PROPAGATE       | Directed message propagation.   |
| 12    | BINARY          | Raw binary payload.             |

### Compression
Payloads can optionally be compressed using the zlib library. A single bit in the header indicates whether compression is applied. Compressed payloads reduce transmission size but may add slight computational overhead during encoding and decoding.

### Metadata (HiveMeta)
HiveMeta is a reserved field for attaching arbitrary metadata to a message. The metadata is encoded as a byte array, prefixed by its size (in bytes). This allows for extensible features like routing hints or debug information.

## Binary Message Structure

The serialized binary message consists of a header and a payload. All fields are packed to maximize efficiency. The structure is as follows:

### Header
The header contains information about the protocol version, message type, compression, and metadata length.

| Field           | Size (bits) | Description                                    |
|------------------|-------------|------------------------------------------------|
| Start Marker     | 1           | Always `1`. Helps align message boundaries.   |
| Versioned Flag   | 1           | Indicates if protocol version is specified.   |
| Protocol Version | 8           | Protocol version (if `Versioned Flag` is `1`).|
| Message Type     | 5           | Encoded message type.                         |
| Compressed Flag  | 1           | Indicates if payload is compressed.           |
| Metadata Length  | 8           | Length of metadata in bytes.                  |

### Metadata
Metadata is optional and encodes key-value pairs or other information. If present, it follows the header and is serialized as a byte array. The length of the metadata is specified in the header.

### Payload
The payload represents the core message data. Its format depends on the message type:

- **For standard messages**: Encoded as a UTF-8 JSON string.

- **For binary messages**: Encoded as raw bytes with an additional 4-bit unsigned integer indicating the binary payload type.

### Padding
To ensure byte alignment, padding bits (`0`) are inserted as needed. The total length of the message must be a multiple of 8 bits.

## Encoding Process

1. **Start Marker**: Add a single bit set to `1` to signify the start of the message.

2. **Header Fields**:
   - Add a 1-bit flag to indicate whether the protocol version is included.
   - If the version is included, append the 8-bit protocol version number.
   - Add a 5-bit message type field.
   - Add a 1-bit flag to indicate compression status.
   - Add an 8-bit metadata length field.
   
3. **Metadata**:
   - Serialize metadata as a JSON object (if any).
   - Compress the metadata if compression is enabled.
   - Append the serialized metadata.
   
4. **Payload**:
   - Serialize the payload according to the message type.
   - Compress the payload if compression is enabled.
   - Append the serialized payload.
   
5. **Padding**: Add `0` bits as needed to ensure the total length is a multiple of 8 bits.

## Decoding Process

1. **Alignment**: Read bits until encountering the start marker (`1`).

2. **Header Fields**:
   - Read the `Versioned Flag` and determine if the protocol version is specified.
   - If specified, read the 8-bit protocol version number.
   - Read the 5-bit message type field.
   - Read the `Compressed Flag`.
   - Read the 8-bit metadata length field.
   
3. **Metadata**:
   - Read the specified number of bytes for metadata.
   - Decompress if the `Compressed Flag` is set.
   - Deserialize the metadata.
   
4. **Payload**:
   - Read the remaining bits as the payload.
   - Decompress if the `Compressed Flag` is set.
   - Deserialize the payload based on the message type.

## Binary Payloads

The protocol provides support for binary payloads, enabling the transmission of non-textual data. Binary payloads are handled based on their designated types, which instruct the HiveMind how to process the binary content. 

The binary payload type is indicated in the header as a **4 bit unsigned integer** after the metadata and before the payload


| Value | Type                   | Description                                                                     |
|-------|------------------------|---------------------------------------------------------------------------------|
| 0     | UNDEFINED              | No information provided about the binary contents.                              |
| 1     | RAW_AUDIO              | Binary content is raw audio.                                                    |
| 2     | NUMPY_IMAGE            | Binary content is an image represented as a numpy array (e.g., webcam picture). |
| 3     | FILE                   | Binary is a file to be saved; additional metadata is provided elsewhere.        |
| 4     | STT_AUDIO_TRANSCRIBE   | Full audio sentence to perform Speech-to-Text (STT) and return transcripts.     |
| 5     | STT_AUDIO_HANDLE       | Full audio sentence to perform STT and handle transcription immediately.        |
| 6     | TTS_AUDIO              | Synthesized Text-to-Speech (TTS) audio to be played.                            |

> 💡 this how how the microphone satellite streams audio to `hivemind-listener`

## Examples

### Serialized Message

For a simple message with:

- Protocol version: 1

- Message type: `BUS`

- No compression

- Metadata: `{}`

- Payload: `{"type": "speak", "data":{"utterance": "Hello"}}`

The binary representation might look like this (in bit groups):
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

- `<metadata>`: Serialized metadata bytes.

- `<payload>`: Serialized payload bytes.


### Binary data

For a binary payload with:

- Protocol version: 1

- Message type: `BINARY`

- No compression

- Metadata: `{}`

- Binary Payload

The binary representation might look like this (in bit groups):
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

- `<metadata>`: Serialized metadata bytes.

- `0001` (Binary Type: Raw audio)

- `<payload>`: audio bytes.


## Compression Metrics

Compression significantly reduces payload size for larger messages but is not always efficient for small messages. Benchmarks indicate a reduction of up to **50%** for text-heavy payloads, while small payloads may see negligible benefits.

## Implementation Notes

- Bit-level operations are critical for compact encoding. Ensure precision when handling individual bits.

- Maintain strict alignment rules to avoid deserialization errors.

- Use a modular design to allow future extensions while retaining compatibility.


