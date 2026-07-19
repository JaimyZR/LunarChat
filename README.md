# LunarChat

End-to-end encrypted communication through a server that is **only**
a dumb relay. The server hands out userids and public keys and
forwards encrypted packets to the right recipient — but can never
read them.

## Architecture

```
pgp_relay_chat/
├── shared/
│   ├── protocol.py     # message type constants (the "contract")
│   └── framing.py      # length-prefixed JSON over TCP
├── server/
│   ├── server.py         # the relay itself
│   ├── config.py          # reads server_config.ini
│   ├── server_config.ini
│   └── logging_setup.py   # metadata-only logging, rotating
└── client/
    ├── client.py        # CLI + networking layer
    ├── crypto.py         # all PGP logic (key management, sign/encrypt/decrypt/verify, TOFU)
    ├── config.py          # reads client_config.ini
    └── client_config.ini
```

**Why this separation?** `shared/` contains only the protocol — not a
single line of encryption. `server/` never imports `crypto.py` and
has no access to it (that file simply doesn't exist in the server
directory). That's intentional: it's an architectural guarantee, not
just a promise in a comment, that the server can never decrypt
anything.

## What the server may / may not do

| Allowed | Not allowed |
|---|---|
| Let clients connect | Read messages |
| Track who is online | Store/receive private keys |
| Hand out public keys | Perform encryption/decryption |
| Forward encrypted data | Retain chat history |

Everything the server knows about a client lives only in memory
(the `self.clients` dict) for as long as that client is connected.
On disconnect, that entry disappears. Nothing is written to disk
except the log files, and those contain metadata only (see
`server/logging_setup.py`).

## Cryptography

- RSA-2048 PGP keypair per userid (via [`pgpy`](https://pypi.org/project/PGPy/)), stored locally as
  `keys/<userid>_private.asc`, protected with a passphrase (AES-256).
- Outgoing message: **sign-then-encrypt** — first you sign it with
  your own private key, then encrypt it with the recipient's public
  key. This gives both confidentiality and authenticity (the
  recipient knows for certain the message came from you and wasn't
  tampered with in transit).
- Key exchange: when you `/msg` an unknown userid, the client
  automatically requests their public key from the server. That's not
  a problem in itself — public keys are not secret by definition —
  but it does open the door to a malicious server returning a
  different key (man-in-the-middle). Hence:
- **Trust-on-first-use (TOFU)**: the first time you see a public key
  for a userid, its fingerprint is pinned locally in
  `keys/trusted_keys.json`. If that fingerprint ever changes, the
  client warns you explicitly. For real assurance, compare the
  fingerprint shown when your key was created (through another
  channel — phone, in person) with what your contact sees.

## Security boundaries — be honest about these

This system protects the **content** of messages from the server and
from eavesdropping on the network (provided you also add TLS on top
of the TCP connection, see below). It does **not** protect against:

- **Metadata**: the server always sees who is talking to whom and
  when, even though it can't see what's being said.
- **Denial of service**: the server (or whoever controls it) can make
  messages disappear simply by not forwarding them.
- **Physical/local access**: if someone gains access to a user's
  device while their private key is unlocked (or knows the
  passphrase), that key is compromised.
- **Transport layer**: this example uses plain TCP sockets without
  TLS. The payload itself is encrypted, but for production use you'd
  still want TLS on top (e.g. via `ssl.wrap_socket`) to also hide
  connection metadata and the fact that you're using this server from
  network observers.
- This is a learning project and has **not been professionally
  security-reviewed or audited**. Don't use it for highly sensitive
  communication without an external review.

## Installing

```bash
pip install -r requirements.txt
```

Windows and Linux work identically — everything is pure Python
stdlib (`socket`, `threading`, `configparser`, `logging`) plus
`pgpy`.

## Running

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

On first use of a userid, the client generates a keypair and asks
for a passphrase; remember it well, there is no "forgot password"
option. Commands inside the client:

- `/list` — who is online (and discoverable)
- `/msg <userid> <message>` — sends an encrypted message
- `/quit` — close the connection

## Configuration

`server/server_config.ini` and `client/client_config.ini` are created
automatically with default values on first run. Adjust things like
`port`, `log_level`, or `key_dir` without touching any code.

## Extension roadmap

The current architecture is deliberately set up so the following
features can be added later without breaking the protocol or the
server/client separation:

- **Contact list** — purely client-side: store a list of
  userids + nicknames locally (e.g. a JSON file alongside
  `trusted_keys.json`). The server doesn't need to know anything
  about this.
- **Online/offline status** — `LIST_RESULT` currently only gives a
  snapshot. Add a new server message type (e.g. `presence`) that gets
  pushed as soon as someone connects/disconnects, instead of only on
  request via `/list`.
- **Discovery toggle** — already partly present: `register` accepts a
  `discoverable` flag that determines whether you show up in
  `/list`. Wire this up in the client config and send it along at
  registration time.
- **Key verification** — the TOFU fingerprint check in `crypto.py` is
  the foundation. Extend it with, for example, a QR code containing
  the fingerprint that can be scanned in person, or a "verified" flag
  in `trusted_keys.json` that only gets set after manual confirmation.
- **File encryption** — reuse `sign_and_encrypt`/`decrypt_and_verify`
  in `crypto.py`, but pass in the bytes of a file instead of a
  string; add a new protocol type `file` with filename + size as
  metadata.
- **Accounts** — careful: this must not make the server "less dumb".
  An account here would at most mean registration requires a password
  hash alongside the userid, purely to prevent userid squatting/
  hijacking — the server stays just as blind to the content.
- **Encrypted storage** — if you later want to store messages
  locally (currently nothing is stored), encrypt that storage with a
  key derived from the passphrase, never in plaintext on disk.
- **Versions/auto-update** — add a `version` field to the `register`
  message; the server can (optionally) warn about an outdated client
  version without learning anything more about the content.
