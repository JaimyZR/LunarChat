# LunarChat

Privacy-first, end-to-end encrypted messaging over an untrusted relay.

The relay routes sealed packets and nothing else. It never holds a private key,
never sees plaintext, and cannot read a single message even if it is fully
compromised.

**Version 1.0.0** · Windows and Linux · Python 3.12+

---

## What it does

- End-to-end encryption with a double ratchet (forward and future secrecy)
- Identities generated on the device: no phone number, e-mail address or account
- Out-of-band fingerprint verification with a warning banner for unverified contacts
- Encrypted local database; keys never leave the machine
- Offline delivery: messages wait encrypted on the relay for up to seven days
- Chunked file transfer with per-file keys and checksum verification
- Desktop interface (PySide6) and a terminal client

## Install

```bash
python -m venv .venv && . .venv/bin/activate
pip install -e ".[gui]"
```

Windows uses `.venv\Scripts\activate` instead. See [INSTALL.md](INSTALL.md) for
packaged builds and per-platform notes.

## Run a relay

The relay needs a TLS certificate. For a local test, generate a self-signed one
and note the pin it prints:

```bash
python scripts/gen_dev_cert.py --out certs --host localhost --host 127.0.0.1
```

```bash
python -m relay --bind 127.0.0.1 --port 6580 --cert certs/relay-cert.pem --key certs/relay-key.pem
```

For a real deployment use Docker; see [Deployment](#deployment).

## Run the client

```bash
python -m lunarchat
```

First launch asks for a display name, how to protect the local database (system
keyring or a passphrase), and the relay address plus its certificate pin. All of
it can be changed later under Settings.

Terminal client:

```bash
python -m lunarchat cli
```

## Adding a contact

1. Open **My identity** and share your contact code (`lunarchat:…`) or QR code.
2. Your contact pastes it under **Add contact**.
3. Compare fingerprints over a channel you already trust — a call, or in person.
4. Press **Verify**. Until you do, the conversation shows a warning banner.

A contact code contains a public key only. Nothing about it is secret, but a
substituted code is exactly what fingerprint verification is there to catch.

## Configuration

`config.toml` lives in `~/.config/lunarchat/` on Linux and
`%APPDATA%\LunarChat\` on Windows. See [config.example.toml](config.example.toml)
for every option. No address, key or credential is compiled into the source.

## Deployment

```bash
docker compose up -d
```

`docker-compose.yml` mounts `./certs` for the certificate and a named volume for
the encrypted queue database. Configuration is passed through
`LUNARCHAT_RELAY_*` environment variables.

Sizing: 2 vCPU and 2 GB RAM handle roughly 5000 concurrent sessions.

## Development

```bash
make install     # dependencies including dev extras
make test        # full test suite
make lint        # ruff
make relay       # local relay on loopback
make run         # client
```

## Layout

```
lunarchat/        client: crypto, network, storage, files, gui
relay/            relay server: registry, routing, queue, rate limits
tests/            unit, integration, security and interface tests
scripts/          certificate generation and other utilities
packaging/        installer and package build files
```

## Documentation

- [PROTOCOL.md](PROTOCOL.md) — wire format, handshake, envelope layout
- [SECURITY.md](SECURITY.md) — threat model, guarantees, known limitations
- [INSTALL.md](INSTALL.md) — installation and packaging
- [CHANGELOG.md](CHANGELOG.md) — release history

## Licence

MIT. See [LICENSE](LICENSE).
