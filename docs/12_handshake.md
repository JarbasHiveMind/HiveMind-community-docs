# Handshake Protocol

This document provides an overview of the handshake protocol used in the HiveMind system, detailing how handshakes are initiated and processed from both the client (slave) and server (master) perspectives.

The handshake process establishes a secure connection between a HiveMind master and its slaves. It ensures authentication, optionally using passwords or public/private key pairs, and sets up cryptographic keys for secure communication.

For detailed code and various usage examples, you can refer to the [Poorman Handshake GitHub Repository](https://github.com/JarbasHiveMind/poorman_handshake).

---

## Handshake Types

**Password-Based Handshake**:

   - Utilizes a shared password for authentication.

   - Requires both client and server to know the password beforehand.

**Public Key Handshake**:

   - Based on public/private key pairs.

   - The server provides a public key to the client, and the client verifies the server's authenticity.

   - Supports implicit trust for first-time connections (when no public key is available).

   - Uses asymmetric encryption (RSA, for example) to ensure that communication is secure and cannot be intercepted or modified.

   - Encrypts the symmetric session key to allow further communication using the shared key.

---

## Workflow: Server Perspective

**HELLO Message**:

   - The server sends a `HELLO` message to the client containing:
   
     - Public key (`pubkey`) for key-based handshake.
     
     - Node ID (`node_id`) for identification.
     
     - Optional `session_id` for session-based communication.

**HANDSHAKE Request**:

   - The server initiates the handshake by sending a `HANDSHAKE` message:
   
     - Specifies whether to use password-based or public-key-based authentication.
     
     - Includes optional fields like:
     
       - `crypto_key`: A flag indicating whether a pre-shared cryptographic key is available for use in the handshake (but not the key itself).
       
       - `binarize`: Flag for binary protocol support.
       
       - `password`: Indicator for password-based handshake.

**Validate Client's Response**:

   - If the client provides an envelope:
   
     - Validate the client's response using the shared password or public key.
     
     - Update the cryptographic key for secure communication.
     
   - If the `crypto_key` flag is set or the client doesnt answer the handshake, use the pre-shared cryptographic key directly, skipping the handshake step.

---

## Workflow: Client Perspective

**Receive HELLO Message**:

   - Extract the server's public key and node ID from the `HELLO` message.

   - Store the session ID if provided.

**Start Handshake**:

   - Determine the handshake type based on the server's `HANDSHAKE` request:
   
     - Password-based handshake:
     
       - Generate an envelope using the shared password.
       
     - Public-key-based handshake:
     
       - Verify the server's public key (if available).
       
       - Generate and send an envelope for authentication.

**Handle Validation**:

   - If the server sends an envelope for validation:
   
     - Verify the server's authenticity using the shared password or public key.
     
     - Update the cryptographic key for secure communication.

---

## Handshake Message Structure

### HELLO Message

- **From Server**:

```json
{
"type": "HELLO",
"payload": {
  "pubkey": "<server_public_key>",
  "node_id": "<server_node_id>",
  "session_id": "<session_id (optional)>"
}
}
```

### HANDSHAKE Message

- **From Server**:

```json
{
"type": "HANDSHAKE",
"payload": {
  "password": "<bool>",
  "crypto_key": "<bool (flag indicating availability of pre-shared key)>",
  "binarize": "<bool>",
  "envelope": "<handshake_envelope (if client has started)>"
}
}
```

- **From Client**:

```json
{
"type": "HANDSHAKE",
"payload": {
  "pubkey": "<client_public_key (if using pubkey)>",
  "envelope": "<handshake_envelope>",
  "binarize": "<bool>",
  "session": "<session_data>",
  "site_id": "<client_site_id>"
}
}
```

---

## Key Functions and Responsibilities

### Server

**Start Handshake**:

  - Ensure the client is authorized to join the HiveMind network.
  
**Broadcast Key**:

  - Send the server's public key for public-key-based handshakes.
  
**Verify Envelope**:

  - Authenticate the client using the received envelope and establish the shared cryptographic key.

### Client

**Generate Envelope**:

  - Create an envelope for authentication based on the handshake type.

**Verify Server**:

  - Use the public key to verify the server's authenticity.

**Update Session**:

  - Store the server-provided session ID and synchronize it with local sessions.

---

## Secure Communication After Handshake

Upon successful handshake:

1. A shared cryptographic key is established between the server and the client.

2. All further communication between the server and client is encrypted using this symmetric key (e.g., AES-256).

3. The session ID ensures continuity and identification in multi-session environments.

This guarantees that all data exchanged between the server and the client is protected, even if intercepted by a third party.

---

## Error Handling

**Illegal Messages**:

  - Messages not adhering to the protocol are logged, and the connection may be terminated.

**Handshake Failures**:

  - Authentication failures result in handshake termination and rejection of the connection.

---

## Example Scenarios

### First-Time Connection (Implicit Trust)
1. Server sends `HELLO` with its public key.
2. Client trusts the server and starts the handshake.
3. A shared cryptographic key is established for encrypted communication.

### Reconnection with Password
1. Server requests a password-based handshake.
2. Client generates an envelope using the shared password.
3. Server validates the envelope and establishes a secure session.

---

For detailed code and various usage examples, please refer to the [Poorman Handshake GitHub Repository](https://github.com/JarbasHiveMind/poorman_handshake).