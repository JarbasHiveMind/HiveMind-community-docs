# Quick Start Guide

This guide sets up a working HiveMind from a clean machine: a hub that runs
OpenVoiceOS, one voice satellite, and one spoken exchange between them.
Every command in this guide was run on a clean install of the versions
listed below.

HiveMind extends your OpenVoiceOS (OVOS) setup across multiple devices, even
low-resource hardware. It connects lightweight devices as satellites to a
central OVOS hub, with centralized control and fine-grained permissions.

![imagem](https://github.com/JarbasHiveMind/HiveMind-community-docs/assets/33701864/fb241c4d-ca84-4b47-b917-b398b16f93bd)

## Installation

Install the latest packages. `hivemind-core` runs on the device that runs
OVOS:
`HiveMind-voice-sat` runs on the satellite device:

```bash
# hub device
uv pip install --prerelease=allow "hivemind-core" "ovos-core" "ovos-messagebus"

# satellite device
uv pip install --prerelease=allow "HiveMind-voice-sat"
```

## Add a satellite device to the hub

`add-client` writes a client record into the hub's client database. Run:

```bash
hivemind-core add-client --name "satellite_1" --access-key "mykey123" --password "mypass"
```

On success the command prints the credentials:

```
Credentials added to database!

Node ID: 18
Admin Privileges: False
Friendly Name: docsqa
Access Key: docsqaA1
Password: Str0ng!Passphrase-9x
```

Two rules apply. First, a too-guessable password is refused. The command
asks for about 40 bits of resistance, so use a long passphrase. Second, the
new client starts with an empty message whitelist: the output says the
client is denied on every message until you grant access. The next
subsection fixes that.

## Choose the allowed message types

HiveMind Core examines every message a satellite sends against that
satellite's allowed-types list. A type not on the list is dropped at the hub
and logged as a `policy denied` INFO line. Nothing on the satellite tells
you. You learn from the hub log or from silence.

A client record stores one list of type names. The list carries the type
exactly as the client sends it. Two satellites on different stack versions
send different spellings for the same meaning, so a satellite running a
voice stack older than the current OVOS message-names migration carries
both spellings of every pair. The pairs are:

| Meaning | Legacy spelling | OVOS spelling |
|---|---|---|
| Speech to interpret | `recognizer_loop:utterance` | `ovos.utterance.handle` |
| Mic recording started | `recognizer_loop:record_begin` | `ovos.listener.record.started` |
| Mic recording ended | `recognizer_loop:record_end` | `ovos.listener.record.ended` |
| Set volume | `mycroft.volume.set` | `ovos.volume.set` |
| Read volume | `mycroft.volume.get` | `ovos.volume.get` |
| Volume up a step | `mycroft.volume.increase` | `ovos.volume.increase` |
| Volume down a step | `mycroft.volume.decrease` | `ovos.volume.decrease` |
| Play a local sound | `mycroft.audio.play_sound` | `ovos.audio.play_sound` |
| Mic status check | `mycroft.mic.get_status` | `ovos.mic.get_status` |
| Mic state report | `mycroft.mic.update_listening` | `ovos.mic.update_listening` |

A satellite needs, at minimum, the utterance type and both spellings of it
(`recognizer_loop:utterance` and `ovos.utterance.handle`) to send speech to
the hub at all. The voice satellite also sends the record begin/end pairs,
the volume set/get pair and the audio play pair during a normal session.
Each denied type shows as a log line at the hub. Grant the types the
satellite you run actually sends.

Allow a type with `allow-msg`, giving type then node id:

```bash
hivemind-core allow-msg recognizer_loop:utterance <node_id>
```

Remove with `blacklist-msg`:

```bash
hivemind-core blacklist-msg recognizer_loop:utterance <node_id>
```

A few other message families ride separate allowlists: broadcast,
escalate, propagate, and per-skill access. See
[Permissions](16_permissions.md) for those.

## Run the hub

Start the server. It reads its listen address and port from
`~/.config/hivemind-core/server.json`. Port 5678 is the default for
the websocket protocol:

```bash
hivemind-core listen
```

The server listens for incoming satellite connections. Print the live
configuration with `hivemind-core print-config`.

> `hivemind-core` must run on the same device as OVOS.

## Connect the satellite

Store the credentials on the satellite device:

```bash
hivemind-client set-identity --key mykey123 --password mypass \
    --host <hub-ip> --port 5678 --siteid satellite_1
```

`set-identity` stores the identity at `~/.config/hivemind/_identity.json`
and prepends `ws://` to the host itself. See [Pairing](03_pairing.md) for
the full flow.

Then connect:

```bash
hivemind-voice-sat
```

The satellite connects, plays the connect sound, and starts listening
locally for the wake word. Speak to it. The round trip reads, at the hub:

```
Forwarding message 'recognizer_loop:utterance' to agent bus from client: ...
```

and, if a skill answers, the satellite speaks the reply. Nothing at the
satellite prints on a denial. The hub log is the place to look.

## HiveMind Core commands overview

List all registered clients with credentials:

```bash
hivemind-core list-clients
```

For detailed help on a command, use `--help` (for example, `hivemind-core add-client --help`).

---
[Home](index.md) · [Terminology →](02_terminology.md)
