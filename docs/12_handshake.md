# Handshake Protocol

This document provides an overview of the handshake protocol used in the HiveMind system, detailing how handshakes are initiated and processed from both the client (slave) and server (master) perspectives.

The handshake process establishes a secure connection between a HiveMind master and its slaves. It ensures authentication, optionally using passwords or public/private key pairs, and sets up cryptographic keys for secure communication.

For detailed code and various usage examples, you can refer to the [Poorman Handshake GitHub Repository](https://github.com/JarbasHiveMind/poorman_handshake).

---

## Handshake Types

**Password-Based Handshake**:

   - Utilizes a shared password for authentication.

   - Requires both client and server to know the password beforehand.

**RSA (Public Key) Handshake**:

   - Based on public/private key pairs.

   - The server provides a public key to the client, and the client verifies the server's authenticity.

   - Supports implicit trust for first-time connections (when no previous public key is available).

   - Uses asymmetric encryption to ensure that communication is secure and cannot be intercepted or modified.

   - The symmetric session key (AES) for further communication is transmitted encrypted with RSA public keys.


> ⚠️ RSA Handshake is a work in progress! 🚧

---

## Workflow: Server Perspective

#### **Send Server Info** `HELLO` ✉️ ➡️  

- **Trigger**: Sent immediately upon connection establishment.  
- **Content**:
      - **`pubkey`**: Public key for key-based handshake and `INTERCOM` messages.  
      - **`node_id`**: A user-friendly identifier for the server.  
- **Security**: This message is **NOT ENCRYPTED** 🔓.  


#### **Establish connection parameters** `HANDSHAKE`✉️ ➡️  

- **Trigger**: Initiates the handshake process immediately after `HELLO` message.  
- **Content**:
    - **`handshake`**: Indicates if the connection will be dropped if the client does not finalize the handshake.  
    - **`binarize`**: Specifies if the server supports the binarization protocol.  
    - **`preshared_key`**: Indicates the availability of a pre-shared key.  
    - **`password`**: Indicates the availability of password-based handshake.  
    - **`crypto_required`**: Specifies if unencrypted messages will be dropped.  
    - **`min_protocol_version`**: The minimum supported HiveMind protocol version.  
    - **`max_protocol_version`**: The maximum supported HiveMind protocol version.  
- **Security**: This message is **NOT ENCRYPTED** 🔓.  

#### **Initiate Key Exchange**  ⬅️ ✉️  `HANDSHAKE`

- **Trigger**: Client initiated handshake in response to previously sent connection parameters.  
- **Content**:
     - **`binarize`**: Specifies if the client supports the binarization protocol.  
     - **`envelope`**: The handshake envelope to be validated by the server.  
- **Behavior**:  
     - If the client does not respond, the server will skip the handshake step and use the pre-shared cryptographic key directly.  
     - Validate the client's `envelope` using the pre-shared password.  
- **Security**: This message is **NOT ENCRYPTED** 🔓.  

#### **Complete Key Exchange** `HANDSHAKE` ✉️ ➡️  

- **Trigger**: Validated client's `envelope` and updated the cryptographic key for secure communication.
- **Content**:
     - **`envelope`**: The handshake envelope to be validated by the client.  
- **Security**: This message is **NOT ENCRYPTED** 🔓.  


#### **Receive Client Info** ⬅️ ✉️ `HELLO`

- **Trigger**: Sent after the handshake is complete and encryption is established.  
- **Content**:
    - **`session`**: The client session data.  
    - **`site_id`**: The client site identifier.  
    - **`pubkey`**: Public key for `INTERCOM` messages.  
- **Security**: This message is **ENCRYPTED** 🔐.  

---

## Workflow: Client Perspective

#### **Receive Server Info** ⬅️ ✉️  `HELLO`

- **Trigger**: Received upon connection establishment.  
- **Content**:
     - **`pubkey`**: The server's public RSA key.  
    - **`node_id`**: A user-friendly identifier for the server.  
- **Security**: This message is **NOT ENCRYPTED** 🔓.  


#### **Establish connection parameters** ⬅️ ✉️  `HANDSHAKE`

- **Trigger**: Received immediately after `HELLO` message.  
- **Content**:
     - **`handshake`**: Indicates if the connection will be dropped if the client does not finalize the handshake.  
    - **`binarize`**: Specifies if the server supports the binarization protocol.  
    - **`preshared_key`**: Indicates the availability of a pre-shared key.  
    - **`password`**: Indicates the availability of password-based handshake.  
    - **`crypto_required`**: Specifies if unencrypted messages will be dropped.  
    - **`min_protocol_version`**: The minimum supported HiveMind protocol version.  
    - **`max_protocol_version`**: The maximum supported HiveMind protocol version.  
- **Security**: This message is **NOT ENCRYPTED** 🔓.  


#### **Initiate Key Exchange** `HANDSHAKE` ✉️ ➡️  

- **Trigger**: Respond to the server's handshake request.  
- **Content**:
     - **`binarize`**: Specifies if the client supports the binarization protocol.  
    - **`envelope`**: The handshake envelope to be validated by the server.  
- **Security**: This message is **NOT ENCRYPTED** 🔓.  

#### **Complete Key Exchange** ⬅️ ✉️  `HANDSHAKE`

- **Trigger**: On reception of the server's final `HANDSHAKE` message.  
- **Content**:
    - **`envelope`**: The handshake envelope to be validated by the client.  
- **Behavior**:
    - Verify the server's authenticity using the shared password.  
    - Update the cryptographic key for secure communication.  
- **Security**: This message is **NOT ENCRYPTED** 🔓.  


#### **Send Session Data** `HELLO` ✉️ ➡️  

- **Trigger**: Send session data after encryption is established.  
- **Content**:
     - **`session`**: The client session data.  
    - **`site_id`**: The client site identifier.  
    - **`pubkey`**: Public key for `INTERCOM` messages.  
- **Security**: This message is **ENCRYPTED** 🔐.  

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

**Authentication Failures**:

  - Authentication failures result in handshake termination and rejection of the connection.

**Skipping Handshake**:

  - Handshake can be skipped if a secret key has been previously exchanged out of band

![img_21.png](img_21.png)

---

For detailed code and various usage examples, please refer to the [Poorman Handshake GitHub Repository](https://github.com/JarbasHiveMind/poorman_handshake).