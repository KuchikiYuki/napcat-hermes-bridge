
# WSL Messaging Bridge Skill

Connect a messaging platform (QQ via NapCat/OneBot, Telegram, Discord, etc.)
to a Hermes Agent instance running in WSL, using a lightweight HTTP bridge
script. The Hermes agent runs as a persistent background session via the API
Server gateway adapter; the bridge forwards inbound messages and relays
replies.

## When to Use

- User wants to send commands to Hermes from a phone messaging app (QQ, etc.).
- The messaging platform has no native Hermes gateway adapter (e.g. QQ via
  NapCat/OneBot v11 — Hermes has a `qqbot` adapter for the official Tencent
  API, but that's a different protocol).
- You need a headless Electron app (NTQQ) running in WSL with xvfb.
- User wants a one-click startup script that chains all services together.

## When NOT to Use

- **Do not use AstrBot as an intermediary.** AstrBot is a standalone LLM bot
  framework with its own model configuration — it is NOT a thin relay. Using
  it as a middle layer between NapCat and Hermes adds complexity without value:
  AstrBot would need its own LLM provider configured, creating a second agent
  that shadows the Hermes agent. The correct architecture is NapCat → Bridge →
  Hermes API Server, with no AstrBot in the loop.
- **Do not spawn a new Hermes session per message within a single chain run.**
  The user's requirement is a persistent mapping: one NapCat process ↔ one
  Hermes agent session that lives for the duration of a chain run. The bridge
  achieves this by calling `/api/sessions/{SID}/chat` with a persistent session
  ID stored in a file. A stateless API call (no session ID) would create a
  throwaway session per message, losing context and defeating the purpose.
- **Do create a FRESH session on each chain startup.** Reusing a hardcoded
  session ID across chain restarts risks cross-contamination with other agent
  processes (e.g. the TUI session) that may have claimed the same ID. Each
  `qq-start.sh` invocation should generate a new session ID using the Hermes
  TUI format (`YYYYMMDD_HHMMSS_6hex`, e.g. `20260629_153354_2d6137`), persist
  it to `current_session.txt`, and use it for all messages until the next chain
  restart or a `/new` command. The user can switch back to an old session with
  `/switch <id>` if desired. Using the TUI format (not a custom `qq_` prefix)
  ensures session IDs are visually consistent across TUI and bridge, and the
  short-hash display in the TUI sidebar matches what `/list` shows.
- **Do not hardcode session IDs.** The original v2 bridge hardcoded
  `api_1782539173_f78ea9ee` and skipped `ensure_session()` on startup. When the
  Hermes Gateway restarted or the session expired, the ID became stale — the
  `/chat` endpoint would silently create a new session with a different ID,
  causing the bridge to lose all context while still sending to the old ID.
  Always persist the session ID to a file and call `ensure_session()` on
  startup, or better yet create a fresh session each time.

## Prerequisites

- WSL2 with systemd enabled (`/etc/wsl.conf` → `[boot] systemd=true`).
- `sudo` available (for apt installs: xvfb, libnss3, etc.).
- A Windows-side HTTP proxy reachable from WSL (typically the WSL gateway IP,
  e.g. `<WSL_GATEWAY_IP>:<PROXY_PORT>`). See `references/wsl-proxy-setup.md`.
- Hermes installed and configured (`hermes config set` works).
- `uv` installed (`pip install uv` in a venv, or system-wide).

## Quick Reference

| Component | Port | Purpose |
|---|---|---|
| Hermes API Server | 8642 | OpenAI-compatible endpoint, runs the agent |
| Bridge script | 8080 | HTTP listener for platform events, relays to Hermes |
| NapCat WebUI | 6099 | QQ login / management panel |
| NapCat HTTP API | 3000 | OneBot v11 send_msg endpoint |

## Architecture

```
Phone QQ ──→ NTQQ (headless, xvfb) ──→ NapCat ──POST──→ Bridge (:8080)
                                                          │
Phone QQ ←── NapCat HTTP API (:3000) ←─── Bridge ←─── Hermes API (:8642)
```

- **NapCat** hooks into NTQQ's Electron entry point (`loadNapCat.js`) and
  exposes an OneBot v11 HTTP API + posts events to a webhook URL.
- **Bridge** is a small `aiohttp` server that receives OneBot events, calls
  Hermes API Server's `/api/sessions/{SID}/chat` endpoint with a persistent
  session ID (stored in `current_session.txt`), and sends the reply back via
  NapCat's HTTP API. A fresh session is created on each chain startup; the
  session ID persists for the lifetime of that chain run. QQ commands (`/new`,
  `/switch`, `/session`, `/list`, `/help`) allow the user to manage sessions
  from their phone.
- **Hermes API Server** is a gateway platform adapter (`platforms.api_server`)
  that runs the agent loop. All QQ messages within a chain run share a single
  session ID so the agent keeps full context across messages.

## Procedure

### 1. Install system dependencies

NTQQ is an Electron app and needs xvfb + several GUI libraries:

```bash
sudo apt-get -o Acquire::http::Proxy="" -o Acquire::https::Proxy="" install -y \
    xvfb unzip libnss3 libnotify4 libsecret-1-0 libgbm1 \
    libasound2t64 fonts-wqy-zenhei gnutls-bin libglib2.0-dev \
    libdbus-1-3 libgtk-3-0 libxss1 libxtst6 libatspi2.0-0 \
    libx11-xcb1 ffmpeg curl jq
```

> **Pitfall:** `libasound2` is a virtual package on Ubuntu 26.04+; use
> `libasound2t64` instead.

> **Pitfall:** If a stale proxy is configured system-wide (e.g.
> `198.18.0.1:7891` from a previous VPN), apt will fail to connect. Bypass
> with `-o Acquire::http::Proxy="" -o Acquire::https::Proxy=""`.

### 2. Download and extract NTQQ

NapCat requires a matching NTQQ version. Check the NapCat release notes for
the recommended QQ version, then download the Linux `.deb`:

```bash
mkdir -p ~/napcat-astrbot && cd ~/napcat-astrbot
# Set proxy for the download
export HTTP_PROXY="http://<WSL_GATEWAY>:<PORT>"
export HTTPS_PROXY="$HTTP_PROXY"

curl -L -o linuxqq.deb "https://dldir1v6.qq.com/qqfile/qq/QQNT/<hash>/linuxqq_<ver>_amd64.deb"
# Extract WITHOUT sudo — dpkg-deb -x works in user space
dpkg-deb -x linuxqq.deb ntqq/
```

### 3. Download NapCat and inject into NTQQ

```bash
curl -L -o NapCat.Shell.zip "https://github.com/NapNeko/NapCatQQ/releases/download/v<ver>/NapCat.Shell.zip"
unzip -q NapCat.Shell.zip -d napcat/

# Inject: point NTQQ's entry to NapCat
QQ_APP="ntqq/opt/QQ/resources/app"
cat > "$QQ_APP/loadNapCat.js" << EOF
(async () => {await import('file://$HOME/napcat-astrbot/napcat/napcat.mjs');})();
EOF

# Update NTQQ package.json main field
python3 -c "
import json; p='$QQ_APP/package.json'; f=open(p); d=json.load(f); f.close()
d['main']='./loadNapCat.js'; open(p,'w').write(json.dumps(d,indent=2))
"
```

> **Critical:** Use an **absolute path** in `loadNapCat.js`. The Docker image
> uses `file:///app/napcat/napcat.mjs`. Relative paths like `./napcat/napcat.mjs`
> fail because NTQQ resolves them relative to the QQ resources directory, not
> where NapCat's `node_modules` live.

> **Pitfall:** NapCat's `napcat.mjs` imports sibling files (`conout-*.js`)
> and `node_modules/` from its own directory. Keep all NapCat files together
> in `~/napcat-astrbot/napcat/` — do NOT split them into the NTQQ app dir.

> **Pitfall:** NapCat creates account-specific config overrides after first
> login. See `references/napcat-config.md` for the full file listing and
> how to update them.

### 4. Configure NapCat OneBot v11

Write `napcat/config/onebot11.json` with HTTP Server (for sending) + HTTP
Client (for receiving events):

```json
{
  "network": {
    "httpServers": [{
      "enable": true, "name": "api",
      "host": "127.0.0.1", "port": 3000,
      "messagePostFormat": "array", "token": "",
      "debug": false, "enableForcePushEvent": true
    }],
    "httpClients": [{
      "enable": true, "name": "bridge",
      "url": "http://127.0.0.1:8080/onebot",
      "messagePostFormat": "array", "reportSelfMessage": false, "token": ""
    }],
    "websocketServers": [], "websocketClients": [], "plugins": []
  },
  "musicSignUrl": "", "enableLocalFile2Url": false, "parseMultMsg": false
}
```

### 5. Enable Hermes API Server

```bash
hermes config set platforms.api_server.enabled true
hermes config set platforms.api_server.extra.key "<your-api-key>"
hermes config set platforms.api_server.extra.host "127.0.0.1"
hermes config set platforms.api_server.extra.port 8642
```

> If no key is set, auth is skipped (OK for localhost-only). The bridge uses
> `X-Hermes-Session-Key` header to maintain a single persistent agent session
> across all messages.

> See `references/hermes-api-session-endpoints.md` for the full API reference
> (create/get/list/chat endpoints, response formats, field semantics). These
> formats were discovered by probing with `curl` — they are not in the Hermes
> docs and are needed for bridge-level session management.

### 6. Write the bridge script

See `templates/bridge.py` for a ready-to-use aiohttp bridge (v3). Key
design points:

- **Fresh session on startup**: each chain startup creates a new session
  (`YYYYMMDD_HHMMSS_6hex` format matching Hermes TUI, e.g.
  `20260629_153354_2d6137`), persisted to `current_session.txt`. This
  prevents cross-contamination with other agent processes (TUI, etc.)
  that might claim the same session ID. Using the TUI format ensures
  consistency between what the user sees in the TUI sidebar and what
  `/list` returns from the bridge.
- **Session chat endpoint**: uses `/api/sessions/{SID}/chat` (not
  `/v1/chat/completions` with `X-Hermes-Session-Key`) — the sessions API
  endpoint provides explicit session lifecycle control.
- **404 auto-recovery**: if the session disappears mid-conversation (Gateway
  restart, session expiry), the bridge recreates the session with
  `ensure_session()` and retries the chat call once.
- **QQ bridge commands**: messages starting with `/` are intercepted before
  reaching Hermes:
  - `/new` — create a new session (clears context)
  - `/switch <id>` — switch to an existing session
  - `/session` — show current session ID
  - `/list` — list all Hermes sessions
  - `/help` — show command help
- **User whitelist**: filter messages by QQ ID so only the owner can command
  the agent.
- **Async fire-and-forget**: respond to NapCat's POST immediately (200 OK),
  process the Hermes call in `asyncio.create_task()` so NapCat doesn't
  timeout. An instant ACK ("⚡ 已收到，正在处理...") is sent to QQ before
  the Hermes call begins.

### 7. Write one-click startup/stop scripts

See `templates/qq-start.sh.template` and `templates/qq-stop.sh.template`.
The startup order is: Hermes Gateway → Bridge → NapCat. Each waits for its
port to be listening before proceeding.

### 8. Launch — quick login + QR fallback

```bash
bash ~/napcat-astrbot/qq-start.sh
```

The script starts NapCat with `-q <QQ_BOT_ACCOUNT>` for **quick login** using
stored credentials — this skips the QR code entirely if tokens are still
valid (typically ~15s to login). If tokens have expired, NapCat falls back to
QR scan mode automatically.

NapCat generates a QR code at `napcat/cache/qrcode.png`. The startup script
copies it to the Windows Desktop for easy scanning:

```bash
cp ~/napcat-astrbot/napcat/cache/qrcode.png /mnt/c/Users/<user>/Desktop/
```

After scanning with phone QQ, port 3000 comes up and the bridge starts
receiving events. The script detects port 3000 and automatically sends a
confirmation message to the owner's QQ.

> **Quick login (`-q`):** After the first successful QR scan, NTQQ stores
> login tokens in `~/.config/QQ/`. Subsequent restarts with `-q <QQID>` will
> auto-login in ~15s without any scan. Tokens can expire — if NapCat shows
> `Login Error, ErrType: 1, ErrCode: 3`, it falls back to QR mode.

> **QR expiry:** NapCat QR codes expire after ~2 minutes. The startup script
> detects refreshed QR codes (by file timestamp) and re-copies them to the
> Desktop automatically. If scanning fails, just wait for the next one.

## Pitfalls

### 0. NapCat account-specific config shadows onebot11.json

When NapCat logs in with a QQ account, it creates per-account config files
like `onebot11_<QQID>.json` in `napcat/config/`. These **override** the
generic `onebot11.json`. If you change `onebot11.json` after the first login,
NapCat will ignore it and load the stale account-specific copy.

**Fix:** After changing the OneBot config, also overwrite the account-specific
file:

```bash
cp napcat/config/onebot11.json napcat/config/onebot11_<QQID>.json
```

Then restart NapCat. The account-specific filename pattern is
`onebot11_<QQ_number>.json` (e.g. `onebot11_<YOUR_BOT_QQ>.json`).

### 1. NTQQ/NapCat proxy interception of localhost

If `HTTP_PROXY`/`http_proxy` env vars are set, curl and aiohttp will route
`127.0.0.1` requests through the proxy, getting 502 Bad Gateway. Always set
`NO_PROXY="127.0.0.1,localhost"` or use `--noproxy '*'` with curl when
accessing local services.

**Critical:** NapCat's internal Node.js HTTP client (the `httpClients` config
that POSTs OneBot events to the bridge) **also respects `HTTP_PROXY`** — but
`env -u HTTP_PROXY` alone is NOT sufficient. Even after unsetting the env var
from the child process, NapCat's Node.js runtime may still pick up proxy
settings from other sources (system configs, DNS-level TUN interception by
Clash/VPN).

**Fix (proven in production):** Both unset the proxy vars AND explicitly set
`NO_PROXY` in the child environment:

```bash
env -u HTTP_PROXY -u HTTPS_PROXY -u http_proxy -u https_proxy \
    NO_PROXY="127.0.0.1,localhost,::1" no_proxy="127.0.0.1,localhost,::1" \
    xvfb-run -a ntqq/opt/QQ/qq --no-sandbox ...
```

Set `NO_PROXY` in BOTH uppercase and lowercase (Node.js checks lowercase
`no_proxy` on some platforms). Without this, NapCat silently routes event
POSTs through the VPN's TUN-mode fake-IP DNS (`198.18.0.x`), getting 502
Bad Gateway — no error in NapCat logs, bridge never receives events.

### 2. NapCat file layout — absolute vs relative paths

The Docker entrypoint copies NapCat to `/app/napcat/` and uses an absolute
`file:///app/napcat/napcat.mjs` in `loadNapCat.js`. If you try to use a
relative `./napcat/napcat.mjs`, NTQQ resolves it relative to
`resources/app/` and fails with `ERR_MODULE_NOT_FOUND`. Always use the full
absolute path.

### 3. PyPI mirrors in China return 403 on wheel downloads

Tsinghua/aliyun PyPI mirrors may return 200 on the simple index but 403 on
actual `.whl` downloads. If `uv tool install` fails with 403, switch to
`--index-url "https://pypi.org/simple"` and route through the proxy.

### 4. apt stale proxy config

WSL may have a leftover proxy at `198.18.0.1:7891` from a previous VPN
session. `apt-get update` fails with "Connection refused." Fix: pass
`-o Acquire::http::Proxy="" -o Acquire::https::Proxy=""` to bypass.

### 5. xvfb authorization

Running `ntqq/opt/QQ/qq` directly with `DISPLAY=:99` fails with "Authorization
required, but no authorization protocol specified." Always use `xvfb-run -a`
which sets up the X authority file automatically.

### 6. nohup backgrounding blocked by Hermes terminal tool

The Hermes `terminal` tool rejects shell-level `&` / `nohup` / `setsid`
backgrounding in foreground mode. Use `terminal(background=true)` to start
long-lived processes, or write a startup script that uses `nohup ... &`
internally and run that script in foreground.

### 7. NapCat config directory location

NapCat looks for `onebot11.json`, `webui.json`, and `napcat.json` in its own
`config/` subdirectory (e.g. `~/napcat-astrbot/napcat/config/`), NOT in the
NTQQ resources directory. Placing configs in the wrong location means NapCat
loads with defaults and the OneBot network config is silently ignored.

### 8. Session ID becomes stale after Gateway restart (silent context loss)

**Symptom:** The user reports that Hermes "loses memory" mid-conversation —
the agent starts a new session even though the bridge is using the same
session ID.

**Root cause:** The bridge uses a hardcoded session ID (e.g.
`api_1782539173_f78ea9ee`). When the Hermes Gateway restarts or the session
expires, the ID becomes stale. The `/api/sessions/{SID}/chat` endpoint may
silently create a new session with a different internal ID, while the bridge
continues sending to the old ID. The bridge has no idea this happened.

**Fix (v3 bridge):**
1. Never hardcode session IDs — generate a fresh one on each chain startup.
2. Persist the session ID to a file (`current_session.txt`).
3. On 404 from `/api/sessions/{SID}/chat`, call `ensure_session()` to recreate
   the session with the same ID, then retry the chat call once.
4. Provide QQ commands (`/new`, `/switch`, `/session`, `/list`) so the user
   can manually manage sessions from their phone.

### 9. NapCat GPU process crash on startup (FATAL: GPU process isn't usable)

**Symptom:** NapCat starts, logs in successfully, sends the confirmation
message — then crashes ~1 minute later with:
```
ERROR:content/browser/gpu/gpu_process_host.cc:951] GPU process launch failed: error_code=1002
FATAL:content/browser/gpu/gpu_data_manager_impl_private.cc:415] GPU process isn't usable. Goodbye.
```

**Root cause:** NTQQ is an Electron app. Even with `--disable-gpu
--disable-software-rasterizer` flags, the Electron zygote still tries to
spawn a GPU child process. On WSL2 without a real GPU, this fails 3 times
and Electron's GPU data manager calls `FATAL` — killing the entire NTQQ
process, which takes NapCat down with it.

**Fix (proven in production):** `--disable-gpu` alone is NOT sufficient.
Add `--disable-zygote --in-process-gpu` to prevent the zygote from
spawning GPU children and force GPU work in-process:
```bash
xvfb-run -a ntqq/opt/QQ/qq \
    --no-sandbox --disable-gpu --disable-software-rasterizer \
    --disable-zygote --in-process-gpu \
    -q "$QQ_BOT"
```
This combination has been tested and confirmed stable on WSL2 Ubuntu 26.04
with NTQQ headless + xvfb. `--use-gl=swiftshader` is an alternative but
adds software rendering overhead; `--in-process-gpu` is lighter.

> **Note:** This crash happens AFTER successful login and message sending,
> so the bridge appears to work for ~1 minute before dying. Always check
> `napcat.log` for FATAL errors if the chain stops working shortly after
> startup.

### 10. Hermes API response format mismatch (/list returns empty)

**Symptom:** The `/list` QQ command returns "没有找到会话列表" even though
sessions exist.

**Root cause:** The Hermes Gateway `/api/sessions` endpoint returns:
```json
{"object": "list", "data": [...]}
```
NOT `{"sessions": [...]}` or a bare array. If `list_sessions()` tries to
extract `data.get("sessions", [])`, it gets an empty list.

**Fix:** Parse the `data` key from the response:
```python
sessions = data.get("data") or data.get("sessions") or []
```
The `data.get("data")` fallback handles the actual API format; the
`data.get("sessions")` fallback is kept for forward-compat in case the
API changes.

### 11. Startup confirmation message should include Session ID

When the chain starts, the confirmation QQ message should include the
current session ID (read from `current_session.txt`) and list available
bridge commands. This helps the user verify the session was created and
knows how to manage it remotely:

```bash
session_id=$(cat "$BASE_DIR/current_session.txt" 2>/dev/null | head -1 | tr -d '[:space:]')
if [ -n "$session_id" ]; then
    confirm_msg="🌉 NapCat ↔ Hermes 桥接已上线！...\\n\\n📌 当前会话: $session_id\\n\\n指令: /new /switch <id> /session /list /help"
fi
```

> **Note:** The `current_session.txt` file is written by the bridge on
> startup. There is a brief race window between bridge startup and the
> confirmation message being sent. The startup script reads the file
> AFTER detecting port 3000 (NapCat login), which gives the bridge enough
> time to have written it.

### 12. Do not read all Hermes internal source files when debugging

When investigating Hermes Gateway or API behavior, do NOT attempt to read
all internal source files (gateway/*.py, agent/*.py, etc.). This will
rapidly overflow the agent's context window. Instead:

- Use `curl` to probe the API endpoints directly and observe response
  formats.
- `grep -n` for specific keywords (e.g. `session`, `chat`, `id`) in
  specific files — never `cat` entire modules.
- Read only the bridge script (`bridge.py`) and the startup script
  (`qq-start.sh`) in full; these are small and contain the relevant
  logic. The Hermes internals are not needed to debug bridge-level
  session management issues.

### 13. NapCat `-q` quick login parameter is easy to miss

**Symptom:** Every chain restart requires a QR code scan, even though
NapCat was previously logged in successfully. The user expects automatic
login but gets a QR code every time.

**Root cause:** NapCat supports a `-q <QQID>` command-line flag for
**quick login** — it reuses stored login credentials from
`~/.config/QQ/` to auto-login in ~15 seconds without scanning a QR
code. Without `-q`, NapCat defaults to QR scan mode every time. The
parameter is documented in NapCat's CLI help but not prominently in
the Wiki, making it easy to miss during initial deployment.

**Fix:** Always pass `-q <QQ_BOT_ACCOUNT>` in the NapCat launch command:
```bash
xvfb-run -a ntqq/opt/QQ/qq \
    --no-sandbox --disable-gpu --disable-software-rasterizer \
    --disable-zygote --in-process-gpu \
    -q "$QQ_BOT"
```

After the first successful QR scan, NTQQ stores login tokens in
`~/.config/QQ/`. Subsequent restarts with `-q` auto-login in ~15s. If
tokens have expired, NapCat automatically falls back to QR scan mode —
no manual intervention needed.

> **Pitfall within a pitfall:** The `-q` flag requires the QQ account
> to have been previously logged in on this machine. On a fresh install
> (no `~/.config/QQ/` directory), `-q` silently falls back to QR mode
> without warning — this is expected behavior, not a bug. The first
> startup will always require a QR scan; subsequent startups use `-q`.

## Verification

After startup, verify all four ports are listening:

```bash
ss -tlnp | grep -E '(8642|8080|3000|6099)'
```

Test the Hermes API directly:

```bash
curl --noproxy '*' -s http://127.0.0.1:8642/health
# Should return: {"status": "ok", "platform": "hermes-agent", ...}
```

Check bridge health (includes current session ID):

```bash
curl --noproxy '*' -s http://127.0.0.1:8080/health
# Should return: {"status": "ok", "service": "napcat-hermes-bridge", "session": "qq_..."}
```

Check bridge logs for incoming messages:

```bash
tail -f ~/napcat-astrbot/logs/bridge.log
```

Send a test message from phone QQ → bridge log should show `📨 ...`,
followed by `💬 Hermes replied in Xs: ...`.

Test bridge commands from QQ:
- `/help` → should list all commands
- `/session` → should show current session ID (TUI format: `YYYYMMDD_HHMMSS_6hex`)
- `/list` → should list all Hermes sessions (if empty, check API response format — see pitfall #10)

Verify session ID format matches TUI:
```bash
cat ~/napcat-astrbot/current_session.txt
# Should look like: 20260629_153354_2d6137 (NOT qq_1782718237_fcbb8013)
```

Verify API response format for `/list`:
```bash
curl --noproxy '*' -s -H "Authorization: Bearer <KEY>" http://127.0.0.1:8642/api/sessions | python3 -m json.tool | head -5
# Should show {"object": "list", "data": [...]}
```

Check for NapCat GPU crash (pitfall #9):

```bash
grep -i 'FATAL\|GPU process' ~/napcat-astrbot/logs/napcat.log
# If found, add --disable-zygote --in-process-gpu to qq-start.sh NapCat launch flags
```

---

# Reference: NapCat Configuration

# NapCat Configuration Reference

## Account-Specific Config Files

NapCat creates per-account config files on first login that **override** the
generic config files. The pattern is `<config_name>_<QQ_number>.json`.

| Generic file | Account-specific override |
|---|---|
| `onebot11.json` | `onebot11_<YOUR_BOT_QQ>.json` |
| `napcat.json` | `napcat_<YOUR_BOT_QQ>.json` |
| `napcat_protocol.json` | `napcat_protocol_<YOUR_BOT_QQ>.json` |
| `webui.json` | (not overridden — shared) |

**If you change `onebot11.json` after the first login, NapCat will ignore it.**
You must also overwrite the account-specific file:

```bash
cp napcat/config/onebot11.json napcat/config/onebot11_<QQID>.json
```

Then restart NapCat.

## Config Directory Location

NapCat looks for config files in its own `config/` subdirectory, relative to
where `napcat.mjs` lives:

```
~/napcat-astrbot/napcat/config/
├── napcat.json                     # Core NapCat settings (logging, packet, bypass)
├── napcat_<QQID>.json              # Per-account core override
├── onebot11.json                   # OneBot v11 network config (generic)
├── onebot11_<QQID>.json            # Per-account OneBot override ← THE IMPORTANT ONE
├── napcat_protocol_<QQID>.json     # Per-account NapCat Protocol config
└── webui.json                       # WebUI settings (shared)
```

## OneBot v11 HTTP Server Mode

For the Hermes bridge architecture, NapCat should be configured with:

1. **HTTP Server** on port 3000 — for the bridge to call `send_msg` API
2. **HTTP Client** posting to `http://127.0.0.1:8080/onebot` — for event forwarding

```json
{
  "network": {
    "httpServers": [{
      "enable": true, "name": "api",
      "host": "127.0.0.1", "port": 3000,
      "messagePostFormat": "array", "token": "",
      "debug": false, "enableForcePushEvent": true
    }],
    "httpClients": [{
      "enable": true, "name": "bridge",
      "url": "http://127.0.0.1:8080/onebot",
      "messagePostFormat": "array", "reportSelfMessage": false, "token": ""
    }],
    "websocketServers": [],
    "websocketClients": []
  }
}
```

## QQ Version Matching

NapCat releases specify a recommended QQ NTQQ version. Using a mismatched
version produces a warning:

```
[Core] [Packet] PacketBackend 不支持当前QQ版本架构：0.0.1-x64
```

This warning is non-fatal — NapCat still works, but some packet-level
features (like fetching specific message types) may not function. For basic
messaging (send/receive text), it works fine with version mismatches.

## Login Credential Persistence

NTQQ stores login tokens in `~/.config/QQ/`. After the first successful QR
code scan:

- Subsequent NapCat restarts **may** auto-login without a QR code
- Tokens **can expire**, requiring re-scan
- `Login Error, ErrType: 1, ErrCode: 3` means QR code expired (not a real
  error) — NapCat will auto-generate a new QR code

### Quick Login with `-q`

NapCat supports `-q <QQID>` flag for quick login using stored credentials:

```bash
xvfb-run -a ntqq/opt/QQ/qq --no-sandbox --disable-gpu -q <YOUR_BOT_QQ>
```

On startup, NapCat prints available quick-login accounts:
```
可用于快速登录 of QQ：
1. <YOUR_BOT_QQ> 千寻
```

With `-q`, login takes ~15s and skips QR entirely. If tokens have expired,
NapCat falls back to QR scan mode automatically. The `qq-start.sh` template
uses `-q` by default.

## WebUI Token

NapCat's WebUI (`http://localhost:6099/webui`) uses a token for auth. The
token is printed in the console log on startup:

```
[WebUi] WebUi Token: <random-token>
[WebUi] WebUi User Panel Url: http://127.0.0.1:6099/webui?token=<random-token>
```

If the token is set to `"napcat"` in `webui.json`, NapCat will auto-generate
a random secure token on startup and print it in the log. Check with:

```bash
grep "WebUi Token" ~/napcat-astrbot/logs/napcat.log | tail -1
```

---

# Reference: WSL Proxy Setup

# WSL Proxy Setup

## Finding the Windows host IP from WSL

WSL2 uses a virtual network adapter. The Windows host is reachable at the
**default gateway** of the WSL virtual interface, NOT at `127.0.0.1` or
`localhost` (those refer to the WSL VM itself).

```bash
# Get the Windows host IP (WSL gateway)
WINHOST=$(ip route | grep default | awk '{print $3}')
echo "Windows host: $WINHOST"
# Typically 172.x.x.1
```

## Common proxy configurations

| VPN/Proxy tool | Listening on | Accessible from WSL? |
|---|---|---|
| Clash for Windows | `127.0.0.1:7890` | ❌ (Windows localhost only) |
| Clash with "Allow LAN" | `0.0.0.0:7890` | ✅ via `WINHOST:7890` |
| V2RayN | `127.0.0.1:<PROXY_PORT>` | ❌ by default, ✅ if "allow LAN" enabled |
| Custom proxy | `<WSL_GATEWAY_IP>:<PROXY_PORT>` | ✅ if bound to the WSL gateway IP |

## Testing connectivity

```bash
WINHOST=$(ip route | grep default | awk '{print $3}')
timeout 10 curl -x "http://$WINHOST:<PROXY_PORT>" -sI "https://pypi.org/simple" | head -3
# HTTP/1.1 200 Connection established → proxy works
```

## Setting proxy for tools

```bash
export HTTP_PROXY="http://$WINHOST:<PROXY_PORT>"
export HTTPS_PROXY="$HTTP_PROXY"
export NO_PROXY="127.0.0.1,localhost"  # CRITICAL: don't proxy localhost!
```

## Pitfalls

- **`/etc/resolv.conf` nameserver is NOT the Windows host IP.** On WSL2 with
  systemd, `resolv.conf` often shows `10.255.255.254` (a DNS forwarder), not
  the actual gateway. Use `ip route` to find the real gateway.
- **Proxy env vars leak into localhost requests.** If `NO_PROXY` is not set,
  curl/aiohttp will route `http://127.0.0.1:8642` through the proxy, getting
  502 Bad Gateway. Always set `NO_PROXY`.
- **NapCat's Node.js HTTP client respects `HTTP_PROXY` but may not respect
  `NO_PROXY` alone.** Even with `NO_PROXY=127.0.0.1` set, NapCat's
  `httpClients` event poster can still route localhost POSTs through
  `HTTP_PROXY` — especially if a VPN uses TUN-mode DNS interception
  (`198.18.0.x` fake IPs). This causes silent event delivery failure: the
  bridge never receives messages and no error appears in logs.
  Fix: BOTH unset the proxy vars AND explicitly set `NO_PROXY` in the child
  environment:
  ```bash
  env -u HTTP_PROXY -u HTTPS_PROXY -u http_proxy -u https_proxy \
      NO_PROXY="127.0.0.1,localhost,::1" no_proxy="127.0.0.1,localhost,::1" \
      xvfb-run -a ntqq/opt/QQ/qq --no-sandbox ...
  ```
  Set `NO_PROXY` in BOTH uppercase and lowercase (Node.js checks lowercase
  `no_proxy` on some platforms).
- **apt has its own proxy config.** Even if you unset env vars, apt may have a
  stale proxy in `/etc/apt/apt.conf.d/`. Bypass with
  `-o Acquire::http::Proxy=""`.

---

# Template: bridge.py

```python
#!/usr/bin/env python3
"""
NapCat ↔ Hermes Bridge v3

Session management:
- Each chain startup creates a FRESH Hermes session (avoids cross-contamination)
- Session ID persisted to file, survives bridge restarts within same chain run
- QQ commands: /new /switch <id> /session /list /help
- 404 auto-recovery: if session disappears mid-conversation, recreate + retry

Architecture:
    QQ消息 → NapCat → POST :8080/onebot → bridge.py
                                        ↓ ACK → QQ
                                        ↓ /api/sessions/{SID}/chat → Hermes API (:8642)
    QQ回复 ← NapCat HTTP API (:3000) ← bridge.py ← final response

Requires: aiohttp

Usage:
    HERMES_API_KEY=*** ALLOWED_QQ=<YOUR_OWNER_QQ> python3 bridge.py
"""

import asyncio
import json
import logging
import os
import time
from typing import Any, Optional

from aiohttp import web, ClientSession, ClientTimeout

# ── Config ──────────────────────────────────────────────────────────────────
NAPCAT_API = "http://127.0.0.1:3000"
HERMES_API = "http://127.0.0.1:8642"
HERMES_KEY = os.environ.get("HERMES_API_KEY", "<your-api-key>")
BRIDGE_HOST = "127.0.0.1"
BRIDGE_PORT = 8080
ALLOWED_QQ = int(os.environ.get("ALLOWED_QQ", "0"))

# Session persistence: store in file so it survives bridge restarts
SCRIPT_DIR = os.path.dirname(os.path.abspath(__file__))
SESSION_FILE = os.path.join(SCRIPT_DIR, "current_session.txt")

logging.basicConfig(level=logging.INFO, format="%(asctime)s [%(levelname)s] %(message)s")
log = logging.getLogger("bridge")

# ── Session persistence helpers ─────────────────────────────────────────────

def load_session_id() -> str:
    """Load session ID from persistence file. Returns empty string if none."""
    try:
        with open(SESSION_FILE, "r") as f:
            return f.read().strip()
    except (FileNotFoundError, IOError):
        return ""

def save_session_id(sid: str) -> None:
    """Persist session ID to file."""
    try:
        with open(SESSION_FILE, "w") as f:
            f.write(sid)
    except IOError as e:
        log.warning(f"Failed to save session ID: {e}")

# ── Session management ──────────────────────────────────────────────────────

async def create_session(sid: str = None, title: str = None) -> str:
    """Create a new Hermes session. Returns the session ID."""
    headers = {"Authorization": f"Bearer {HERMES_KEY}", "Content-Type": "application/json"}
    if sid is None:
        # Match Hermes TUI format: YYYYMMDD_HHMMSS_6hex
        sid = time.strftime("%Y%m%d_%H%M%S") + "_" + os.urandom(3).hex()
    if title is None:
        title = f"QQ Bridge {time.strftime('%Y-%m-%d %H:%M')}"

    try:
        async with ClientSession(trust_env=False) as session:
            async with session.post(
                f"{HERMES_API}/api/sessions",
                headers=headers,
                json={"id": sid, "title": title},
                timeout=ClientTimeout(total=10),
            ) as resp:
                if resp.status in (200, 201):
                    data = await resp.json()
                    created_sid = data.get("session", {}).get("id", sid)
                    log.info(f"Session created: {created_sid}")
                    return created_sid
                else:
                    text = await resp.text()
                    log.warning(f"Session create returned {resp.status}: {text[:200]}")
                    return sid
    except Exception as e:
        log.warning(f"Session create failed ({e}), using {sid}")
        return sid

async def ensure_session(sid: str) -> str:
    """Verify a session exists. If not, create it with the same ID. Return its ID."""
    headers = {"Authorization": f"Bearer {HERMES_KEY}"}
    try:
        async with ClientSession(trust_env=False) as session:
            async with session.get(
                f"{HERMES_API}/api/sessions/{sid}",
                headers=headers,
                timeout=ClientTimeout(total=10),
            ) as resp:
                if resp.status == 200:
                    log.info(f"Session exists: {sid}")
                    return sid
    except Exception:
        pass
    log.info(f"Session {sid} not found, creating...")
    return await create_session(sid)

async def list_sessions() -> list[dict]:
    """List all sessions from Hermes API."""
    headers = {"Authorization": f"Bearer {HERMES_KEY}"}
    try:
        async with ClientSession(trust_env=False) as session:
            async with session.get(
                f"{HERMES_API}/api/sessions",
                headers=headers,
                timeout=ClientTimeout(total=10),
            ) as resp:
                if resp.status == 200:
                    data = await resp.json()
                    # API returns {"object": "list", "data": [...]}
                    if isinstance(data, dict):
                        sessions = data.get("data") or data.get("sessions") or []
                    else:
                        sessions = data
                    return sessions if isinstance(sessions, list) else []
                return []
    except Exception as e:
        log.error(f"Failed to list sessions: {e}")
        return []

# ── QQ helpers ──────────────────────────────────────────────────────────────

def extract_text(message: list[dict] | str) -> str:
    if isinstance(message, str):
        return message
    parts = []
    for seg in message:
        t = seg.get("type", "")
        if t == "text":
            parts.append(seg.get("data", {}).get("text", ""))
        elif t == "image":
            parts.append("[图片]")
        elif t == "face":
            parts.append("[表情]")
        elif t == "record":
            parts.append("[语音]")
        elif t == "video":
            parts.append("[视频]")
        elif t == "file":
            parts.append(f"[文件:{seg.get('data',{}).get('file','')}]")
    return "".join(parts).strip()

async def send_to_qq(message_type: str, target_id: int, text: str) -> bool:
    """Send a message via NapCat HTTP API. Returns True on success."""
    endpoint = f"{NAPCAT_API}/send_{message_type}_msg"
    params = {
        "message_type": message_type,
        "message": [{"type": "text", "data": {"text": text}}],
    }
    if message_type == "private":
        params["user_id"] = target_id
    else:
        params["group_id"] = target_id
    try:
        async with ClientSession(trust_env=False) as session:
            async with session.post(endpoint, json=params, timeout=ClientTimeout(total=30)) as resp:
                result = await resp.json()
                return result.get("status") == "ok"
    except Exception as e:
        log.error(f"Failed to send to QQ: {e}")
        return False

# ── Hermes API call ─────────────────────────────────────────────────────────

PERSISTENT_SID: str = ""

async def call_hermes_session(prompt: str, session_id: str) -> str:
    """Call Hermes session chat endpoint. Auto-recovers from 404."""
    global PERSISTENT_SID
    headers = {"Authorization": f"Bearer {HERMES_KEY}", "Content-Type": "application/json"}
    payload = {"message": prompt}

    async def _do_chat(sid: str) -> tuple[int, str]:
        async with ClientSession(trust_env=False) as session:
            async with session.post(
                f"{HERMES_API}/api/sessions/{sid}/chat",
                headers=headers,
                json=payload,
                timeout=ClientTimeout(total=600),
            ) as resp:
                if resp.status != 200:
                    error_text = await resp.text()
                    return resp.status, f"[Hermes API 错误 {resp.status}: {error_text[:100]}]"
                data = await resp.json()
                return 200, data.get("message", {}).get("content", "[空回复]")

    try:
        status, text = await _do_chat(session_id)
        if status == 404:
            log.warning(f"Session {session_id} not found (404), recreating...")
            new_sid = await ensure_session(session_id)
            PERSISTENT_SID = new_sid
            save_session_id(new_sid)
            log.info(f"Session recreated: {new_sid}, retrying chat...")
            status, text = await _do_chat(new_sid)
        return text
    except Exception as e:
        log.error(f"Failed to call Hermes session API: {e}")
        return f"[调用 Hermes 失败: {e}]"

# ── Bridge commands ─────────────────────────────────────────────────────────

async def handle_bridge_command(text: str) -> Optional[str]:
    """Handle bridge-level QQ commands. Returns response text or None if not a command."""
    global PERSISTENT_SID
    cmd_parts = text.strip().split()
    if not cmd_parts:
        return None
    command = cmd_parts[0].lower()

    if command == "/new":
        new_sid = await create_session()
        PERSISTENT_SID = new_sid
        save_session_id(new_sid)
        return f"✅ 已创建新会话: {new_sid}\n之前的历史将不再可见。"

    elif command == "/switch" and len(cmd_parts) > 1:
        target_sid = cmd_parts[1]
        headers = {"Authorization": f"Bearer {HERMES_KEY}"}
        try:
            async with ClientSession(trust_env=False) as session:
                async with session.get(
                    f"{HERMES_API}/api/sessions/{target_sid}",
                    headers=headers,
                    timeout=ClientTimeout(total=10),
                ) as resp:
                    if resp.status == 200:
                        PERSISTENT_SID = target_sid
                        save_session_id(target_sid)
                        return f"✅ 已切换到会话: {target_sid}"
                    else:
                        return f"❌ 会话 {target_sid} 不存在 (HTTP {resp.status})"
        except Exception as e:
            return f"❌ 切换会话失败: {e}"

    elif command == "/session":
        return f"📌 当前会话: {PERSISTENT_SID}"

    elif command == "/list":
        sessions = await list_sessions()
        if not sessions:
            return "📋 没有找到会话列表"
        lines = ["📋 可用会话:"]
        for s in sessions[:20]:
            sid = s.get("id", "?")
            title = s.get("title", "")
            marker = " ← 当前" if sid == PERSISTENT_SID else ""
            lines.append(f"  • {sid}  ({title}){marker}")
        return "\n".join(lines)

    elif command == "/help":
        return (
            "🌉 Bridge 指令:\n"
            "  /new         — 开启新会话（清除上下文）\n"
            "  /switch <id> — 切换到指定会话\n"
            "  /session     — 显示当前会话 ID\n"
            "  /list        — 列出所有会话\n"
            "  /help        — 显示此帮助\n"
            "其他消息将直接发给 Hermes Agent 对话。"
        )

    return None

# ── Event Handler ────────────────────────────────────────────────────────────

async def handle_onebot_event(request: web.Request) -> web.Response:
    try:
        event = await request.json()
    except Exception:
        return web.Response(status=400, text="Invalid JSON")

    post_type = event.get("post_type", "")
    if post_type != "message":
        return web.json_response({"status": "ok", "ignored": True})

    message_type = event.get("message_type", "")
    user_id = event.get("user_id", 0)
    group_id = event.get("group_id", 0)
    sender = event.get("sender", {})
    sender_name = sender.get("nickname", str(user_id))
    text = extract_text(event.get("message", ""))

    if not text:
        return web.json_response({"status": "ok", "ignored": "empty"})

    if ALLOWED_QQ and user_id != ALLOWED_QQ:
        log.info(f"🚫 Ignored message from {sender_name}({user_id}) — not owner")
        return web.json_response({"status": "ok", "ignored": "not_owner"})

    if message_type == "group":
        target_id = group_id
        reply_type = "group"
        log.info(f"📨 Group {group_id} | {sender_name}({user_id}): {text[:100]}")
    else:
        target_id = user_id
        reply_type = "private"
        log.info(f"📨 Private | {sender_name}({user_id}): {text[:100]}")

    # Check for bridge commands (/new, /switch, /session, /list, /help)
    if text.startswith("/"):
        cmd_response = await handle_bridge_command(text)
        if cmd_response is not None:
            await send_to_qq(reply_type, target_id, cmd_response)
            return web.json_response({"status": "ok", "command": True})

    if message_type == "group":
        prompt = f"[QQ群消息 来自 {sender_name}(QQ:{user_id})]\n{text}"
    else:
        prompt = f"[QQ私聊消息 来自 {sender_name}(QQ:{user_id})]\n{text}"

    async def process_and_reply():
        await send_to_qq(reply_type, target_id, "⚡ 已收到，正在处理...")
        t0 = time.time()
        reply = await call_hermes_session(prompt, PERSISTENT_SID)
        elapsed = time.time() - t0
        log.info(f"💬 Hermes replied in {elapsed:.1f}s: {reply[:100]}...")
        await send_to_qq(reply_type, target_id, reply)

    asyncio.create_task(process_and_reply())
    return web.json_response({"status": "ok"})

# ── Health ───────────────────────────────────────────────────────────────────

async def handle_health(request: web.Request) -> web.Response:
    return web.json_response({
        "status": "ok",
        "service": "napcat-hermes-bridge",
        "session": PERSISTENT_SID,
    })

# ── Main ─────────────────────────────────────────────────────────────────────

async def startup(app: web.Application):
    """Create a FRESH session on each chain startup (no cross-contamination)."""
    global PERSISTENT_SID
    log.info("Creating fresh session for this chain instance...")
    PERSISTENT_SID = await create_session()
    save_session_id(PERSISTENT_SID)
    log.info(f"🌉 NapCat ↔ Hermes Bridge v3 starting on {BRIDGE_HOST}:{BRIDGE_PORT}")
    log.info(f"   NapCat API: {NAPCAT_API}")
    log.info(f"   Hermes API: {HERMES_API}")
    log.info(f"   Session:    {PERSISTENT_SID}  (fresh)")
    log.info(f"   Owner QQ:   {ALLOWED_QQ}")
    log.info(f"   Commands:   /new /switch <id> /session /list /help")

def main():
    app = web.Application()
    app.on_startup.append(startup)
    app.router.add_post("/onebot", handle_onebot_event)
    app.router.add_get("/health", handle_health)
    web.run_app(app, host=BRIDGE_HOST, port=BRIDGE_PORT, print=None)

if __name__ == "__main__":
    main()
```

---

# Template: qq-start.sh

```bash
#!/bin/bash
#
# qq-start.sh — 一键连锁启动 NapCat + Hermes Gateway + Bridge
#
# Usage: ~/napcat-astrbot/qq-start.sh
# Stop:  ~/napcat-astrbot/qq-stop.sh
#
# Customize: EDIT the BASE_DIR, proxy, Windows username, QQ IDs below.
#

set -euo pipefail

BASE_DIR="$HOME/napcat-astrbot"
LOG_DIR="$BASE_DIR/logs"
PID_DIR="$BASE_DIR/pids"
QR_FILE="$BASE_DIR/napcat/cache/qrcode.png"
DESKTOP_QR="/mnt/c/Users/<WIN_USER>/Desktop/napcat-qrcode.png"
QQ_OWNER=<YOUR_OWNER_QQ>          # 大号 QQ，用于发送启动确认消息
QQ_BOT=<YOUR_BOT_QQ>            # 小号 QQ（NapCat 登录账号），用于快速登录
PROXY_HOST="<WSL_GATEWAY_IP>"     # WSL gateway IP
PROXY_PORT="<PROXY_PORT>"           # Proxy port on Windows host
WIN_USER="<WIN_USER>"        # Windows username for QR code copy

mkdir -p "$LOG_DIR" "$PID_DIR"

# ── Proxy: 只给 Hermes Gateway 用，NapCat 不走代理 ──────────────────────────
export HTTPS_PROXY="http://${PROXY_HOST}:${PROXY_PORT}"
export NO_PROXY="127.0.0.1,localhost,::1"
export HOME="${HOME:-/home/yuki}"

RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[0;33m'; CYAN='\033[0;36m'; NC='\033[0m'
log()  { echo -e "${CYAN}[$(date +%H:%M:%S)]${NC} $1"; }
ok()   { echo -e "${GREEN}[$(date +%H:%M:%S)] ✓ $1${NC}"; }
warn() { echo -e "${YELLOW}[$(date +%H:%M:%S)] ⚠ $1${NC}"; }
fail() { echo -e "${RED}[$(date +%H:%M:%S)] ✗ $1${NC}"; exit 1; }

start_gateway() {
    log "Starting Hermes Gateway (API Server :8642)..."
    cd /home/yuki/.hermes/hermes-agent
    nohup ./venv/bin/python -m gateway.run > "$LOG_DIR/gateway.log" 2>&1 &
    echo $! > "$PID_DIR/gateway.pid"
    for i in $(seq 1 15); do
        ss -tlnp 2>/dev/null | grep -q ":8642" && { ok "Hermes Gateway (pid $(cat $PID_DIR/gateway.pid), :8642)"; return 0; }
        sleep 1
    done
    fail "Hermes Gateway failed to start"
}

start_bridge() {
    log "Starting Bridge (:8080)..."
    cd "$BASE_DIR"
    nohup /home/yuki/uv-env/bin/python3 bridge.py > "$LOG_DIR/bridge.log" 2>&1 &
    echo $! > "$PID_DIR/bridge.pid"
    for i in $(seq 1 10); do
        ss -tlnp 2>/dev/null | grep -q ":8080" && { ok "Bridge (pid $(cat $PID_DIR/bridge.pid), :8080)"; return 0; }
        sleep 1
    done
    fail "Bridge failed to start"
}

start_napcat() {
    log "Starting NapCat (NTQQ headless + xvfb)..."
    cd "$BASE_DIR"
    # CRITICAL: Unset ALL proxy env vars AND set NO_PROXY for NapCat.
    # NapCat's internal Node.js HTTP client respects HTTP_PROXY but NOT
    # NO_PROXY — even with NO_PROXY=127.0.0.1 set, it will silently route
    # localhost event POSTs through the proxy (502 Bad Gateway, no error
    # in NapCat logs, bridge never receives events).
    # The NO_PROXY env var must be explicitly set in the child env because
    # env -u only removes from the child, but NapCat's Node.js runtime may
    # still pick up proxy settings from other sources.
    # -q enables quick login using stored credentials (skips QR scan if
    # tokens are still valid). Falls back to QR scan if tokens expired.
    nohup env -u HTTP_PROXY -u HTTPS_PROXY -u http_proxy -u https_proxy \
        NO_PROXY="127.0.0.1,localhost,::1" no_proxy="127.0.0.1,localhost,::1" \
        xvfb-run -a ntqq/opt/QQ/qq \
        --no-sandbox --disable-gpu --disable-software-rasterizer \
        --disable-zygote --in-process-gpu \
        -q "$QQ_BOT" \
        > "$LOG_DIR/napcat.log" 2>&1 &
    echo $! > "$PID_DIR/napcat.pid"
    for i in $(seq 1 40); do
        ss -tlnp 2>/dev/null | grep -q ":6099" && { ok "NapCat (pid $(cat $PID_DIR/napcat.pid), WebUI :6099)"; return 0; }
        sleep 1
    done
    fail "NapCat failed to start"
}

wait_for_login() {
    log "Waiting for NapCat login..."
    local login_wait=0 max_wait=180 qr_announced=false
    while [ $login_wait -lt $max_wait ]; do
        if ss -tlnp 2>/dev/null | grep -q ":3000"; then
            ok "NapCat login successful! HTTP API is up (:3000)"
            sleep 2
            log "Sending startup confirmation to QQ $QQ_OWNER..."
            local session_id=""
            session_id=$(cat "$BASE_DIR/current_session.txt" 2>/dev/null | head -1 | tr -d '[:space:]')
            local confirm_msg
            if [ -n "$session_id" ]; then
                confirm_msg="🌉 NapCat ↔ Hermes 桥接已上线！发送消息即可与 Hermes Agent 对话。\\n\\n📌 当前会话: $session_id\\n\\n指令: /new /switch <id> /session /list /help"
            else
                confirm_msg="🌉 NapCat ↔ Hermes 桥接已上线！发送消息即可与 Hermes Agent 对话。\\n\\n⚠ 未获取到会话 ID，请检查 Bridge 日志。"
            fi
            local resp
            resp=$(curl --noproxy '*' -s -m 10 -X POST http://127.0.0.1:3000/send_private_msg \
                -H "Content-Type: application/json" \
                -d "{\"user_id\": $QQ_OWNER, \"message\": [{\"type\": \"text\", \"data\": {\"text\": \"$confirm_msg\"}}]}" 2>&1)
            echo "$resp" | grep -q '"status":"ok"' && ok "Confirmation sent to QQ $QQ_OWNER ✓" || warn "Failed to send confirmation: $resp"
            return 0
        fi
        # If QR code appears (quick login failed, fell back to scan mode)
        if [ "$qr_announced" = false ] && [ -f "$QR_FILE" ]; then
            qr_announced=true
            cp "$QR_FILE" "$DESKTOP_QR" 2>/dev/null || true
            log "Quick login failed — QR code required. Copied to Desktop."
            log "WebUI: http://localhost:6099/webui"
            log "请用手机 QQ 扫码登录..."
        fi
        # Re-copy QR if refreshed
        if [ "$qr_announced" = true ] && [ -f "$QR_FILE" ]; then
            local new_qr_time
            new_qr_time=$(stat -c %Y "$QR_FILE" 2>/dev/null || echo 0)
            if [ "${new_qr_time}" != "${last_qr_time:-0}" ]; then
                cp "$QR_FILE" "$DESKTOP_QR" 2>/dev/null || true
                last_qr_time="$new_qr_time"
                log "QR code refreshed. 桌面 napcat-qrcode.png 已更新，请重新扫码"
            fi
        fi
        sleep 3; login_wait=$((login_wait + 3))
        [ $((login_wait % 15)) -eq 0 ] && log "Still waiting... (${login_wait}s / ${max_wait}s)"
    done
    warn "Login timeout after ${max_wait}s."
    return 1
}

echo ""
echo "  ╔═════════════════════════════════════════════════════╗"
echo "  ║   NapCat ↔ Hermes Agent — 一键连锁启动             ║"
echo "  ║   扫码登录后自动发 QQ 确认消息                      ║"
echo "  ╚═════════════════════════════════════════════════════╝"
echo ""

pkill -f "gateway.run" 2>/dev/null || true
pkill -f "bridge.py" 2>/dev/null || true
pkill -f "ntqq/opt/QQ/qq" 2>/dev/null || true
pkill -f "xvfb-run.*ntqq" 2>/dev/null || true
sleep 2

start_gateway; sleep 1
start_bridge;  sleep 1
start_napcat
wait_for_login

echo ""
ok "All services are up!"
echo "  Hermes API :8642 | Bridge :8080 | NapCat WebUI :6099 | NapCat API :3000"
echo "  Logs: $LOG_DIR/"
echo "  Stop: ~/napcat-astrbot/qq-stop.sh"
echo ""
```

---

# Template: qq-stop.sh

```bash
#!/bin/bash
# qq-stop.sh — Stop all QQ bridge services
PID_DIR="$HOME/napcat-astrbot/pids"

ok() { echo -e "\033[0;32m[$(date +%H:%M:%S)] ✓ $1\033[0m"; }

echo "Stopping all QQ services..."
for svc in napcat bridge gateway; do
    [ -f "$PID_DIR/$svc.pid" ] && kill "$(cat $PID_DIR/$svc.pid)" 2>/dev/null && rm -f "$PID_DIR/$svc.pid"
done
pkill -f "ntqq/opt/QQ/qq" 2>/dev/null || true
pkill -f "bridge.py" 2>/dev/null || true
pkill -f "gateway.run" 2>/dev/null || true
sleep 2
ok "All services stopped."
```
