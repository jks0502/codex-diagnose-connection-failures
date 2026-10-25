---
name: codex-diagnose-and-fix-connection-failures
description: "Operational guide for agents to diagnose AND repair Codex connection failures ('Connection failed: error sending request', 'Reconnecting... waiting for network'). Walks through evidence collection, a decision table mapping observations to root causes, and copy-paste fix procedures with verification and rollback. Covers the hidden ~/.codex/.env proxy file, poisoned codex processes, dead local proxy ports, VPN node degradation, and provider config. Use when Codex cannot reach chatgpt.com while other apps work, or when connection errors persist across network or config changes."
---

# Codex Connection Failures — Agent Operations Guide

Follow this guide top-to-bottom. Every step gives you: the exact command, the expected output, and a branch telling you where to go next. Do not skip evidence collection and jump to a fix.

## 0. Symptom and the #1 cause

Codex shows `Connection failed: error sending request` or `Reconnecting... waiting for network`. Turns fail at or near 100% rate. Often the ChatGPT desktop app's **chat tab works fine on the same machine**.

The single most common root cause in the wild: **a stale `~/.codex/.env` containing `HTTP_PROXY`/`HTTPS_PROXY` pointing at a local port where no proxy listens anymore**. Codex loads this file at every startup; Chromium (the chat tab) does not read it. "Chat works, Codex doesn't" is the signature of exactly this. You will still run the full procedure to confirm, because a wrong fix costs the user a restart cycle.

## 1. Non-negotiable rules

1. **Never print secrets.** Do not cat `auth.json`; do not echo token values into your report.
2. **Backup before any mutation**, same directory, timestamped: `cp file file.bak-$(date +%Y%m%d-%H%M%S)`.
3. **Strip your own sandbox proxy before spawning codex.** Agent sandboxes commonly inject `HTTP_PROXY=http://127.0.0.1:<port>`. A probe run without stripping is invalid. Use the exact `env -u` prefix shown in Step B3.
4. **Never hand-compute epoch cutoffs.** The log DB stores unix seconds. Convert with the same Python you query with (`datetime.fromtimestamp`). Field agents have lost 10 minutes of reasoning to a 600-second mental-arithmetic error here.
5. Do not kill processes until the evidence names one, and the fix section tells you to.

## 2. Flow overview

```
Phase A (collect evidence) ──► Phase B (branch on findings) ──► Phase C (fix + verify)
   A1 config & env paths         B1 .env proxy found?             F1 remove .env proxy
   A2 log DB classification      B2 fresh codex works?            F2 restart codex process
   A3 ~/.codex/.env              B3 raw network probe?            F3 zombie proxy port
   A4 proxy layers               B4 node benchmark                F4 degraded VPN node
   A5 live netstat                                                F5 VPN down → user action
                                                                 F6 provider config → other skill
```

---

## Phase A — Evidence collection (do all five, in order)

### A1. Resolve the config root and confirm the provider config

```bash
echo "CODEX_HOME=[$CODEX_HOME]"     # empty → default
ls -la "$HOME/.codex/" | head -20
```

Then verify `config.toml` parses and check the active provider:

```bash
python -c "
import tomllib, os
p = os.path.expanduser('~/.codex/config.toml')
cfg = tomllib.load(open(p,'rb'))
name = cfg.get('model_provider')
prov = cfg.get('model_providers',{}).get(name)
print('active provider:', name)
print('fields:', {k: prov.get(k) for k in ('wire_api','supports_websockets','base_url')} if prov else 'MISSING TABLE')
"
```

**Expected:** provider name matches an existing table; if the user runs the disable-WebSocket workaround, `wire_api="responses"`, `supports_websockets=false`, `base_url` present. A clean parse also proves there are no duplicate tables.

**Branch:** config malformed or provider table missing → **F6**, stop here.

### A2. Read the log database and classify the failure

Codex logs live in `<CODEX_HOME>/logs_2.sqlite`, table `logs` (`ts` unix seconds, `level`, `target`, `feedback_log_body`). There is no plain-text log file.

```bash
python << 'EOF'
import sqlite3, os, datetime
con = sqlite3.connect(os.path.expanduser('~/.codex/logs_2.sqlite'))
cur = con.cursor()
print('newest entry:', cur.execute('SELECT MAX(ts) FROM logs').fetchone())
for ts, lv, t, b in cur.execute(
    "SELECT ts, level, target, substr(feedback_log_body,1,260) FROM logs "
    "WHERE level IN ('ERROR','WARN') ORDER BY id DESC LIMIT 10"):
    print(datetime.datetime.fromtimestamp(ts).strftime('%H:%M:%S'), lv, t, '|', b.replace(chr(10),' ')[:240])
print('connect failures total:',
      cur.execute("SELECT COUNT(*) FROM logs WHERE feedback_log_body LIKE '%is_connect=true%'").fetchone())
EOF
```

Classify on three axes — write your classification down before continuing:

- **Scope**: only `/responses` turns failing, or ALL `chatgpt.com` requests (plugin catalog `/backend-api/ps/plugins/installed`, MCP `/backend-api/ps/mcp`)? All of them ⇒ transport-agnostic ⇒ network/client layer.
- **Class**: `is_connect=true` (instant refuse/reset — think dead local proxy), `is_timeout` (slow tunnel), HTTP 4xx/5xx (auth/endpoint).
- **Transport**: `transport="responses_http"` in the span text ⇒ any disable-WebSocket config IS active and is NOT the cause. This closes the most common false lead in one query.

**Branch:** if failures are only on `/responses` with HTTP status codes → auth/endpoint problem, outside this guide. Everything `is_connect=true` → continue.

### A3. Inspect `~/.codex/.env` — the prime suspect

```bash
cat "$HOME/.codex/.env" 2>/dev/null || echo "(no .env file)"
```

If it contains proxy lines, extract the port and test whether it is alive:

```bash
netstat -ano | grep -E "LISTEN" | grep ":<PORT> "   # e.g. :7892
```

**Branch:**
- Proxy line present + no LISTENING line (or SYN_SENT from codex.exe to it — see A5) → **root cause found, go to F1**.
- Proxy line present + port listens → test it once: `curl -x http://127.0.0.1:<PORT> -sS -o /dev/null -w "%{http_code}\n" --max-time 8 https://chatgpt.com` — TLS failure means a zombie listener → **F1 + F3**.
- No proxy lines / no file → continue.

### A4. Check the remaining proxy layers

```bash
env | grep -i proxy        # your shell (only matters if codex inherits it)
```

```bash
python -c "
import winreg
k = winreg.OpenKey(winreg.HKEY_CURRENT_USER, 'Environment')
i = 0
while True:
    try: n, v, _ = winreg.EnumValue(k, i); print('user env:', n, '=', v); i += 1
    except OSError: break
k2 = winreg.OpenKey(winreg.HKEY_CURRENT_USER, r'Software\Microsoft\Windows\CurrentVersion\Internet Settings')
for n in ('ProxyEnable','ProxyServer','AutoConfigURL'):
    try: print(n, '=', winreg.QueryValueEx(k2, n)[0])
    except FileNotFoundError: print(n, '= (not set)')
"
```

Note: use Python `winreg`, not `reg.exe` — `reg.exe` is blacklisted in many agent sandboxes.

**Branch:** any proxy value found pointing at a dead local port → **F1/F3** with that source. All clean → continue.

### A5. Catch codex in the act (netstat)

Find the embedded codex process (inside the ChatGPT desktop app), excluding the sandbox service:

```bash
tasklist | grep -i codex
# note the PID of codex.exe (Console session), not codex-windows-sandbox-ser
netstat -ano | grep "<PID>" | grep -E "SYN_SENT|ESTABLISHED" | head
```

**Read the connections:**
- `SYN_SENT 127.0.0.1:x → 127.0.0.1:<port>` — codex is hammering a dead local proxy → confirm with A3/A4, then **F1**.
- Zero outbound connections while errors stream — the client dies before the NIC: proxy env, almost certainly `.env` → **F1**.
- `SYN_SENT` to a public IP — direct connection being blocked (VPN off) → **F5**.
- `ESTABLISHED` to `198.18.x.x` — traffic enters the TUN stack; problem is node quality → **F4**.

---

## Phase B — Differential probes (only if A didn't name the cause)

### B1. Rule out stale process state: run a FRESH codex

Two gotchas, both mandatory: strip proxy env (rule 3), and close stdin (codex exec hangs on `Reading additional input from stdin...` without `< /dev/null`).

```bash
CODEX_BIN=$(grep CODEX_CLI_PATH "$HOME/.codex/config.toml" | head -1 | sed "s/.*= '//; s/'//")
cd "$HOME/Documents" && timeout 90 env -u HTTP_PROXY -u HTTPS_PROXY -u http_proxy -u https_proxy -u ALL_PROXY -u all_proxy \
  "$CODEX_BIN" exec --skip-git-repo-check "Reply with just: OK" < /dev/null 2>&1 | tail -12
```

**Expected on success:** a `codex` reply `OK` and a token-usage line. **Expected on failure:** `ERROR: Reconnecting... waiting for network` and rmcp `error sending request` lines.

**Branch:** fresh codex **succeeds** → the long-running process was poisoned → **F2**. Fresh codex **fails** → continue to B2.

### B2. Raw network probe (bypasses every proxy)

```bash
python << 'EOF'
import socket, ssl, http.client, time
print('DNS:', socket.getaddrinfo('chatgpt.com', 443, socket.AF_INET)[0][4])
for i in range(5):
    ctx = ssl.create_default_context(); s = socket.socket(); s.settimeout(10); t0 = time.time()
    try:
        s.connect(('chatgpt.com', 443))
        ss = ctx.wrap_socket(s, server_hostname='chatgpt.com'); ss.close()
        print(f'  TLS #{i+1} OK {time.time()-t0:.2f}s')
    except Exception as e:
        print(f'  TLS #{i+1} FAIL {time.time()-t0:.2f}s {type(e).__name__}')
conn = http.client.HTTPSConnection('chatgpt.com', timeout=15)
conn.request('POST', '/backend-api/codex/responses', body=b'{}',
             headers={'Content-Type': 'application/json'})
print('POST /responses ->', conn.getresponse().status)
EOF
```

**Expected:** DNS `198.18.x.x` in TUN mode; TLS OK; POST returns 401/403 (that means you reached the backend — it is rejecting your empty request, which is correct).

**Branch:**
- Probe fast and clean (<1s handshakes), fresh codex still fails → codex-specific config → **F6**.
- Probe succeeds but handshakes take 3–6s with occasional timeouts → degraded node → **F4**.
- Probe fails outright → **F5**.

---

## Phase C — Fixes

### F1. Stale `~/.codex/.env` proxy (most likely)

1. Backup, then remove:
   ```bash
   mv "$HOME/.codex/.env" "$HOME/.codex/.env.bak-$(date +%Y%m%d-%H%M%S)"
   ls -la "$HOME/.codex/".env*    # confirm only the .bak remains
   ```
2. The running codex process still holds the old env in memory → do **F2**.
3. **Rollback:** `mv` the `.bak` back to `.env` and redo F2.

### F2. Restart the codex process

The embedded `codex.exe` is a child of the ChatGPT desktop app and **respawns automatically within seconds** after being killed:

```bash
taskkill //PID <PID> //F          # Git Bash needs the double slashes
sleep 3 && tasklist | grep -i codex   # confirm a NEW codex.exe PID exists
```

(PowerShell equivalent: `Stop-Process -Id <PID> -Force`.) If the app does not respawn it or the Codex UI stays stuck, ask the user to fully quit and reopen the ChatGPT app. Config files are only read at process start — this restart is what applies the fix.

### F3. Zombie listener on the old proxy port

A half-dead process may still hold the port open (`LISTENING` exists, but CONNECT + TLS fails). If A3/A4 showed codex using such a port: fix is still F1 (stop codex from using it). Optionally identify the owner (`tasklist /FI "PID eq <pid>`") and let the user decide whether to kill that zombie. Do not kill unknown processes yourself.

### F4. Degraded VPN node

Evidence: handshakes succeed but take 3–6s, intermittent timeouts, codex sometimes works. This degrades UX but cannot cause a 100% failure with zero outbound connections — if you saw that, go back to F1.

Action: tell the user to **switch nodes**, then re-run the B2 benchmark after ~10s. Healthy = consistent handshakes under ~1s. Codex retries failed turns every 60s on its own; once the node is healthy it recovers without a restart (F2 only if the UI stays stuck).

### F5. VPN/TUN fully down

Evidence: raw probe fails, DNS returns real (non-198.18.x.x) IPs or fails. This is user action: reconnect the VPN / enable TUN. Afterwards re-run B2, then F2 if codex doesn't self-recover.

### F6. Provider config problem

Evidence: A1 malformed, or transport in logs is not what the user configured. Re-verify with the A1 script; if the user's goal is disabling WebSocket transport, apply the `codex-disable-websocket` skill (HTTP-only provider table, top-level `model_provider` switched, original provider preserved for rollback). If the config is ambiguous (duplicate tables, conflicting providers), stop after backing up and report the exact ambiguity — do not guess.

---

## Final verification checklist (all three, or the fix is not done)

1. **Fresh run:** the B1 command returns a model reply (`OK` + token usage).
2. **Log evidence:** new entries contain `completed remote installed plugin bundle sync` (previously `failed remote installed plugin bundle sync`).
3. **No recurrence:** zero new `error sending request` entries after the new process start time — compute that cutoff with `datetime.fromtimestamp`, never by hand.

```bash
python -c "
import sqlite3, os, datetime
con = sqlite3.connect(os.path.expanduser('~/.codex/logs_2.sqlite'))
newest = con.execute('SELECT MAX(ts) FROM logs').fetchone()[0]
print('newest log:', datetime.datetime.fromtimestamp(newest))
print('recent errors:', con.execute(\"SELECT COUNT(*) FROM logs WHERE feedback_log_body LIKE '%error sending request%' AND ts > ?\", (newest-300,)).fetchone()[0])
"
```

## Report template

Tell the user, in this order: root cause in one sentence; changed files and their backup paths; verification evidence (probe output / log lines, no secrets); whether any restart is still required; residual risks (e.g. current VPN node quality, or the `.env` backup they may want to delete later).

## Field-tested gotchas (read once, save an hour)

- `codex exec` without `< /dev/null` hangs forever on stdin.
- Agent sandboxes inject their own `HTTP_PROXY` — every codex probe needs the `env -u` prefix.
- `reg.exe` is blacklisted in many sandboxes — use Python `winreg`.
- `taskkill` in Git Bash requires `//PID <pid> //F` (double slashes), or it parses the flags as paths.
- The logs DB `ts` column is unix seconds; converting timestamps in your head produces confidently wrong cutoffs.
- A port can be `LISTENING` and still broken (zombie). Test through it, don't just check it exists.
- "Chat works but Codex doesn't" on the ChatGPT desktop app = Chromium vs Rust networking failing independently. It is a routing clue, never proof the network is fine.

