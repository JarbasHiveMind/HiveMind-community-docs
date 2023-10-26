# Auto Discovery

[Hivemind-presence](https://github.com/JarbasHiveMind/HiveMind-presence) is an utility to enable auto discovery of
HiveMind nodes in your network

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

Announce your HiveMind node in your lan via UpnP and Zeroconf

```
$ hivemind-presence announce --help
Usage: hivemind-presence announce [OPTIONS]

  Advertise node in the local network

Options:
  --port INTEGER      HiveMind port number (default: 5678)
  --name TEXT         friendly device name (default: HiveMind-Node)
  --service TEXT      HiveMind service type (default: HiveMind-websocket)
  --zeroconf BOOLEAN  advertise via zeroconf
  --upnp BOOLEAN      advertise via UPNP
  --ssl BOOLEAN       report ssl support
  --help              Show this message and exit.
```

Scan for HiveMind nodes in your lan via UpnP and Zeroconf

```
$ hivemind-presence scan --help
Usage: hivemind-presence scan [OPTIONS]

  scan for hivemind nodes in the local network

Options:
  --zeroconf BOOLEAN  scan via zeroconf
  --upnp BOOLEAN      scan via UPNP
  --help              Show this message and exit.


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