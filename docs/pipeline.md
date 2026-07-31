# HiveMind Pipeline Plugin

The HiveMind Pipeline Plugin lets an OpenVoiceOS device query a smarter HiveMind when an intent is uncertain. It integrates directly into the OVOS intent pipeline.

---

## Install

Install this plugin in `ovos-core`:

```bash
pip install ovos-hivemind-pipeline-plugin
```

---

## Configuration

Configure the plugin through the standard `mycroft.conf`.

> Learn more about intent pipelines and configuration in the [OVOS technical manual](https://openvoiceos.github.io/ovos-technical-manual/pipelines_overview/).

Example configuration:

```json
{
  "intents": {
    "pipeline": [
      "...",
      "ovos-hivemind-pipeline-plugin",
      "..."
    ],
    "ovos-hivemind-pipeline-plugin": {
      "name": "Hive Mind",
      "confirmation": true,
      "slave_mode": false,
      "allow_selfsigned": false
    }
  }
}
```

| Option             | Value       | Description                                                                                   |
| ------------------ | ----------- | --------------------------------------------------------------------------------------------- |
| `name`             | `Hive Mind` | Name of the HiveMind AI assistant in confirmation dialogs                                     |
| `confirmation`     | `true`      | Speaks a confirmation when a request is sent to HiveMind                                       |
| `allow_selfsigned` | `false`     | Allow self-signed SSL certificates for the HiveMind connection                                |
| `slave_mode`       | `false`     | When `true`, the device passively monitors HiveMind and can inject messages into the OVOS bus |

---

## HiveMind setup

1. Register the pipeline plugin on HiveMind Core:

```bash
hivemind-core add-client
```

Example output:

```
Node ID: 2
Friendly Name: HiveMind-Node-2
Access Key: 5a9e580a2773a262cbb23fe9759881ff
Password: 9b247ca66c7cd2b6388ad49ca504279d
Encryption Key: 4185240103de0770
WARNING: Encryption Key is deprecated
```

2. Set identity on the OVOS device:

```bash
hivemind-client set-identity \
  --key 5a9e580a2773a262cbb23fe9759881ff \
  --password 9b247ca66c7cd2b6388ad49ca504279d \
  --host 0.0.0.0 \
  --port 5678 \
  --siteid test
```

Verify identity:

```bash
cat ~/.config/hivemind/_identity.json
```

Test the connection:

```bash
hivemind-client test-identity
```

> If this step fails, the OVOS device cannot connect to HiveMind.

---

## Slave mode

In slave mode, the device can passively monitor HiveMind and emit serialized HiveMessages over the regular OVOS bus.

- From slave to master (the master might reject this): emit `"hive.send.upstream"` with `message.data`, `{"msg_type": "bus", "payload": message.serialize()}`.
- From master to slave: emit `"hive.send.downstream"` with `message.data`, `{"msg_type": "bus", "payload": message.serialize()}`.

> This is what enables [nested hives](15_nested.md). OpenVoiceOS can act as both a master (by running `hivemind-core`) and a slave (by running this plugin).

---
[← Mic Satellite](07_micsat.md) · [Home](index.md) · [Matrix →](09_matrix.md)
