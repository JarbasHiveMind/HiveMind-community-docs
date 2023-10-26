# Auto Discovery

[Hivemind-presence](https://github.com/JarbasHiveMind/HiveMind-presence) is an utility to enable auto discovery of HiveMind nodes in your network

## Command line usage

Announce your HiveMind node in your lan via UpnP and Zeroconf

```
[user@user-predatorph31551 HiveMind-presence]$ HiveMind-announce --help
usage: HiveMind-announce [-h] [--port PORT] [--ssl] [--zeroconf] [--name NAME] [--service SERVICE]

optional arguments:
  -h, --help         show this help message and exit
  --port PORT        HiveMind port number (default: 5678)
  --ssl              use wss://
  --zeroconf         use zeroconf
  --name NAME        friendly device name (default: HiveMind-Node)
  --service SERVICE  HiveMind service type (default: HiveMind-websocket)

```

Scan for HiveMind nodes in your lan via UpnP and Zeroconf

```
[user@user-predatorph31551 HiveMind-presence]$ HiveMind-scan
Scanning....
                 HiveMind Devices                  
┏━━━━━━━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━━━━━━━┳━━━━━━┓
┃     Name      ┃ Protocol ┃     Host      ┃ Port ┃
┡━━━━━━━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━━━━━━━╇━━━━━━┩
│ HiveMind Node │   wss    │ 192.168.1.112 │ 5678 │
└───────────────┴──────────┴───────────────┴──────┘
```