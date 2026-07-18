# Installation Instructions

This guide walks through installing both the server side and the
client side, for both Windows and Linux. Follow the section that
applies to you.

---

## 0. Requirements (server AND client)

- **Python 3.9 or higher**
- **pip** (comes bundled with Python by default)

Check whether Python is already installed:

**Windows** (PowerShell or cmd):
```
python --version
```
No version shown, or a version lower than 3.9? Download the latest
version from https://www.python.org/downloads/ and check
**"Add python.exe to PATH"** during installation — that saves a lot
of hassle later on.

**Linux**:
```bash
python3 --version
```
Usually already present. If not, install it via your package manager,
e.g. on Debian/Ubuntu:
```bash
sudo apt update && sudo apt install python3 python3-pip
```

---

## 1. Installing the server side

The server runs on the machine that acts as the relay point — this
can be a VPS, a home server, or just your own laptop, as long as it's
reachable by the clients.

### 1.1 Extracting the files

Extract `pgp_relay_chat.zip` on the server machine, for example to
`C:\pgp_relay_chat` (Windows) or `~/pgp_relay_chat` (Linux).

### 1.2 Installing dependencies

Open a terminal in the extracted folder and install the required
package:

**Windows**:
```
cd C:\pgp_relay_chat
pip install -r requirements.txt
```

**Linux**:
```bash
cd ~/pgp_relay_chat
pip3 install -r requirements.txt
```

> The server itself only needs the Python standard library
> (`socket`, `threading`, `configparser`, `logging`). `requirements.txt`
> includes `pgpy`, which strictly speaking only the client needs — but
> installing it on the server machine can't hurt, in case you later
> want to run the server and client on the same machine.

### 1.3 Opening the port

By default, the server listens on port **5050** (configurable in
`server/server_config.ini`). If there's a firewall or router between
the clients and the server, that port needs to be open.

**Windows Firewall** (PowerShell as Administrator):
```powershell
New-NetFirewallRule -DisplayName "PGP Relay Server" -Direction Inbound -Protocol TCP -LocalPort 5050 -Action Allow
```

**Linux (ufw)**:
```bash
sudo ufw allow 5050/tcp
```

**Linux (firewalld)**:
```bash
sudo firewall-cmd --add-port=5050/tcp --permanent
sudo firewall-cmd --reload
```

If the server runs behind a router (e.g. a home network) and also
needs to be reachable from outside your network, you'll also need to
set up port forwarding there for TCP 5050 to the internal IP of the
server machine.

### 1.4 (Optional) adjusting settings

Open `server/server_config.ini` in a text editor to change, for
example, the port or the log level:

```ini
[server]
host = 0.0.0.0
port = 5050

[logging]
log_file = server.log
log_level = INFO
max_bytes = 1000000
backup_count = 3
```

Leave `host = 0.0.0.0` as is unless you have specific reasons to bind
the server to a single network interface.

### 1.5 Starting the server

**Windows**:
```
cd C:\pgp_relay_chat\server
python server.py
```

**Linux**:
```bash
cd ~/pgp_relay_chat/server
python3 server.py
```

On success you'll see:
```
2026-07-18 20:32:48 [INFO] Relay server listening on 0.0.0.0:5050
```

The server keeps running until you press `Ctrl+C`. If you want it to
run in the background (e.g. on a Linux VPS), you can use for example:
```bash
nohup python3 server.py > /dev/null 2>&1 &
```
or set it up as a systemd service for a more robust setup (let me
know if you'd like help with that).

### 1.6 Checking that the server is reachable

From another machine on the same network:

**Windows**:
```
Test-NetConnection -ComputerName <SERVER_IP> -Port 5050
```

**Linux**:
```bash
nc -zv <SERVER_IP> 5050
```

If you see `succeeded` / `open`, the server is reachable and you can
move on to the client side.

---

## 2. Installing the client side

Repeat this on every device that wants to chat.

### 2.1 Extracting the files

Extract the same `pgp_relay_chat.zip`, for example to
`C:\pgp_relay_chat` (Windows) or `~/pgp_relay_chat` (Linux). Every
client needs the **entire folder**, not just `client/`, because
`client/client.py` depends on the `shared/` folder for the protocol.

### 2.2 Installing dependencies

**Windows**:
```
cd C:\pgp_relay_chat
pip install -r requirements.txt
```

**Linux**:
```bash
cd ~/pgp_relay_chat
pip3 install -r requirements.txt
```

This installs `pgpy` and its dependency `cryptography`. Ready-made
wheels exist for both, for Windows, macOS, and most Linux
distributions, so normally no compiler is needed. If you do get a
build error, see the troubleshooting section at the bottom.

### 2.3 (Optional) fixing the server IP

By default, the client asks for the server's IP address on startup.
To skip that, set it permanently in `client/client_config.ini`:

```ini
[connection]
server_host = 192.168.1.50
server_port = 5050
```

### 2.4 Starting the client

**Windows**:
```
cd C:\pgp_relay_chat\client
python client.py
```

**Linux**:
```bash
cd ~/pgp_relay_chat/client
python3 client.py
```

### 2.5 First startup — creating an identity

On the first run, the client will ask for:

1. **Server IP** (if you haven't already set it in the config)
2. **User ID** — choose something unique; this is how other clients
   will find you
3. **Passphrase** — protects your private key on this machine. There
   is no "forgot password" option, so keep it somewhere safe (e.g. in
   a password manager)

You'll then see your fingerprint appear:
```
[FINGERPRINT] Your fingerprint: XXXX XXXX XXXX XXXX XXXX  XXXX XXXX XXXX XXXX XXXX
```
Share this fingerprint through a different channel (e.g. a phone call
or in person) with people you want to talk to, so they can later
verify that the key they receive from the server is really yours.

Your private key is stored locally in
`client/keys/<userid>_private.asc` — it never leaves your device.

### 2.6 Chatting

After starting up:
```
Commands: /list  |  /msg <userid> <message>  |  /quit
```

- `/list` — shows who's currently online
- `/msg alice hello!` — sends an encrypted message to user ID `alice`
- `/quit` — closes the connection cleanly

---

## 3. Quick end-to-end test

To check that everything works:

1. Start the server (§1.5)
2. Start client 1, choose user ID `alice`
3. Start client 2 (on a different device, or just a second terminal
   window on the same machine), choose user ID `bob`
4. Type at `alice`: `/msg bob hello bob!`
5. `bob` should see: `[VERIFIED] alice: hello bob!`

Do you see `[NEW KEY]` instead of `[VERIFIED]` the first time? That's
normal — that's the moment the other person's fingerprint gets pinned
for the first time (trust-on-first-use). From the second message
onward you'll see `[VERIFIED]`.

---

## 4. Troubleshooting

**`ModuleNotFoundError: No module named 'pgpy'`**
You haven't run `pip install -r requirements.txt` (or ran it in the
wrong Python environment). Check with `python -m pip list` whether
`pgpy` is in the list.

**`[ERROR] Server is offline` on the client**
- Is the IP address correct?
- Is the server actually running (§1.5)?
- Is the firewall/port open (§1.3)?

**Build errors when installing `cryptography` (rare, mostly on older
Windows versions or unusual Linux distros)**
Install a recent version of pip first and try again:
```
python -m pip install --upgrade pip
pip install -r requirements.txt
```
If it keeps failing on Windows, install the "Microsoft C++ Build
Tools" (https://visualstudio.microsoft.com/visual-cpp-build-tools/)
and try again.

**Lost passphrase**
There is no recovery mechanism — that's inherent to how private key
encryption is supposed to work. You'll need to create a new identity
(a new user ID, or delete the existing
`<userid>_private.asc` file to force a new key pair under the same
user ID).

**`[WARNING] The public key of 'x' has CHANGED since last time!`**
This is a TOFU warning (see README, Cryptography section). It could
be harmless (the other person regenerated their key), but it could
also indicate a man-in-the-middle attack. Verify the new fingerprint
through a different channel before proceeding.
