# HiveMind-Core Configuration

HiveMind-Core uses a **JSON-based configuration file** to manage its core behavior and plugins.
By default, this file is located at:

```
~/.config/hivemind-core/server.json
```

It defines **protocols, plugins, and security options** for the HiveMind node.

---

## Example Configuration

The configuration includes agent, network, database, and binary protocol settings:

```jsonc
{
  "binarize": false,
  "allowed_encodings": [
    "JSON-B64",
    "JSON-URLSAFE-B64",
    "JSON-B32",
    "JSON-HEX"
  ],
  "allowed_ciphers": [
    "CHACHA20-POLY1305",
    "AES-GCM"
  ],
  "agent_protocol": {
    "module": "hivemind-ovos-agent-plugin",
    "hivemind-ovos-agent-plugin": {
      "host": "127.0.0.1",
      "port": 8181
    }
  },
  "binary_protocol": {
    "module": "hivemind-audio-binary-protocol-plugin",
    "hivemind-audio-binary-protocol-plugin": {
      "stt": {
        "module": "XXX-plugin",
        "XXX-plugin": {}
      },
      "tts": {
        "module": "XXX-plugin",
        "XXX-plugin": {}
      },
      "vad": {
        "module": "XXX-plugin",
        "XXX-plugin": {}
      },
      "wake_word": "hey_mycroft",
      "hotwords": {
        "hey_mycroft": {
          "module": "ovos-ww-plugin-precise-lite",
          "model": "https://github.com/OpenVoiceOS/precise-lite-models/raw/master/wakewords/en/hey_mycroft.tflite"
        }
      }
    }
  },
  "network_protocol": {
    "hivemind-websocket-plugin": {
      "host": "0.0.0.0",
      "port": 5678,
      "ssl": false,
      "cert_dir": "~/.local/share/hivemind",
      "cert_name": "hivemind"
    },
    "hivemind-http-plugin": {
      "host": "0.0.0.0",
      "port": 5679,
      "ssl": false,
      "cert_dir": "~/.local/share/hivemind",
      "cert_name": "hivemind"
    }
  },
  "database": {
    "module": "hivemind-json-db-plugin",
    "hivemind-json-db-plugin": {
      "name": "clients",
      "subfolder": "hivemind-core"
    }
  }
}
```

---

## Loading the Configuration

HiveMind-Core loads the configuration using the `get_server_config()` function:

```python
from hivemind_core.config import get_server_config

config = get_server_config()
```

* Creates the configuration file if it doesn’t exist.
* Merges default values for missing keys.
* Returns a `JsonStorageXDG` object with all HiveMind settings.

---

## Notes

* Default file path: `~/.config/hivemind-core/server.json`
* HiveMind merges defaults if keys are missing.
* Binary protocol plugins allow HiveMind nodes to **stream audio for STT/TTS and handle wakeword detection**, separating heavy tasks from lightweight satellites.

