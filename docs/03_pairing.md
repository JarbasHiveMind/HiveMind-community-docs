# Pairing devices

You can register clients in a Mind via command line or via audio

## Command Line Pairing

First, you need to register the satellite devices in the HiveMind **server**

```bash
$ hivemind-core add-client
Credentials added to database!

Node ID: 2
Friendly Name: HiveMind-Node-2
Access Key: 5a9e580a2773a262cbb23fe9759881ff
Password: 9b247ca66c7cd2b6388ad49ca504279d
Encryption Key: 4185240103de0770
WARNING: Encryption Key is deprecated, only use if your client does not support password
```

And then set the identity file in the **satellite** device
```bash
$ hivemind-client set-identity --key 5a9e580a2773a262cbb23fe9759881ff --password 9b247ca66c7cd2b6388ad49ca504279d --host 0.0.0.0 --port 5678 --siteid test
identity saved: /home/miro/.config/hivemind/_identity.json
```

check the created identity file if you like
```bash
$ cat ~/.config/hivemind/_identity.json
{
    "password": "9b247ca66c7cd2b6388ad49ca504279d",
    "access_key": "5a9e580a2773a262cbb23fe9759881ff",
    "site_id": "test",
    "default_port": 5678,
    "default_master": "ws://0.0.0.0"
}
```

test that a connection is possible using the identity file
```bash
$ hivemind-client test-identity
(...)
2024-05-20 21:22:28.003 - OVOS - hivemind_bus_client.client:__init__:112 - INFO - Session ID: 34d75c93-4e65-4ea9-b5f4-87169dcfda01
(...)
== Identity successfully connected to HiveMind!
```

If the identity test passed, then your satellite is paired with the Hive!

## Audio Pairing via GGWave

Data over sound for HiveMind

hivemind-core and hivemind-voice-sat have hivemind-ggwave support

pre-requisites:

- a device with a browser, eg a phone

- a hivemind-core device with mic and speaker, eg a mark2

- a (unpaired) voice satellite device with mic and speaker, eg a raspberry pi

- all devices need to be in audible range, they each need to be able to listen to sounds emitted by each other

workflow:

- when launching hivemind-core take note of the provided code, eg `HMPSWD:ce357a6b59f6b1f9`

- copy paste the code and emit it via ggwave (see below)

- the voice satellite will decode the password, generate an access key and send it back via ggwave

- master adds a client with key + password, send an ack (containing host) via ggwave

- satellite devices get the ack then connect to received host

![img_9.png](img_9.png)

- manually exchanged string [via browser](https://jarbashivemind.github.io/hivemind-ggwave/)
  
<iframe src="https://jarbashivemind.github.io/hivemind-ggwave"></iframe>

- with a [talking button](https://github.com/ggerganov/ggwave/discussions/27)

<video src="https://user-images.githubusercontent.com/1991296/166411509-5e1b9bcb-3655-40b1-9dc3-9bec72889dcf.mp4" width="320"></video>
