# HiveMind Pipeline Plugin

The **HiveMind Pipeline Plugin** allows an OpenVoiceOS device to query a smarter HiveMind when an intent is uncertain. It integrates directly into the OVOS **intent pipeline**.

---

## Install

This plugin should be installed in `ovos-core`

```bash
pip install ovos-hivemind-pipeline-plugin
```

---

## Configuration

The plugin is configured via the standard `mycroft.conf`:

> 💡 Learn more about intent pipelines and configuration in the [OVOS technical manual](https://openvoiceos.github.io/ovos-technical-manual/pipelines_overview/).

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
| `confirmation`     | `true`      | Spoken confirmation is triggered when a request is sent to HiveMind                           |
| `allow_selfsigned` | `false`     | Allow self-signed SSL certificates for the HiveMind connection                                |
| `slave_mode`       | `false`     | When `true`, the device passively monitors HiveMind and can inject messages into the OVOS bus |

---

## HiveMind Setup

1. **Register the pipeline plugin on HiveMind Core:**

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

2. **Set identity on the OVOS device:**

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

Test connection:

```bash
hivemind-client test-identity
```

> ⚠️ If this step fails, the OVOS device will not be able to connect to HiveMind.

---

## Slave Mode

When running in **slave mode**, the device can passively monitor HiveMind and **emit serialized HiveMessages** via the regular OVOS bus.

* **From slave → master:**

```json
"msg_type": "bus", "payload": message.serialize()
```

* **From master → slave:**

```json
"msg_type": "bus", "payload": message.serialize()
```

This feature enables **nested hives**, where a device can act as both **master** (running `hivemind-core`) and **slave** (running this plugin).

> See the [HiveMind protocol documentation](https://jarbashivemind.github.io/HiveMind-community-docs/04_protocol) for valid message payloads and more details.
