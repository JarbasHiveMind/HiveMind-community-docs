# DeltaChat Bridge

[DeltaChat](https://delta.chat/en/) bridge 

![img.png](https://github.com/JarbasHiveMind/HiveMind-deltachat-bridge/raw/master/img.png)

## Install

```bash
$ pip install HiveMind-deltachat-bridge
```
## Usage

If host is not provided auto discovery will be used

```bash
$ HiveMind-deltachat-bridge --help

usage: __main__.py [-h] --access_key ACCESS_KEY --email EMAIL --password
                   PASSWORD [--crypto_key CRYPTO_KEY] [--name NAME]
                   [--host HOST] [--port PORT]

optional arguments:
  -h, --help            show this help message and exit
  --access_key ACCESS_KEY
                        hivemind access key
  --email EMAIL         deltachat email
  --password PASSWORD   deltachat password
  --crypto_key CRYPTO_KEY
                        payload encryption key
  --name NAME           human readable device name
  --host HOST           HiveMind host
  --port PORT           HiveMind port number
```

NOTE: you only need to provide the password the first time you connect