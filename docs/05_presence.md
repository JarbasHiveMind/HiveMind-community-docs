# Auto Discovery

[HiveMind-presence](https://github.com/JarbasHiveMind/HiveMind-presence) enables automatic discovery of HiveMind nodes in your local network using zero-dependency UDP broadcast (HiveBeacon), mDNS (Zeroconf), or manual address entry.

## Command line usage

```
$ hivemind-presence --help
Usage: hivemind-presence [OPTIONS] COMMAND [ARGS]...

Options:
  --help  Show this message and exit.

Commands:
  announce  Advertise node in the local network
  scan      scan for hivemind nodes in the local network
```

Announce your HiveMind node in your local network via HiveBeacon (UDP broadcast) and optionally mDNS:

```
$ hivemind-presence announce --help
Usage: hivemind-presence announce [OPTIONS]

  Advertise node in the local network

Options:
  --port INTEGER       HiveMind port number (default: 5678)
  --name TEXT          friendly device name (default: HiveMind-Node)
  --service-type TEXT  HiveMind service type (default: HiveMind-websocket)
  --beacon BOOLEAN     advertise via UDP broadcast (default: True)
  --zeroconf BOOLEAN   advertise via mDNS/Zeroconf (default: False)
  --help               Show this message and exit.

```

Scan for HiveMind nodes in your local network via HiveBeacon (UDP broadcast) and optionally mDNS:

```
$ hivemind-presence scan --help
Usage: hivemind-presence scan [OPTIONS]

  scan for hivemind nodes in the local network

Options:
  --beacon BOOLEAN     scan via UDP broadcast (default: True)
  --zeroconf BOOLEAN   scan via mDNS/Zeroconf (default: False)
  --service-type TEXT  HiveMind service type (default: HiveMind-websocket)
  --help               Show this message and exit.
```

```
$ hivemind-presence scan
            HiveMind Nodes            
┏━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━┓
┃ Friendly Name ┃ Host         ┃ Port ┃
┡━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━┩
│   living_room │ 192.168.1.9  │ 5678 │
│       kitchen │ 192.168.1.13 │ 5678 │
└───────────────┴──────────────┴──────┘

```