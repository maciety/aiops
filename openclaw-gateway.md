# OpenClaw Gateway — Troubleshooting Notes

Gateway runs as a user systemd service on `ollama` host (port 18789).
Web UI: https://openclaw.moleda.io, proxied through Nginx Proxy Manager (NPM) running on the TrueNAS NAS.
Config: `/root/.openclaw/openclaw.json`

`ollama` is an Incus container running on the TrueNAS host; OpenClaw is installed inside that container.

---

## Fix: "origin not allowed"

Add the UI's origin to `gateway.controlUi.allowedOrigins` in `/root/.openclaw/openclaw.json`:

```json
"controlUi": {
  "allowedOrigins": [
    "http://localhost:18789",
    "http://127.0.0.1:18789",
    "https://openclaw.moleda.io"
  ]
}
```

Then restart: `systemctl --user kill -s SIGUSR1 openclaw-gateway`

---

## Fix: "Proxy headers detected from untrusted address"

OpenClaw is proxied through Nginx Proxy Manager (NPM), not Traefik. NPM runs as a TrueNAS app on bridged networking at `192.168.50.12` and uses ports `80` and `443`. Add the NPM app IP to `gateway.trustedProxies` in `/root/.openclaw/openclaw.json`:

```json
"gateway": {
  "trustedProxies": ["192.168.50.12"]
}
```

Then restart: `systemctl --user kill -s SIGUSR1 openclaw-gateway`

Known correction: `192.168.50.12` is the NPM TrueNAS app IP for OpenClaw, not Traefik. OpenClaw's proxy path is NAS/TrueNAS NPM app → `ollama` Incus container → OpenClaw gateway.

---

## Fix: Device pairing deadlock

**Symptom:** Browser loops on "device pairing required" with a new requestId each reload.
Running `openclaw devices approve <requestId>` returns "missing scope: operator.admin".

**Root cause:** `approveDevicePairing()` enforces that the *caller's* scopes must cover all scopes being *granted*. The browser requests `operator.admin`, so the CLI approver must also have `operator.admin` in its active scopes. If the CLI was approved with limited scopes (e.g. only `operator.pairing`), this deadlocks.

**Fix:** Call `approveDevicePairing` directly from Node.js, bypassing the gateway WebSocket:

```javascript
// node --input-type=module (run on ollama host)
import { n as approveDevicePairing } from '/usr/lib/node_modules/openclaw/dist/device-pairing-CMJxtqB1.js';
const result = await approveDevicePairing('<requestId>', {
  callerScopes: ['operator.admin', 'operator.read', 'operator.write', 'operator.approvals', 'operator.pairing']
});
console.log(JSON.stringify(result));
process.exit(0);
```

Get the current requestId from `/root/.openclaw/devices/pending.json`.
After approval: `systemctl --user kill -s SIGUSR1 openclaw-gateway`

**Key files on ollama host:**
- `/root/.openclaw/devices/paired.json` — approved devices and their scopes
- `/root/.openclaw/devices/pending.json` — pending pairing requests (contains current requestId)
- `/root/.openclaw/identity/device-auth.json` — CLI's stored device token and scopes
- `/usr/lib/node_modules/openclaw/dist/device-pairing-CMJxtqB1.js` — `approveDevicePairing` impl
