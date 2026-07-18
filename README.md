# LunarChat

End-to-end encrypted communication via a server that is **only** a
dumb relay. The server passes along user IDs and public keys and
forwards encrypted packets to the right recipient — but can never read
them.

## Architecture

```
pgp_relay_chat/
├── shared/
│   ├── protocol.py     # message type constants (the "contract")
│   └── framing.py      # length-prefixed JSON over TCP
├── server/
│   ├── server.py        # the relay itself
│   ├── config.py         # reads server_config.ini
│   ├── server_config.ini
│   └── logging_setup.py  # metadata-only logging, rotating
└── client/
    ├── client.py        # CLI + network layer
    ├── crypto.py         # all PGP logic (key management, sign/encrypt/decrypt/verify, TOFU)
    ├── config.py          # reads client_config.ini
    └── client_config.ini
```

**Why this separation?** `shared/` contains only the protocol — not a
single line of encryption. `server/` doesn't import `crypto.py`
anywhere and doesn't have access to it either (the file literally
isn't in the server folder). That's deliberate: it's an architectural
guarantee, not just a promise in a comment, that the server can never
decrypt anything.

## What the server can and can't do

| Can | Can't |
|---|---|
| Let clients connect | Read messages |
| Keep track of who's online | Store/receive private keys |
| Pass along public keys | Perform encryption/decryption |
| Forward encrypted data | Retain chat history |

Everything the server knows about a client lives only in memory (the
`self.clients` dict) for as long as that client is connected. On
disconnect, that entry disappears. Nothing is written to disk except
the log files, and those contain metadata only (see
`server/logging_setup.py`).

## Cryptography

- An RSA-2048 PGP key pair per user ID (via [`pgpy`](https://pypi.org/project/PGPy/)), stored
  locally as `keys/<userid>_private.asc`, encrypted with a passphrase
  (AES-256).
- Outgoing message: **sign-then-encrypt** — first you sign it with
  your own private key, then encrypt it with the recipient's public
  key. That provides both confidentiality and authenticity (the
  recipient can be sure the message really came from you and wasn't
  altered in transit).
- Key exchange: when you `/msg` an unknown user ID, the client
  automatically requests that user's public key from the server. That
  isn't inherently a problem — public keys are by definition not
  secret — but it does open the door to a malicious server handing
  back a different key (man-in-the-middle). Hence:
- **Trust-on-first-use (TOFU)**: the first time you see a public key
  for a user ID, the fingerprint gets pinned locally in
  `keys/trusted_keys.json`. If that fingerprint changes later, the
  client warns you explicitly. For real certainty, compare the
  fingerprint shown when your key was created, via a different
  channel (phone, in person), with what your contact sees.

## Security boundaries — be honest about these

This system protects the **content** of messages against the server
and against network eavesdropping (provided you also add TLS on top
of the TCP connection yourself, see below). It does **not** protect
against:

- **Metadata**: the server always sees who's talking to whom and
  when, even though it can't see what's being said.
- **Denial of service**: the server (or anyone controlling the
  server) can make messages disappear simply by not forwarding them.
- **Physical/local access**: if someone gets access to a user's
  device while the private key is unlocked (or knows the passphrase),
  that key is compromised.
- **Transport layer**: this example uses plain TCP sockets without
  TLS. The payload itself is encrypted, but for production use you'd
  still want TLS on top (e.g. via `ssl.wrap_socket`) to also hide
  connection metadata and the fact that you're using this server from
  network observers.
- This is a learning project and has **not been professionally
  security-reviewed/audited**. Don't use it for highly sensitive
  communication without an external review.

## Installing

```bash
pip install -r requirements.txt
```

Windows and Linux work identically — everything is pure Python
standard library (`socket`, `threading`, `configparser`, `logging`)
plus `pgpy`.

## Starting

Server (on the machine acting as the relay):
```bash
cd server
python server.py
```

Client (on any device that wants to chat):
```bash
cd client
python client.py
```

The first time a user ID is used, the client generates a key pair and
asks for a passphrase; remember it well, there's no "forgot password"
option. Commands in the client:

- `/list` — who's online (and discoverable)
- `/msg <userid> <message>` — sends an encrypted message
- `/quit` — close the connection

## Configuration

`server/server_config.ini` and `client/client_config.ini` are
automatically created with default values on the first run. Adjust,
for example, `port`, `log_level`, or `key_dir` without needing to
touch the code.

## Extension roadmap

The current architecture is deliberately set up so that the features
below can be added later without breaking the protocol or the
server/client separation:

- **Contact list** — purely client-side: store a list of user IDs +
  nicknames locally (e.g. in a JSON file next to `trusted_keys.json`).
  The server doesn't need to know anything about this.
- **Online/offline status** — `LIST_RESULT` currently only gives a
  snapshot. Add a new server message type (e.g. `presence`) that gets
  pushed as soon as someone connects/disconnects, instead of only on
  request via `/list`.
- **Discovery toggle** — already partly present: `register` accepts a
  `discoverable` flag that determines whether you show up in `/list`.
  Set this in the client config and include it when registering.
- **Key verification** — the TOFU fingerprint check in `crypto.py` is
  the foundation. Could be extended with, for example, a QR code
  containing the fingerprint that you can scan in person, or a
  "verified" flag in `trusted_keys.json` that only gets set after
  manual confirmation.
- **File encryption** — reuse `sign_and_encrypt`/`decrypt_and_verify`
  in `crypto.py`, but pass in the bytes of a file instead of a
  string; add a new protocol type `file` with filename + size as
  metadata.
- **Accounts** — note: this must not make the server "less dumb."
  An account here should at most mean that registration requires a
  password hash alongside the user ID, purely to prevent user ID
  hijacking — the server stays just as blind to the content.
- **Encrypted storage** — if you later want to store messages locally
  (currently this doesn't happen), encrypt that storage with a key
  derived from the passphrase, never in plaintext on disk.
- **Versioning/auto-update** — add a `version` field to the
  `register` message; the server can (optionally) warn about an
  outdated client version without knowing anything further about the
  content.
