# Handshake Protocol

This document provides an overview of the handshake protocol used in the HiveMind system, detailing how handshakes are initiated and processed from both the client (slave) and server (master) perspectives.

The handshake process establishes a secure connection between a HiveMind master and its slaves. It ensures authentication, optionally using passwords or public/private key pairs, and sets up cryptographic keys for secure communication.

For detailed code and various usage examples, you can refer to the [Poorman Handshake GitHub Repository](https://github.com/JarbasHiveMind/poorman_handshake).

---

## Handshake Types

**Password-Based Handshake**:

   - Utilizes a shared password for authentication.

   - Requires both client and server to know the password beforehand.

**PGP (Public Key) Handshake**:

   - Based on public/private key pairs.

   - The server provides a public key to the client, and the client verifies the server's authenticity.

   - Supports implicit trust for first-time connections (when no previous public key is available).

   - Uses asymmetric encryption to ensure that communication is secure and cannot be intercepted or modified.

   - The symmetric session key (AES) for further communication is transmitted encrypted with PGP public keys.


> ⚠️ PGP Handshake is a work in progress! 🚧

---

## Workflow: Server Perspective

**HELLO Message** (sent to client):

   - On connection established the server sends a `HELLO` message to the client containing:
       - `pubkey`: Public key for key-based handshake. 
       - `node_id`: friendly identifier of the server
       - `session_id`: (optional) assigned session_id for the client, in case it doesn't report it's own
     
**HANDSHAKE Request** (sent to client):

   - The server initiates the handshake by sending a `HANDSHAKE` message:
       - `handshake`: A flag indicating whether the connection will be dropped if client doesn't finalize handshake
       - `binarize`: A flag indicating whether the server supports the binarization protocol.
       - `preshared_key`: A flag indicating whether a pre-shared key is available.
       - `password`: A flag indicating whether password-based handshake is available.
       - `crypto_required`: A flag indicating whether unencrypted messages will be dropped
       - `min_protocol_version`: the lowest hivemind protocol version the server allows
       - `max_protocol_version`: the highest hivemind protocol version the server allows

**HANDSHAKE Response** (received from client):

   - If the client doesnt answer the handshake, use the pre-shared cryptographic key directly, skipping the handshake step.
   - If the client answers the handshake by sending back another `HANDSHAKE` message:
       - `session`: the client Session data (may overwrite `session_id` from `HELLO` message)
       - `site_id`: the client site_id
       - `binarize`: A flag indicating whether the client supports the binarization protocol.
       - `envelope`: the handshake envelope to be validated by the server
   - Validate the client's `envelope` using the shared password
       - Update the cryptographic key for secure communication.
       - Update the client session data and site_id
       - Send back another `HANDSHAKE` message with server `envelope` to be validated by the client


---

## Workflow: Client Perspective

**Receive HELLO Message**:

   - Extract the server's public key and node ID from the `HELLO` message:
       - `pubkey`: the server public PGP key
       - `node_id`: friendly identifier of the server
       - `session_id`: (optional) the server assigned session_id

**Receive HANDSHAKE Request**:

   - Proceed with **Password-based handshake**  by sending back another `HANDSHAKE` message:
       - `session`: the client Session data
       - `site_id`: the client site_id
       - `binarize`: A flag indicating whether the client supports the binarization protocol.
       - `envelope`: the handshake envelope to be validated by the server

**Complete HANDSHAKE**:

   - Received final`HANDSHAKE` message with server `envelope`
       - Verify the server's authenticity using the shared password.
       - Update the cryptographic key for secure communication.

---

## Secure Communication After Handshake

Upon successful handshake:

1. A shared cryptographic key is established between the server and the client.

2. All further communication between the server and client is encrypted using this symmetric key (AES-256).

3. The session ID ensures continuity and identification in multi-session environments.

This guarantees that all data exchanged between the server and the client is protected, even if intercepted by a third party.

---

## Error Handling

**Illegal Messages**:

  - Messages not adhering to the protocol are logged, and the connection may be terminated.

**Handshake Failures**:

  - Authentication failures result in handshake termination and rejection of the connection.

---

For detailed code and various usage examples, please refer to the [Poorman Handshake GitHub Repository](https://github.com/JarbasHiveMind/poorman_handshake).