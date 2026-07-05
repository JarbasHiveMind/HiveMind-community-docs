# Security

Think of adding a device to your hive like handing someone a key to the house. Before
they get in, they have to prove they hold the right key — without ever sliding it under
the door where someone could copy it. Once they're inside, they still can't touch
everything: a new device arrives able to do *nothing at all*, and you decide, one
permission at a time, what it's allowed to say. And everything spoken between the device
and the server is scrambled the moment the door closes, whether or not you ever bother
with TLS. Those three ideas — prove the key, grant nothing by default, encrypt
everything — are the whole security model.

!!! abstract "In a nutshell"
    - The `access_key` is a non-secret identifier sent in the clear; the `password` is the only secret and is never transmitted — both sides derive the session key from it.
    - Protocol v3 runs a Noise handshake (`Noise_XXpsk2_25519_ChaChaPoly_SHA256`) for an always-encrypted, forward-secret session; older clients fall back to a PBKDF2 password handshake.
    - Permissions are fail-closed: a new client can send nothing until you `allow-msg` each message type, and the admin flag never bypasses the `allowed_types` ACL.
    - TLS is optional defence-in-depth; HiveMind's own handshake already encrypts every payload.

!!! tip "Three layers protect a HiveMind"
    1. **Handshake + encryption** — a connecting device proves it knows the password and the two sides agree on a session key without ever sending it over the wire. On protocol **v3** this is a **Noise** handshake and the whole session is encrypted end to end.
    2. **Permissions** — hivemind-core checks every message against a per-device allowlist; an unknown message type is dropped.
    3. **Transport (TLS)** — an *optional* outer layer of standard WebSocket encryption. HiveMind's own handshake already encrypts every payload, so **TLS is not required** — it is defence-in-depth, useful mainly to hide metadata from a network observer.

---

## Credentials: access key vs. password

Every satellite is issued two things by `add-client`:

- **`access_key`** — a **non-secret identifier**, like a username. It is sent in the clear in the WebSocket `authorization` header so hivemind-core knows *which* client is connecting. Leaking it does not compromise the hive.
- **`password`** — the **only secret**, and it is **never transmitted**. Both sides derive the session key from it; a connecting peer that does not know the password simply fails the handshake and is disconnected. There is no password-recovery path over the wire — knowledge of the password is proven, never revealed.

Because the password is the sole secret, its strength is the security of the whole hive (see [Weak-password refusal](#weak-password-refusal)).

---

## Handshake and encryption

### Protocol v3 — the Noise handshake (current default)

Protocol **v3** replaces the legacy handshake with the **[Noise Protocol Framework](https://noiseprotocol.org/)**. A v3-capable client (Noise primitive available and a password configured) negotiates an always-encrypted, forward-secret session:

- **Default suite:** `Noise_XXpsk2_25519_ChaChaPoly_SHA256` — mutual static-key authentication over X25519, per-handshake ephemeral Diffie-Hellman (forward secrecy), ChaCha20-Poly1305 AEAD, SHA-256. This is the suite browser and JavaScript clients ([HiveMind-js](https://github.com/JarbasHiveMind/HiveMind-js)) negotiate too: they pair the native Web Crypto API with the pure-JS [`@noble/ciphers`](https://github.com/paulmillr/noble-ciphers) + [`@noble/hashes`](https://github.com/paulmillr/noble-hashes) for the two primitives Web Crypto lacks (ChaCha20-Poly1305 and argon2id), giving **full cipher parity with hivemind-core** — no server-side configuration required. A `Noise_XXpsk2_25519_AESGCM_SHA256` suite is also registered as an AES-GCM alternative for minimal Web-Crypto-only peers (see the browser caveat below).
- **PSK derivation:** the shared secret mixed into the handshake is `PSK = argon2id(password, salt = SHA-256(node_id))`. Salting with the server's `node_id` makes the PSK server-specific, so the same password on two servers yields two different PSKs. A full browser client derives this **in-browser** with `@noble/hashes` argon2id (same parameters as the server), so a password alone is enough — no provisioning and no server-side KDF change. A pre-provisioned 32-byte `psk` remains an option, and PBKDF2 remains an explicit fallback when a server advertises it.
- **Pinned-key case:** once a client's static key has been pinned by a prior `XXpsk2` handshake, both ends may use `Noise_KKpsk0_25519_ChaChaPoly_SHA256` (both static keys known in advance).
- Only `HELLO` and `HANDSHAKE` frames travel unencrypted (they precede session-key establishment). On a `crypto_required` server, any other cleartext frame is rejected and the client disconnected.

!!! info "Constrained devices use a provisioned PSK"
    Microcontrollers (ESP32, MicroPython) cannot run argon2id on-device. Instead you **pre-compute** the 32-byte PSK on the server and flash it onto the device:

    ```bash
    hivemind-core derive-psk --password <site_password> --node-id <server_node_id>
    ```

    This prints the hex PSK, which equals `argon2id(password, SHA-256(node_id))` — identical to what a capable peer derives at connect time, so a provisioned device and a password-deriving device interoperate with no server-side distinction. The device never sees the password itself.

!!! note "Browser caveat"
    Browsers are **not** constrained: with `@noble` loaded (a five-line ESM shim exposing `chacha20poly1305` + `argon2id` on `globalThis.HiveMindNoble`) a HiveMind-js client negotiates the default ChaChaPoly suite and derives the argon2id PSK on-device, exactly like a Python client. Only a **minimal** browser bundle shipped *without* `@noble` degrades to the AES-GCM + PBKDF2 subset, and then needs either a provisioned `psk` or a PBKDF2-advertising server.

### Legacy handshake (protocol v1 / v2)

Older clients that cannot negotiate v3 fall back to the password (or RSA) handshake. It also establishes a shared session key without transmitting it. The password handshake uses a salted hash for proof and PBKDF2-SHA256 for the key:

```
Server → Client  HELLO        (node_id, server public key — plaintext)
Server → Client  HANDSHAKE    (capabilities: binarize, crypto_required, etc. — plaintext)
Client → Server  HANDSHAKE    (binarize flag + PBKDF2 envelope — plaintext)
Server → Client  HANDSHAKE    (PBKDF2 envelope — plaintext)
Client → Server  HELLO        (session data, site_id, client public key — ENCRYPTED)
```

The handshake works as follows:

1. Each side generates a random IV and computes a salted-hash subject `HSUB = IV ‖ SHA256(IV + password)` (the IV concatenated with the hash, not a PBKDF2 value)
2. Both sides exchange their `HSUB` and `IV`
3. Each side verifies the other's `HSUB` by recomputing it locally
4. A common salt is derived: `salt = IV_client XOR IV_server`
5. The session key is derived with PBKDF2: `key = PBKDF2-HMAC-SHA256(password, salt, 100_000 iterations)` — 256 bits

After the handshake all messages are encrypted with the negotiated cipher using the derived session key. The cipher is **negotiated, not fixed**: the client offers an ordered list of ciphers it supports, the server filters that list against its own config `allowed_ciphers` (default `["CHACHA20-POLY1305", "AES-GCM"]`, with ChaCha20-Poly1305 listed first), and the server picks the client's most-preferred surviving choice. Both **ChaCha20-Poly1305** and **AES-GCM** are supported. Each message carries a unique nonce and an authentication tag. (AES-256-GCM is only the dataclass fallback used when a side offers no cipher list at all.)

An alternative v0 path skips the handshake and uses a pre-shared `crypto_key` directly (the legacy `Encryption Key` printed by `add-client`). This path is kept for backward compatibility only.

### Refusing old clients — `min_protocol_version`

The server will only admit a client that can negotiate at least `min_protocol_version` (config key in `~/.config/hivemind-core/server.json`, **default `2`**). At the default, the oldest JSON-only / no-binary **v0 and v1** clients are refused outright; a v2 client still connects with the legacy binary+handshake path, and a v3 client gets the Noise session. Raise the floor to `3` to require Noise from every client. The `ProtocolVersion` enum is `ZERO`/`ONE` (handshake) / `TWO` (binary) / `THREE` (Noise).

### Weak-password refusal

Because the password is the only secret and its verifier is offline-crackable, `poorman_handshake` **refuses guessable passwords** (`WeakPasswordError`). It estimates guess-resistance with [zxcvbn](https://github.com/dropbox/zxcvbn) and rejects anything below `min_password_bits` (default **40 bits** — enough to reject `Password123!`, `correct_password`, or the xkcd `Tr0ub4dour&3`, while real passphrases pass).

The check runs in **two places**:

1. **At `add-client` (ingestion)** — a weak `--password` is rejected before the credential is ever stored. Override for a known high-entropy machine-generated secret with `--allow-weak-password`.
2. **At handshake time (runtime backstop)** — re-checked when a client connects, in case the credential database was edited by hand. Disable this backstop with config `runtime_password_strength_check: false` or the env var `HIVEMIND_DISABLE_PASSWORD_STRENGTH_CHECK=1`.

??? note "Advanced: exact handshake parameters"
    - The handshake IV is **64-bit (8 bytes)**, generated with `os.urandom` (`generate_iv`).
    - The common salt is `salt = IV_client XOR IV_server`.
    - The session key is `PBKDF2-HMAC-SHA256(password, salt, 100_000 iterations)`, producing a **256-bit** key.

    These are defined in `poorman_handshake/symmetric/__init__.py` and `symmetric/utils.py`.

### RSA identity (asymmetric)

Each node maintains an **RSA 2048-bit** key pair, PEM-encoded and stored at `~/.config/hivemind/HiveMindComs.pem`. Encryption uses PKCS#1 OAEP; signatures use PSS with SHA-256. The public key is exchanged in the encrypted `HELLO` after the handshake. The RSA keys are used for:

- **INTERCOM** — encrypting point-to-point messages so intermediate nodes cannot read them
- **Node authentication** — verifying sender identity on INTERCOM messages via signature

Reset the RSA key at any time with `hivemind-client reset-pgp` (the command name is retained; it recreates the RSA key pair).

??? note "Advanced: how INTERCOM actually encrypts arbitrary-length payloads"
    Bare RSA-OAEP can only encrypt a few hundred bytes, so INTERCOM uses a **hybrid RSA + AES-256-GCM** scheme (`hybrid_encrypt_RSA` in `poorman_handshake/asymmetric/utils.py`): a fresh random 256-bit AES key encrypts the payload with AES-GCM, and only that 32-byte AES key is wrapped with the recipient's RSA public key (PKCS#1 OAEP). This removes the RSA size limit while keeping end-to-end confidentiality.

    Separately, the **RSA handshake path** (an alternative to the password handshake) derives the session secret by XOR-ing both sides' 32-byte secrets together, so neither side alone determines the key.

#### Bootstrapping satellite-to-satellite trust

For INTERCOM (and the signed trust checks on PROPAGATE / CASCADE), a node only accepts messages from peers whose public key it has been told to trust. That trust is configured out of band: each node maintains a `trusted_keys` mapping (alias → public-key string) in its identity file. Add a peer's key with `NodeIdentity.add_trusted_key(alias, pubkey)` (and `trusted_keys` / `remove_trusted_key` to read or revoke). After PING discovery has mapped the network, `HiveMapper.mark_trusted_nodes(trusted_keys)` flips each discovered node's `trusted` flag based on that mapping, so subsequent source-trust checks resolve quickly. Keys must be exchanged through a channel you already trust — there is no automatic key acceptance.

---

## Identity file

The identity file at `~/.config/hivemind/_identity.json` stores all credentials a satellite needs to connect:

| Field | Description |
|---|---|
| `access_key` | Non-secret client identifier assigned by hivemind-core (sent in the clear, like a username) |
| `password` | The only secret; used to derive the session key (v3 Noise PSK, or legacy PBKDF2). Never transmitted |
| `default_master` | Server host address |
| `default_port` | Server port (default 5678 for WebSocket) |
| `site_id` | Physical location identifier injected into OVOS context |
| `public_key` | RSA public key string |
| `secret_key` | Path to the RSA private key (PEM) file |

Write the identity file:

```bash
hivemind-client set-identity \
  --key <access_key> \
  --password <password> \
  --host <hub_host> \
  --port 5678 \
  --siteid living-room
```

---

## Permissions

Permissions in HiveMind are configured per client — there are no roles. Each client has its own `allowed_types` whitelist and per-client skill/intent blacklists.

### How the policy chain works

For every message received from a satellite, hivemind-core runs a policy admission chain:

1. **`allowed_types` check** — the client's per-client whitelist is checked first. If the OVOS message type is not in the allowed list, the message is dropped immediately. This is fail-closed: a new client with an empty whitelist can send nothing.

2. **Policy plugins** — configured policies run in order. The default policy is `OVOSAgentPolicy`, which reads `Client.metadata` to build per-client session blacklists (skills and intents).

Admin flag (`make-admin`) does **not** bypass the `allowed_types` check. It signals to policy plugins that extra-privileged operations are permitted, but the hard ACL is always enforced first.

### Writing a custom policy

Admission control beyond the `allowed_types` ACL is an extension point. A custom policy is a plugin that subclasses `PolicyPlugin` (from `hivemind_plugin_manager.policy`) and is registered under the `hivemind.policy` entry-point group. hivemind-core loads it into an ordered **policy chain** and calls it for every inbound message.

**The contract.** Override any of three hooks:

- `review(message, client)` — inspect a Mycroft `Message` before it is emitted onto the agent bus. Return a `Verdict`.
- `review_binary(payload, client)` — same, for binary payloads (e.g. raw audio). The default allows everything, so override it only if you care about binaries.
- `observe(message, client)` — fire-and-forget hook called *after* a message was successfully emitted. Use for counters, audit logs, telemetry. Must not raise.

A `Verdict` is either an allow or a deny:

- `Verdict.allow(*mutations)` — let the message proceed. Optionally carries one or more `Mutation` objects describing changes to apply before the next policy runs. (Concrete mutation types are agent-specific and ship with the agent plugin, e.g. the OVOS bridge's skill/intent blacklist mutations — they are not part of the base primitives.)
- `Verdict.deny(code, reason="", **data)` — drop the message and short-circuit the chain. `code` is a stable machine-readable string; the `DenyCodes` enum provides the built-in codes (`POLICY_ERROR`, `POLICY_CHAIN_UNAVAILABLE`, `ACL_DISALLOWED_TYPE`, `SESSION_ID_DEFAULT_FORBIDDEN`), but custom policies may emit their own string codes.

The chain is **fail-closed**: an exception raised inside `review` / `review_binary` (or a mutation's `apply`) is converted to `Verdict.deny("policy_error", ...)`. There is no operator knob to make it lenient. `Client.is_admin` is informational only — the chain runner never skips a policy based on it; a policy that wants to treat admins specially checks `client.is_admin` itself.

When a policy denies a message, the originating client is notified on the bus with a `hive.policy.denied` message carrying the `denied_type`, `code`, `reason`, and `data` from the verdict.

**The chain.** Operators configure the chain as an ordered list under `policy.chain` in `~/.config/hivemind-core/server.json`:

```json
{
  "policy": {
    "chain": [
      {"module": "hivemind-intent-quota-policy", "config": {"limit": 100}},
      {"module": "my-custom-policy", "config": {}, "optional": true}
    ]
  }
}
```

Each entry names a `module` (the entry-point name), an optional `config` dict passed to the plugin, and an optional `optional` flag. An `optional: true` policy that raises is logged and skipped (the chain continues); a mandatory policy that raises fails the chain closed. The built-in `MessageTypeACLPolicy` (the `allowed_types` ACL) is **always force-prepended** to the chain and is mandatory — it cannot be removed, made optional, or reordered, even if you list it explicitly. If the chain fails to build at startup, hivemind-core installs a `DenyAllPolicy` fallback that rejects everything until the config is fixed.

**Minimal skeleton.** A policy plugin and its entry point:

```python
# my_policy/__init__.py
from hivemind_plugin_manager.policy import PolicyPlugin, Verdict


class MyPolicy(PolicyPlugin):
    def review(self, message, client):
        if message.msg_type == "some.forbidden.type":
            return Verdict.deny("forbidden_type",
                                "this type is never allowed here")
        return Verdict.allow()
```

```toml
# pyproject.toml
[project.entry-points."hivemind.policy"]
"my-custom-policy" = "my_policy:MyPolicy"
```

See [Writing Plugins — Policy plugins](../developers/writing-plugins.md#5-policy) for the full plugin-authoring walkthrough.

### Managing permissions via CLI

The node ID is a **positional** argument (not a `--node-id` option). If omitted, the command prompts you to pick a client.

```bash
# Allow a message type
hivemind-core allow-msg "speak" 2

# Remove an allowed message type
hivemind-core blacklist-msg "speak" 2

# Allow/deny ESCALATE from a client
hivemind-core allow-escalate 2
hivemind-core blacklist-escalate 2

# Allow/deny PROPAGATE from a client
hivemind-core allow-propagate 2
hivemind-core blacklist-propagate 2

# Blacklist a skill (OVOS-policy)
hivemind-core blacklist-skill "skill-homeassistant.openvoiceos" 2

# Un-blacklist a skill
hivemind-core allow-skill "skill-homeassistant.openvoiceos" 2

# Blacklist an intent (OVOS-policy)
hivemind-core blacklist-intent "HomeAssistant.DeviceControllerIntent" 2

# Un-blacklist an intent
hivemind-core allow-intent "HomeAssistant.DeviceControllerIntent" 2

# Grant admin flag
hivemind-core make-admin 2

# Revoke admin flag
hivemind-core revoke-admin 2

# Set arbitrary metadata (read by policy plugins)
hivemind-core set-metadata 2 --key role --value guest
```

### Per-client defaults

When a client is added via `add-client`, its `allowed_types` whitelist is **empty** — it is denied on every message until you explicitly `allow-msg` each message type it needs. This applies to admin clients too: the admin flag does **not** exempt a client from the `allowed_types` ACL (see *How the policy chain works* above). This deny-all-by-default posture ensures that compromised credentials give an attacker no capability until access is granted.

---

## Transport security (TLS)

TLS is **optional**. HiveMind's own handshake already encrypts and authenticates every payload (Noise on v3, negotiated AEAD on v1/v2), so a hive is fully secure over plain WebSocket. Add TLS only as defence-in-depth — chiefly to hide connection metadata from a passive network observer, or to satisfy an external requirement. `hivemind-core listen` takes no flags — TLS is configured in `~/.config/hivemind-core/server.json` under `network_protocol.hivemind-websocket-plugin`:

```json
{
  "network_protocol": {
    "hivemind-websocket-plugin": {
      "host": "0.0.0.0",
      "port": 5678,
      "ssl": true,
      "cert_dir": "/path/to/certs",
      "cert_name": "mycert"
    }
  }
}
```

**For local networks**: a self-signed certificate is sufficient. Pass `--selfsigned` on satellite commands to accept it.

**For internet-facing deployments**: use a reverse proxy (nginx, Caddy, Traefik) with valid certificates from Let's Encrypt. Keep HiveMind on an internal port and expose only the proxy externally. Do not expose port 5678 directly to the internet.

---

## Security checklist

**Local/private networks:**

- [ ] Use a strong, randomly generated password (12+ chars, mixed case and symbols)
- [ ] Store the password in an environment variable, not in plain config files
- [ ] Firewall port 5678 to trusted subnets only
- [ ] Monitor logs for repeated failed handshakes

**Internet-facing deployments:**

- [ ] Deploy a reverse proxy (nginx proxy manager, Caddy, Traefik)
- [ ] Obtain valid TLS certificates (Let's Encrypt)
- [ ] Enable automatic certificate renewal
- [ ] Keep port 5678 unexposed to the public internet (behind the proxy only)
- [ ] Apply rate limiting and firewall rules at the proxy

---

## Limitations

- Security is proportional to password entropy. Weak passwords are the primary attack surface — which is why `poorman_handshake` refuses guessable ones (see [Weak-password refusal](#weak-password-refusal)).
- Without TLS, a network observer can see the encrypted ciphertext and connection metadata but not payload content. TLS adds metadata-hiding defence-in-depth; it is not required for confidentiality.
- INTERCOM authentication requires both nodes to have pre-exchanged public keys through a trusted channel.

---

**Next:** [Mesh Topology](mesh.md) for how permissions compose across nested hives, or [Writing Plugins](../developers/writing-plugins.md#5-policy) to ship a custom admission policy.

---

## Source

Validated against the HiveMind source:

- [`hivemind_core/policy.py`](https://github.com/JarbasHiveMind/HiveMind-core/blob/HEAD/hivemind_core/policy.py) — `MessageTypeACLPolicy` and the fail-closed policy chain; admins are bound by `allowed_types`
- [`poorman_handshake/noise/__init__.py`](https://github.com/JarbasHiveMind/poorman_handshake/blob/HEAD/poorman_handshake/noise/__init__.py) — v3 Noise suites (`XXpsk2`/`KKpsk0`, ChaChaPoly/AES-GCM), `derive_psk = argon2id(password, SHA-256(node_id))`
- [`poorman_handshake/symmetric/strength.py`](https://github.com/JarbasHiveMind/poorman_handshake/blob/HEAD/poorman_handshake/symmetric/strength.py) — `WeakPasswordError`, zxcvbn-based 40-bit floor
- [`hivemind_core/config.py`](https://github.com/JarbasHiveMind/HiveMind-core/blob/HEAD/hivemind_core/config.py) — `min_protocol_version` (default 2), `min_password_bits`, `runtime_password_strength_check`
- [`poorman_handshake/symmetric/__init__.py`](https://github.com/JarbasHiveMind/poorman_handshake/blob/HEAD/poorman_handshake/symmetric/__init__.py) — legacy password handshake, salt = IV XOR IV, PBKDF2-HMAC-SHA256 100k iters
- [`poorman_handshake/asymmetric/utils.py`](https://github.com/JarbasHiveMind/poorman_handshake/blob/HEAD/poorman_handshake/asymmetric/utils.py) — hybrid RSA + AES-256-GCM INTERCOM encryption
- [`hivemind_bus_client/identity.py`](https://github.com/JarbasHiveMind/hivemind-websocket-client/blob/HEAD/hivemind_bus_client/identity.py) — identity file fields and `trusted_keys`
