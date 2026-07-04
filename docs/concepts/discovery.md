# Auto Discovery

In plain terms: instead of typing the hub's network address into every device by hand, discovery lets satellites find the hub on their own — either by listening for it on the local network, or (for first-time setup with no keyboard) by exchanging credentials through sound.

> **Auto-discovery is optional.** `hivemind-core` works fine with a manually configured `--host` (or a `default_master` in the identity file) — satellites connect straight to a known address with no discovery layer. Install [HiveMind-presence](https://github.com/JarbasHiveMind/HiveMind-presence) only if you want satellites to find the hub automatically.

[HiveMind-presence](https://github.com/JarbasHiveMind/HiveMind-presence) enables automatic discovery of HiveMind nodes on the local network without manual address configuration. It is an optional extra package — `hivemind-core` runs without it.

## Discovery transports

**mDNS / Zeroconf (default)**: Standard multicast DNS service discovery. Enabled by default (`--zeroconf` defaults to `True`). Requires the optional `zeroconf` package (LGPL, imported lazily); without it, announce/scan silently fall back to UPnP only.

**UPnP / SSDP (legacy, off by default)**: An SSDP server advertises a UPnP device descriptor; satellites scan for it. Supported but disabled by default (`--upnp` defaults to `False`). Useful where mDNS is unavailable.

**HiveBeacon (planned)**: A zero-dependency UDP-broadcast transport is on the roadmap and is intended to become the default once it ships, with mDNS kept as an optional transport. It is **not yet implemented** — until it lands, mDNS/Zeroconf and UPnP/SSDP are the available transports.

## Integration with hivemind-core

When `hivemind-presence` is installed, start the announcer alongside the hub. The `hivemind-presence announce` command is independent of `hivemind-core listen` — run them together, or configure `hivemind-presence` as a separate service on the same machine. No changes to `~/.config/hivemind-core/server.json` are required.

## Running the hub announcer

On the hub machine, start advertising:

```bash
hivemind-presence announce --port 5678 --name "living-room-hub"
```

Options:

```
Options:
  --port INTEGER       HiveMind port number (default: 5678)
  --name TEXT          Friendly device name (default: HiveMind-Node)
  --service-type TEXT  HiveMind service type (default: HiveMind-websocket)
  --zeroconf BOOLEAN   Advertise via mDNS/Zeroconf (default: True)
  --upnp BOOLEAN       Advertise via UPnP/SSDP (default: False)
  --ssl BOOLEAN        Report SSL support (default: False)
```

## Scanning for hubs

On a satellite (or any device on the same network):

```bash
hivemind-presence scan
```

Example output:

```
            HiveMind Nodes            
┏━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━┓
┃ Friendly Name ┃ Host         ┃ Port ┃
┡━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━┩
│   living_room │ 192.168.1.9  │ 5678 │
│       kitchen │ 192.168.1.13 │ 5678 │
└───────────────┴──────────────┴──────┘
```

Scan options:

```
Options:
  --zeroconf BOOLEAN   Scan via mDNS/Zeroconf (default: True)
  --upnp BOOLEAN       Scan via UPnP/SSDP (default: False)
  --service-type TEXT  HiveMind service type (default: HiveMind-websocket)
```

## Audio pairing (GGWave)

> **Note**: This feature is a proof-of-concept and is a work in progress.

GGWave encodes data as audio tones, allowing a hub and satellite to exchange pairing credentials when they are in audible range of each other — useful for initial setup without a keyboard.

**Prerequisites:**
- Hub device with microphone and speaker
- Satellite device with microphone and speaker
- Any GGWave transmitter to initiate the exchange — the [browser GGWave tool](https://jarbashivemind.github.io/hivemind-ggwave/) **or** the native `ggwave-cli` / `ggwave-rx` binaries
- All devices within audible range of each other

**Workflow:** The exchange uses three opcodes — `HMPSWD` (password), `HMKEY` (access key), and `HMHOST` (host address):

1. A password is broadcast as audio (`HMPSWD`), e.g. via the [browser GGWave tool](https://jarbashivemind.github.io/hivemind-ggwave/)
2. The satellite decodes the password, generates an access key, and sends it back as audio (`HMKEY`)
3. The hub receives the key, adds the client, and sends an acknowledgement containing the host address as audio (`HMHOST`)
4. The satellite decodes the acknowledgement and connects

After a successful GGWave pairing the satellite has a populated identity file and can connect as normal.

## See also: hivemind-rendezvous (proof-of-concept)

!!! note "Optional / proof-of-concept"
    [hivemind-rendezvous](https://github.com/JarbasHiveMind/hivemind-rendezvous) is a separate, experimental package — not part of normal discovery and not required to run a hive.

Discovery above assumes nodes are online at the same time. **hivemind-rendezvous** solves the opposite case: an asynchronous **store-and-forward dead-drop** that lets two hives which are *never* online simultaneously still exchange [`INTERCOM`](protocol.md#intercom-end-to-end-encrypted-peer-to-peer) messages. A sender deposits an encrypted message keyed by the **recipient's RSA public key**; the recipient retrieves it later by proving ownership of that key with a **signed timestamp** (no server-side challenge state). It is a proof-of-concept — useful to understand the model, but not something to design a production deployment around yet.

---

**Next:** [Security](security.md) for the handshake, credentials, and admission control, or [Mesh Topology](mesh.md) for how discovered nodes route messages.

## Source

Validated against the HiveMind source:

- [`hivemind_presence/scripts.py`](https://github.com/JarbasHiveMind/HiveMind-presence/blob/HEAD/hivemind_presence/scripts.py) — `announce` / `scan` commands and their mDNS/UPnP defaults
- [`hivemind_ggwave/__init__.py`](https://github.com/JarbasHiveMind/hivemind-ggwave/blob/HEAD/hivemind_ggwave/__init__.py) — `HMPSWD` / `HMKEY` / `HMHOST` opcodes and the `ggwave-cli` / `ggwave-rx` binaries
- [`hivemind_rendezvous/__init__.py`](https://github.com/JarbasHiveMind/hivemind-rendezvous/blob/HEAD/hivemind_rendezvous/__init__.py) — async store-and-forward dead-drop keyed by recipient RSA pubkey
