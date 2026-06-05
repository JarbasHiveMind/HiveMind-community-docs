# Auto Discovery

[HiveMind-presence](https://github.com/JarbasHiveMind/HiveMind-presence) enables automatic discovery of HiveMind nodes on the local network without manual address configuration. It is an optional extra package — `hivemind-core` runs without it.

## Discovery transports

**HiveBeacon (default)**: Zero-dependency UDP broadcast. The hub announces itself and satellites scan for announcements. No external dependencies. Enabled by default when `hivemind-presence` is running.

**mDNS / Zeroconf (optional)**: Standard multicast DNS discovery. Requires `zeroconf` to be installed. Useful in environments where UDP broadcast is filtered by switches.

**UPnP**: Not supported.

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
  --beacon BOOLEAN     Advertise via UDP broadcast (default: True)
  --zeroconf BOOLEAN   Advertise via mDNS/Zeroconf (default: False)
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
  --beacon BOOLEAN     Scan via UDP broadcast (default: True)
  --zeroconf BOOLEAN   Scan via mDNS/Zeroconf (default: False)
  --service-type TEXT  HiveMind service type (default: HiveMind-websocket)
```

## Audio pairing (GGWave)

> **Note**: This feature is a proof-of-concept and is a work in progress.

GGWave encodes data as audio tones, allowing a hub and satellite to exchange pairing credentials when they are in audible range of each other — useful for initial setup without a keyboard.

**Prerequisites:**
- Hub device with microphone and speaker
- Satellite device with microphone and speaker
- A browser (phone) to initiate the exchange
- All devices within audible range of each other

**Workflow:**

1. Launch `hivemind-core` — note the pairing code printed (e.g. `HMPSWD:ce357a6b59f6b1f9`)
2. Play the code as audio via the [browser GGWave tool](https://jarbashivemind.github.io/hivemind-ggwave/)
3. The satellite decodes the password, generates an access key, and sends it back as audio
4. The hub receives the key, adds the client, and sends an acknowledgement (containing the host address) as audio
5. The satellite decodes the acknowledgement and connects

After a successful GGWave pairing the satellite has a populated identity file and can connect as normal.
