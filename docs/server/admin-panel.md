# Admin Panel

If editing JSON by hand and juggling CLI flags isn't your idea of a good evening, this
is the door you want. The Admin Panel is a web dashboard for your whole hive: one
command starts `hivemind-core` *and* opens a browser page to run it from. Pair a new
satellite by scanning a QR code, flip a client's permissions with a toggle, install a
plugin, watch the mesh light up on a live topology graph — all without touching
`server.json` or babysitting a second process. It is the easiest way to stand up and
manage a HiveMind, full stop.

!!! abstract "In a nutshell"
    - One command starts `hivemind-core` in-process and serves a web UI to administer it.
    - Manage clients, per-client ACLs, personas, plugins, and config from the browser, with QR pairing and an ACL-enforcing test chat.
    - A privileged control plane: authenticated over HTTP Basic, bound to `127.0.0.1` by default, with a forced first-run password change.

!!! tip "New here? Start with the panel"
    If the CLI flow in the [Quick Start](../quickstart.md) feels like a lot, use this
    instead. `pip install hivemind-admin-panel`, run one command, and pair satellites
    from a web page (complete with QR codes). You can always drop down to the CLI later.

## What it is

`hivemind-admin-panel` is a standalone, optional package (a FastAPI backend plus a
single-page web UI) that administers [`hivemind-core`](ovos-server.md). From the browser you
can:

- **Clients & access keys** — create, list, update, and revoke satellite credentials,
  with **QR pairing** for one-tap onboarding.
- **Per-client ACLs** — allow message types, blacklist skills/intents, toggle the
  admin / escalate / propagate flags, and apply reusable ACL templates.
- **Test Chat** — an in-browser chat that impersonates any client and talks through the
  server to the real agent, **enforcing that client's real ACLs** — so you test the actual
  permission path, not a bypass.
- **Personas & agents** — manage personas and the agent backend, with a memory-aware
  test chat and pre-activation validation.
- **Plugins** — install, upgrade, and uninstall plugins (with an active-module guard),
  and save named **plugin presets** (`{module, config}` for STT/TTS/WW/VAD/agent/network).
- **Config safety** — `server.json` is **snapshotted before every change**; diff and
  one-click **revert** from the Operations page.
- **Live monitoring** — SSE-streamed metrics, the `hivemind-core` log tail, an audit log, and an
  interactive **topology graph** (`hivemind-core` ↔ satellites with online status).
- **Security self-check**, **backup/restore**, **self-signed TLS cert generation**, and
  one-click provisioning of the **Matrix / Twitch / Mattermost / DeltaChat / HackChat**
  chat bridges.

---

## The key idea: one launcher

!!! note "The panel *is* the `hivemind-core` launcher"
    By default the panel starts `hivemind-core` **in-process** (in a daemon thread) and
    serves the admin UI on top of it. So a single command gives you a running `hivemind-core`
    **plus** a place to administer it — you do **not** run `hivemind-core` separately.

    Pass `--no-core` to serve the panel against on-disk config/database only (useful
    when a `hivemind-core` is managed elsewhere on the host, or to provision clients
    without touching a live service). In that mode the live-connection views degrade to
    placeholder data, but everything backed by the database and config still works.

---

## Install & run

```bash
pip install hivemind-admin-panel
```

Installing the panel also pulls in `hivemind-core`. Then:

```bash
hivemind-admin-panel --host 127.0.0.1 --port 8100
```

Open <http://127.0.0.1:8100>. You will be prompted for HTTP Basic credentials (see
below). The `hivemind-core` transports (WebSocket on `:5678`, etc.) are configured in
`server.json`, independently of the panel's `--host` / `--port`.

| Flag | Default | Meaning |
|---|---|---|
| `--host` | `127.0.0.1` | bind address for the panel UI |
| `--port` | `8100` | port for the panel UI |
| `--no-core` | off | serve the panel only; don't start an in-process `hivemind-core` |
| `--reload` | off | dev auto-reload (implies `--no-core`) |
| `--log-level` | `INFO` | log level for the in-process `hivemind-core` |

### Docker

A bundled Docker / Compose stack runs Redis plus the panel and exposes the WebSocket
transport (`5678`) and the panel UI (`8100`):

```bash
docker compose up --build
# open http://127.0.0.1:8100  (edit docker/server.json to set admin_pass first)
```

---

## Configuration & security

The panel reads the **same** `~/.config/hivemind-core/server.json` that `hivemind-core` uses.
Authentication is HTTP Basic (or a bearer token from `POST /auth/login`) on every
endpoint, with credentials in the `admin_user` / `admin_pass` keys (both default
`admin`). Comparison is timing-safe and new passwords are stored hashed (PBKDF2).

On your **first login with the default password, the panel forces a password change**
(a modal that blocks the whole UI until you set a new one). A dashboard **security
self-check** then flags, in red/yellow/green, whether the password is still default,
whether the panel is bound to a non-loopback address, and whether the `hivemind-core` WebSocket
has TLS — and stays red until the critical items clear.

!!! danger "Never expose the default credentials to a network"
    The panel is a **privileged control plane** — through it an authenticated caller can
    `pip install` packages into the server's interpreter, migrate or clear databases,
    mint client credentials, and restart `hivemind-core`. Treat access as equivalent to shell
    access on the host.

    - Keep the bind on `127.0.0.1` (the default) unless it sits behind a trusted,
      TLS-terminating, authenticating reverse proxy.
    - **Change `admin` / `admin` immediately** — the forced first-run change exists for
      exactly this reason; don't defeat it by pre-seeding a weak password.
    - `--host 0.0.0.0` and the Docker image bind all interfaces — only do that behind a
      proxy / firewall.

---

## See also

- [Quick Start](../quickstart.md) — the CLI flow; the panel is the easier alternative to it.
- [OVOS Skills Server](ovos-server.md) — what the in-process `hivemind-core` actually runs.
- [Security](../concepts/security.md) — access keys, certificates, and permissions.

---

## Source

Validated against the HiveMind source:

- [`hivemind_admin_panel/__main__.py`](https://github.com/JarbasHiveMind/hivemind-admin-panel/blob/HEAD/hivemind_admin_panel/__main__.py) — the launcher, the `--host` / `--port` / `--no-core` / `--reload` flags, and the in-process `HiveMindService`
- [`docs/getting-started.md`](https://github.com/JarbasHiveMind/hivemind-admin-panel/blob/HEAD/docs/getting-started.md) — install, first run, and the forced first-login password change
- [`docs/security.md`](https://github.com/JarbasHiveMind/hivemind-admin-panel/blob/HEAD/docs/security.md) — auth model, the security self-check, and the privileged endpoints
