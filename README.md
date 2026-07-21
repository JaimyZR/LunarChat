# LunarChat

A working Python implementation of the LunarChat design spec: a privacy-first,
end-to-end encrypted messaging platform with an untrusted relay server.

Everything here is **real, tested code** — not a mockup. The 44 tests in
`tests/` include full integration tests that spin up a relay server over a
genuine TLS 1.3 socket, connect two clients, and verify a message encrypted
by Alice's Double Ratchet is only readable after Bob decrypts it — the relay
in between only ever sees opaque bytes.

## Quickstart

```bash
pip install -r requirements.txt   # just `cryptography` — see "Dependencies" below

# Run the whole test suite (44 tests, no network access needed, ~2s)
python -m unittest discover -s tests -v

# 1. Start a relay with an auto-generated dev TLS certificate
python -m lunarchat relay run --host 127.0.0.1 --port 6580 --dev-cert
#    -> prints:  cert_pin = "sha256/AbCd..."

# 2. In two other terminals, create identities
python -m lunarchat identity create --data-dir ~/.lunarchat_alice --name Alice
python -m lunarchat identity create --data-dir ~/.lunarchat_bob   --name Bob

# 3. Exchange public identities out-of-band (this is the "verify fingerprint" step)
python -m lunarchat identity show --data-dir ~/.lunarchat_alice > alice.json
python -m lunarchat identity show --data-dir ~/.lunarchat_bob   > bob.json
python -m lunarchat contact add --data-dir ~/.lunarchat_alice --file bob.json   --trust
python -m lunarchat contact add --data-dir ~/.lunarchat_bob   --file alice.json --trust

# 4. Chat
python -m lunarchat chat --data-dir ~/.lunarchat_alice --host 127.0.0.1 --port 6580 \
    --cert-pin "sha256/AbCd..." --to <bob_user_id>
python -m lunarchat chat --data-dir ~/.lunarchat_bob   --host 127.0.0.1 --port 6580 \
    --cert-pin "sha256/AbCd..." --to <alice_user_id>
```

## What's implemented (matches spec sections 4–13)

| Spec section | Module | Status |
|---|---|---|
| 4. Identity System | `lunarchat/crypto/identity.py` | Ed25519 + X25519 keypairs, fingerprints, Argon2id+AES-GCM encrypted export/import |
| 5. Cryptographic Protocol | `lunarchat/crypto/ratchet.py`, `cipher.py`, `protocol.py` | Full Double Ratchet (DH ratchet + symmetric chains + skipped-key cache for out-of-order delivery), ChaCha20-Poly1305/AES-256-GCM, replay + timestamp protection |
| 6. Network Protocol | `lunarchat/network/protocol.py`, `models/packet.py` | Length-prefixed framing, all 12 packet types, TLS 1.3 + certificate pinning |
| 7. Relay Server | `relay/server.py`, `relay/storage.py` | Ed25519-signature auth, session registry, rate limiting (token bucket), offline queueing with 7-day retention, heartbeat monitoring |
| 8. Client Architecture | `lunarchat/core.py` | Composition root wiring identity/storage/network/crypto together |
| 9. Storage Layer | `lunarchat/storage/*` | SQLite (spec's exact schema) encrypted at rest with AES-256-GCM |
| 10. GUI | `lunarchat/gui/*` | PyQt6 widgets built to spec (see caveat below) |
| 11. File Transfer | `lunarchat/files/transfer.py` | Chunked, per-file AES-256-GCM keys, checksum verification, out-of-order chunk reassembly |
| 13. Error Handling | `lunarchat/errors.py` | Full error code taxonomy + user-facing messages |
| 14. Testing | `tests/` | 44 unit + integration tests, all passing |
| CLI client (Phase 1 roadmap) | `lunarchat/__main__.py` | Fully functional: identity, contacts, relay, interactive chat |

## Dependencies: what's real vs. stubbed, and why

This was built in a sandboxed environment with **no internet access**, so
only the `cryptography` package (already present) could be used. The spec's
Appendix A lists several packages that aren't installable here
(`PyQt6`, `aiosqlite`, `aiofiles`, `pydantic`, `structlog`, `qrcode`,
`pytest`). Rather than leave gaps, each was replaced with a documented,
equivalent stdlib approach:

- **aiosqlite → stdlib `sqlite3`** run on a dedicated single thread via
  `ThreadPoolExecutor`, wrapped in the same async interface aiosqlite would
  give you. Fully tested.
- **pydantic → stdlib `dataclasses`** (the spec's own code samples already
  used dataclass-style definitions).
- **structlog → stdlib `logging`** with a key=value formatter.
- **pytest → stdlib `unittest`**. All 44 tests use only the standard library,
  so they run with zero setup. (`pyproject.toml` still lists `pytest` etc.
  under `[project.optional-dependencies].dev` if you want to add them.)
- **qrcode → optional, guarded import.** `gui/identity.py` shows a QR code
  if `qrcode` is installed, and falls back to the text fingerprint (which
  spec 4.4 explicitly allows as an alternative to QR) if it isn't.
- **SQLCipher → AES-256-GCM page wrapper**, exactly the fallback the spec
  itself names in 9.2. See the docstring in `storage/database.py`.
- **PyQt6 → written to spec, not runnable here.** `gui/*.py` implements
  the window layout, dark theme, message bubbles, typing indicators, and
  screens from spec section 10, and imports cleanly. But without PyQt6
  installed in this sandbox, it could not actually be launched or
  screenshotted. `gui/app.py`'s `run_gui()` detects this and tells the user
  to `pip install PyQt6`, then points them at the fully-working CLI instead.
  **If you have PyQt6 available, this is the one part of the deliverable
  that hasn't been execution-tested** — everything else has.

Everything in the table above that *isn't* the GUI has been executed and
verified working in this environment, including two clients talking through
a real relay over real TLS 1.3 sockets with certificate pinning.

## Design notes / simplifications vs. the spec

- **Initial key agreement** (spec 5.2) is the simplified X25519 exchange the
  spec itself describes, not full Signal X3DH with prekey bundles — there
  are no one-time prekeys, since the spec doesn't call for them either.
- **TLS trust model** is pinning-only by default (`network/client.py`), since
  self-hosted relays generally use self-signed certs; `verify_ca=True` is
  available for deployments with a real CA-issued cert.
- **Relay's "unknown recipient" check** needed a small addition not in the
  spec's minimal `message_queue` schema: a `known_users` table so the relay
  can tell "offline but real" apart from "never existed" (spec 6.5's
  `RECIPIENT_NOT_FOUND` case). This is the one schema addition beyond spec 9.1/7.5.

## Project layout

```
lunarchat/
├── lunarchat/
│   ├── crypto/       identity, double ratchet, AEAD ciphers, packet protocol
│   ├── network/      TCP framing, TLS client, relay packet dispatch
│   ├── storage/      encrypted SQLite: contacts, conversations, messages
│   ├── files/         chunked E2E file transfer
│   ├── gui/          PyQt6 screens (spec 10) -- see caveat above
│   ├── models/       message / contact / packet dataclasses
│   ├── utils/        config (tomllib) + logging
│   ├── core.py        composition root used by the CLI (and GUI)
│   ├── errors.py      spec 13 error taxonomy
│   └── __main__.py    CLI: identity / contact / relay / chat
├── relay/            the untrusted relay server (separate deployable, spec 15.1)
├── scripts/
│   └── gen_dev_cert.py  self-signed TLS cert + pin generator
├── tests/            44 tests, stdlib unittest, no network required
├── config.example.toml
└── pyproject.toml
```

## Security notes

This is a faithful, tested implementation of the spec's design — but it
hasn't had the external security audit the spec itself calls for in its
Phase 2 roadmap (section 16) before real-world use. Treat it as a strong
reference implementation / learning project, not a production-hardened app.
