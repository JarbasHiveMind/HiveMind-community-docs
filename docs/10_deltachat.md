# DeltaChat Bridge

[Delta Chat](https://delta.chat/en/) is a messaging app that runs over ordinary email. Any two Delta Chat users can message each other using nothing but an email address, and the app hides the email plumbing behind a normal chat interface. It encrypts messages end-to-end with [Autocrypt](https://autocrypt.org/) once both sides have exchanged keys.

The DeltaChat bridge gives a HiveMind hub its own Delta Chat identity. It logs in as a mailbox, and any message sent to that address becomes an utterance on the hub. The hub's spoken reply is sent back as a Delta Chat message in the same conversation.

```
Delta Chat  ⇄  HiveMind-deltachat-bridge  ⇄  HiveMind hub  ⇄  OVOS skills
```

## Get the bot a mailbox

The bridge needs an email account it can log in to with IMAP/SMTP. The easiest way is **chatmail**, a kind of account made specifically for Delta Chat that self-provisions with no signup form: open the Delta Chat app, choose "create a new account", and it registers one automatically on a public chatmail server (for example `nine.testrun.org`). Read the account's address and password back out of the app's settings, or provision one programmatically — see the [chatmail documentation](https://chatmail.at) for both.

You can also use any regular IMAP/SMTP mailbox, or self-host a chatmail relay with [`chatmaild`](https://github.com/chatmail/relay) for full control.

One thing this deployment hit in practice: the **first** message from a brand-new contact may go unanswered. Delta Chat treats an unknown sender as unverified until it completes an Autocrypt key exchange, so add the bot as a contact (or scan its invite QR code) and give it a moment before expecting a reply.

## Install

```bash
pip install HiveMind-deltachat-bridge
```

This installs the `hm-deltachat-bridge` command.

## Register the bridge on the hub

On the machine running `hivemind-core`:

```bash
hivemind-core add-client --name deltachat-bridge \
  --access-key "your-access-key" --password "your-password"
```

A freshly registered client is denied every message type until you whitelist it. Skipping this is the most common reason a bridge connects but never replies:

```bash
hivemind-core allow-msg recognizer_loop:utterance deltachat-bridge
hivemind-core allow-msg speak deltachat-bridge
```

If you run more than one HiveMind bridge on the same machine, give each its own identity with `hivemind-client set-identity`. Bridges that share an identity share a Noise session pin, and the hub treats reconnects from either as the same client, breaking encryption for both.

## Run it

```bash
hm-deltachat-bridge \
  --email "bot@example.com" \
  --email-password "mailbox-password"
```

HiveMind credentials (`--key/--password/--host`) are read from the identity stored by `hivemind-client set-identity`, or can be passed as flags.

## Try it

From any Delta Chat app (or plain email), message the bot's address:

```
what time is it?
```

The bridge forwards the message to the hub, waits for the `speak` reply, and answers in the same chat.

## Troubleshooting

- **Bridge connects but never replies**: the client is registered but not whitelisted. Run the two `allow-msg` commands above.
- **First message from a new contact gets no reply**: the contact needs to complete the Autocrypt key exchange first. Add the bot as a contact and wait a moment.
- **Mailbox login fails**: verify IMAP/SMTP is enabled for the account, and that the password is an app password where the provider requires one.
- **`invalid api key` at connect time**: the bridge, or its `hivemind-bus-client` dependency, is older than the hub. Upgrade it.
- **Handshake fails after the hub was reinstalled or the client's key changed**: the bridge is holding a stale Noise session pin. Clear it on the hub with `hivemind-core reset-noise-pin deltachat-bridge` and restart the bridge.

## More

The bridge's own repository has a full [setup walkthrough](https://github.com/JarbasHiveMind/HiveMind-deltachat-bridge/blob/dev/docs/setup.md) and an [accounts & chatmail guide](https://github.com/JarbasHiveMind/HiveMind-deltachat-bridge/blob/dev/docs/accounts-and-chatmail.md) covering every way to get a mailbox.

---
[← Mattermost](11_mattermost.md) · [Home](index.md) · [HackChat →](12_hackchat.md)
