<p align="center">
  <img src="https://cdn.simpleicons.org/cloudflare/F38020?viewbox=auto&size=68" alt="Cloudflare logo" height="68" />
  <span>&nbsp;+&nbsp;</span>
  <img src="https://camo.githubusercontent.com/ede9710f2920f243f0e56cb036684fff6fef9c0a174ea5bb92109e5ef72c3812/68747470733a2f2f726177322e736561646e2e696f2f657468657265756d2f3078356565333632383636303031363133303933333631656238353639643539633431343162373664312f3766613963653930306662333962343432323633343864623333306533322f38623766613963653930306662333962343432323633343864623333306533322e737667" alt="Xray logo" height="68" />
</p>

<p align="center">
  <a href="https://workers.cloudflare.com/">
    <img src="https://img.shields.io/badge/Cloudflare-Workers-F38020?logo=cloudflare&logoColor=white" alt="Cloudflare Workers" />
  </a>
  <a href="https://www.typescriptlang.org/">
    <img src="https://img.shields.io/badge/TypeScript-Strict-3178C6?logo=typescript&logoColor=white" alt="TypeScript Strict" />
  </a>
  <a href="/.github/workflows/deploy.yml">
    <img src="https://img.shields.io/github/actions/workflow/status/YrustPd/cf-xray-proxy/deploy.yml?branch=main&label=deploy" alt="Deploy" />
  </a>
  <a href="/LICENSE">
    <img src="https://img.shields.io/badge/License-MIT-green.svg" alt="MIT License" />
  </a>
</p>

# cf-xray-proxy

Cloudflare Worker reverse-proxy frontend for VLESS/VMess traffic, forwarding `ws`, `xhttp`, and `httpupgrade` requests to an Xray or sing-box backend.

## What this project is

This repository provides a Worker entrypoint (`src/index.ts`) plus transport handlers (`src/transports/*`) that:

- accept inbound HTTP/Upgrade requests at Cloudflare edge,
- select a transport handler (`ws`, `xhttp`, or `httpupgrade`),
- forward path/query to backend as-is,
- bridge upgraded sockets between client and backend.

The backend remains the protocol/authentication authority.

## Why you would use it

- Put Cloudflare edge in front of an existing Xray/sing-box backend.
- Terminate TLS at the edge while keeping origin/backend on plain HTTP.
- Select transports per request via query/header/path without redeploying.
- Keep Worker logic thin and backend-focused for VLESS/VMess validation and policy.

## Features

- Multi-backend support with weighted selection and automatic failover.
- Periodic backend health checking with auto-recovery.
- Exponential backoff retry with jitter for backend retries.
- Connection-based rate limiting (per-IP concurrent and per-minute attempts).
- UUID-based maximum active connection limiting.
- Optional subscription proxy (`/sub/...`) with in-memory caching.
- Built-in observability endpoints: `GET /health` and `GET /status` (when `DEBUG=true`).

## Architecture

```text
Client (VLESS / VMess)
        |
        | HTTPS / TLS
        v
Cloudflare Worker (this repo)
  - Router / transport selection
  - BackendManager (weights, health checks, failover)
  - RateLimiter (connection-based, per IP)
  - UUIDManager (per-UUID active connection cap)
  - SubscriptionProxy (optional `/sub` routes)
        |
        | HTTP or HTTPS (BACKEND_URL / BACKEND_LIST)
        v
Backend pool (Xray / sing-box)
  - backend-1
  - backend-2
  - backend-N
  - authentication
  - protocol validation
  - routing / outbound
```

> TLS terminates at Cloudflare Worker edge. `BACKEND_URL` and each `BACKEND_LIST` entry can be `http://...` or `https://...`.

## Supported transports

| Transport | Handler file | Upgrade detection | Notes |
| --- | --- | --- | --- |
| `ws` | `src/transports/ws.ts` | `Connection: upgrade` + `Upgrade: websocket` | WebSocket upgrade + passthrough fallback |
| `xhttp` | `src/transports/xhttp.ts` | `Connection: upgrade` + `Upgrade: websocket` | Supports `mode` (`auto`/`packet-up`) and `ed` hint |
| `httpupgrade` | `src/transports/httpupgrade.ts` | `Connection: upgrade` + any `Upgrade` value | HTTP Upgrade semantics with shared WS bridging |

### Transport selection order

Selection logic is implemented in `src/index.ts`:

1. Query parameter `transport` (`xhttp`, `httpupgrade`, `ws`)
2. Header `x-transport-type`
3. Path prefix (`/xhttp/...`, `/httpupgrade/...`, `/ws/...`)
4. Environment/default transport (`TRANSPORT`, otherwise default `xhttp`)

## Routing behavior

- Path and query are forwarded exactly from inbound request to backend URL.
- Worker does not inject fixed paths.
- Worker strips transport prefix only when that same prefix selected routing:
  - `/ws/<path>` -> `/<path>`
  - `/xhttp/<path>` -> `/<path>`
  - `/httpupgrade/<path>` -> `/<path>`
- Worker-only routing selectors are removed before backend forward:
  - query `transport`
  - header `x-transport-type`
- Worker does not validate UUID, port, or path.

> Authentication, UUID checks, and policy enforcement belong on backend Xray/sing-box.

## Configuration

### Runtime variables and defaults

| Name | Required | Default | Description | Examples |
| --- | --- | --- | --- | --- |
| `BACKEND_URL` | No | `http://127.0.0.1:10000` | Backward-compatible single backend origin | `http://127.0.0.1:10000` |
| `BACKEND_LIST` | No | `http://127.0.0.1:10000` | Comma-separated backend list, supports optional weights via `url\|weight` | `http://be1:10000\|2,http://be2:10000\|1` |
| `BACKEND_HEALTH_CHECK_INTERVAL` | No | `30000` | Backend health check interval in milliseconds | `10000`, `30000` |
| `BACKEND_STICKY_SESSION` | No | `false` | When `true`, prefer first healthy backend in list order; when `false`, use weighted random | `true`, `false` |
| `MAX_RETRIES` | No | `3` | Maximum retry attempts for backend failover/retry paths | `1`, `3`, `5` |
| `RATE_LIMIT_ENABLED` | No | `false` | Enables connection-based per-IP rate limiting | `true`, `false` |
| `RATE_LIMIT_MAX_CONN_PER_IP` | No | `5` | Maximum concurrent upgraded connections per IP | `3`, `5`, `20` |
| `RATE_LIMIT_MAX_CONN_PER_MIN` | No | `10` | Maximum new upgraded connections per IP per minute | `10`, `30`, `60` |
| `UUID_MAX_CONNECTIONS` | No | `0` | Maximum active connections per UUID (`0` disables feature) | `0`, `1`, `2` |
| `SUBSCRIPTION_ENABLED` | No | `false` | Enables subscription proxy routes | `true`, `false` |
| `SUBSCRIPTION_PRESERVE_DOMAIN` | No | `false` | Rewrites upstream subscription domains back to configured target domain | `true`, `false` |
| `SUBSCRIPTION_TARGETS` | No | empty | Subscription backend mapping, JSON array or `name\|url\|port\|path` format | `phone\|https://phonepanel.ir\|443\|/sub,xui\|https://sub.xui.com:2096\|2096\|/sub` |
| `SUBSCRIPTION_TRANSFORM` | No | `false` | Enables response link transformation for subscription proxy responses | `true`, `false` |
| `SUBSCRIPTION_CACHE_TTL_MS` | No | `300000` | In-memory subscription cache TTL in milliseconds | `60000`, `300000` |
| `TRANSPORT` | No | `xhttp` | Default transport when no query/header/path selector matches | `xhttp`, `httpupgrade`, `ws` |
| `DEBUG` | No | `false` | Enables debug logs and `GET /status` endpoint | `true`, `false` |
| `BACKEND_ORIGIN` (code constant) | No (not an env var) | `http://127.0.0.1:10000` | Fallback backend origin defined in `src/config.ts` | `http://127.0.0.1:10000` |

### Set variables for local `wrangler dev`

Option A: one command invocation

```bash
BACKEND_LIST="http://127.0.0.1:10000|2,http://127.0.0.1:10001|1" \
BACKEND_HEALTH_CHECK_INTERVAL="30000" \
BACKEND_STICKY_SESSION="false" \
MAX_RETRIES="3" \
RATE_LIMIT_ENABLED="true" \
RATE_LIMIT_MAX_CONN_PER_IP="5" \
RATE_LIMIT_MAX_CONN_PER_MIN="10" \
UUID_MAX_CONNECTIONS="2" \
SUBSCRIPTION_ENABLED="false" \
SUBSCRIPTION_PRESERVE_DOMAIN="false" \
TRANSPORT="xhttp" \
DEBUG="true" \
wrangler dev
```

Option B: keep defaults in `wrangler.toml` under `[vars]` and run:

```bash
wrangler dev
```

### Set variables for `wrangler deploy`

1. Edit `wrangler.toml`:

```toml
[vars]
BACKEND_LIST = "http://127.0.0.1:10000|2,http://127.0.0.1:10001|1"
BACKEND_HEALTH_CHECK_INTERVAL = "30000"
BACKEND_STICKY_SESSION = "false"
MAX_RETRIES = "3"
RATE_LIMIT_ENABLED = "true"
RATE_LIMIT_MAX_CONN_PER_IP = "5"
RATE_LIMIT_MAX_CONN_PER_MIN = "10"
UUID_MAX_CONNECTIONS = "2"
SUBSCRIPTION_ENABLED = "false"
SUBSCRIPTION_PRESERVE_DOMAIN = "false"
TRANSPORT = "xhttp"
DEBUG = "false"
# BACKEND_URL remains supported for single-backend compatibility
# SUBSCRIPTION_TARGETS = "phone|https://phonepanel.ir|443|/sub,xui|https://sub.xui.com:2096|2096|/sub"
```

2. Deploy:

```bash
wrangler deploy
```

### Set variables in Cloudflare Dashboard

1. Open **Workers & Pages**.
2. Create/select this Worker.
3. Go to **Settings** -> **Variables and Secrets**.
4. Add variables:
   - `BACKEND_URL`
   - `BACKEND_LIST`
   - `BACKEND_HEALTH_CHECK_INTERVAL`
   - `BACKEND_STICKY_SESSION`
   - `MAX_RETRIES`
   - `RATE_LIMIT_ENABLED`
   - `RATE_LIMIT_MAX_CONN_PER_IP`
   - `RATE_LIMIT_MAX_CONN_PER_MIN`
   - `UUID_MAX_CONNECTIONS`
   - `SUBSCRIPTION_ENABLED`
   - `SUBSCRIPTION_PRESERVE_DOMAIN`
   - `SUBSCRIPTION_TARGETS`
   - `TRANSPORT`
   - `DEBUG`
5. Save and deploy.

## Multi-Backend Setup

### Configure multiple backends

Use `BACKEND_LIST` as comma-separated entries:

- Format: `url` or `url|weight`
- Weight defaults to `1` when omitted
- `BACKEND_URL` still works and is kept for backward compatibility

```bash
BACKEND_LIST="http://be1.internal:10000|3,http://be2.internal:10000|1,http://be3.internal:10000"
```

### Load balancing and failover behavior

- Backend selection uses weighted random distribution by default.
- Set `BACKEND_STICKY_SESSION=true` to prefer first healthy backend in list order.
- Unhealthy backends are skipped while healthy backends are available.
- On backend failure, the Worker retries using another backend up to `MAX_RETRIES`.
- Retry waits use exponential backoff with jitter to avoid synchronized retry bursts.
- Health checks run every `BACKEND_HEALTH_CHECK_INTERVAL` and auto-recover backends when they respond again.
- If all backends are unhealthy, the manager falls back to any available backend to avoid total blackhole behavior.

## Rate Limiting

### How it works

- Limiting is connection-based, not packet/message-based.
- It applies to new upgrade attempts only.
- Two limits are enforced per IP:
  - concurrent upgraded connections (`RATE_LIMIT_MAX_CONN_PER_IP`)
  - new upgraded connections per minute (`RATE_LIMIT_MAX_CONN_PER_MIN`)
- Blocked requests return `429 Too Many Requests` with `Retry-After`.

### Why it does not break VPN traffic

- No message count limits.
- No bandwidth throttling.
- Long-lived connections remain allowed.
- Disabled by default (`RATE_LIMIT_ENABLED=false`) for minimal overhead when not needed.

### Example

```toml
[vars]
RATE_LIMIT_ENABLED = "true"
RATE_LIMIT_MAX_CONN_PER_IP = "5"
RATE_LIMIT_MAX_CONN_PER_MIN = "10"
```

## Subscription Proxy

### When to use it

Enable this when you want to serve subscription endpoints through the Worker without exposing subscription backends directly.

### How to enable

```toml
[vars]
SUBSCRIPTION_ENABLED = "true"
SUBSCRIPTION_TARGETS = "phone|https://phonepanel.ir|443|/sub,xui|https://sub.xui.com:2096|2096|/sub,node|http://10.0.0.8|4005|/subscribe"
# Optional:
# SUBSCRIPTION_TRANSFORM = "true"
# SUBSCRIPTION_CACHE_TTL_MS = "300000"
```

### Supported target formats

Delimited format:

```text
name|url|port|path,name2|url2|port2|path2
```

JSON format:

```json
[
  { "name": "phone", "url": "https://phonepanel.ir", "port": 443, "path": "/sub" },
  { "name": "xui", "url": "https://sub.xui.com:2096", "port": 2096, "path": "/sub" }
]
```

### Routes

- `/sub/:token` -> uses default/fallback target selection.
- `/:service/sub/:token` -> uses named target (falls back to first configured target if name is not found).

The proxy enforces:

- upstream timeout: 10 seconds
- max upstream response size: 10 MB
- cache: in-memory, successful (`200`) responses only

When disabled (`SUBSCRIPTION_ENABLED=false`), subscription routing is bypassed for minimal impact.

## Quickstart

### 1) Install dependencies

```bash
npm install
```

### 2) Run local Worker

```bash
BACKEND_URL="http://127.0.0.1:10000" TRANSPORT="xhttp" DEBUG="true" wrangler dev
```

### 3) Minimal checks

Replace placeholders:

- `<worker-domain>`: your Worker URL (for local dev typically `127.0.0.1:8787`)
- `<path>`: backend inbound path

HTTP passthrough:

```bash
curl -i "http://<worker-domain>/<path>?check=1"
```

`ws` upgrade handshake:

```bash
curl -i --http1.1 \
  -H "Connection: Upgrade" \
  -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Version: 13" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  "http://<worker-domain>/ws/<path>"
```

`xhttp` upgrade handshake:

```bash
curl -i --http1.1 \
  -H "Connection: Upgrade" \
  -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Version: 13" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  "http://<worker-domain>/xhttp/<path>?mode=auto&ed=0"
```

`httpupgrade` upgrade handshake:

```bash
curl -i --http1.1 \
  -H "Connection: Upgrade" \
  -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Version: 13" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  "http://<worker-domain>/httpupgrade/<path>"
```

Expected handshake result: `HTTP/1.1 101 Switching Protocols` when backend accepts upgrade.

## Deployment guide

### Wrangler CLI deployment

1. Authenticate:

```bash
wrangler login
```

2. Confirm Worker settings in `wrangler.toml`:
   - `name`
   - `main`
   - `compatibility_date`
   - `[vars]` values (backend pool, retries, limits, transport, and debug settings)

3. Deploy:

```bash
wrangler deploy
```

4. Verify:
   - open deployed Worker URL in browser for landing page (`/`),
   - run the quickstart `curl` checks against deployed domain.

### Cloudflare Dashboard deployment

1. Go to **Workers & Pages** and create/import Worker.
2. Ensure main script entry maps to this repository Worker.
3. In **Settings** -> **Variables and Secrets**, add runtime vars.
4. Deploy from dashboard.
5. Test:
   - `GET /` for landing page,
   - transport checks (`/ws/<path>`, `/xhttp/<path>`, `/httpupgrade/<path>`).

## Troubleshooting

### 502 backend unreachable

Check backend listener on origin host:

```bash
ss -ltnp | grep -E '(:10000|:443|:80)'
lsof -iTCP -sTCP:LISTEN -n -P
```

Check direct backend response:

```bash
curl -i --http1.1 "http://<backend-host>:<backend-port>/<path>"
```

If direct backend fails, fix backend binding/firewall/path first.

### Upgrade not returning 101

Test backend upgrade directly:

```bash
curl -i --http1.1 \
  -H "Connection: Upgrade" \
  -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Version: 13" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  "http://<backend-host>:<backend-port>/<path>"
```

If backend direct returns non-101, Worker will also fail upgrade.

### Host/SNI/path mismatch pitfalls

| Field | Where to set | Must align with |
| --- | --- | --- |
| Host | Client URI/headers | Worker domain and backend expectations |
| SNI | Client TLS config | Worker certificate domain |
| Path | Client `path` | Backend inbound path |
| Transport type | Client config | Worker route selection and backend inbound type |

Use matching transport on both client and backend (`ws`, `xhttp`, `httpupgrade`).

### Debug mode

Enable debug:

```bash
DEBUG="true" wrangler dev
```

Tail deployed logs:

```bash
wrangler tail
```

Look for handler prefixes:

- `[cf-xray-proxy]` (router)
- `[ws]`
- `[xhttp]`
- `[httpupgrade]`

## Security considerations

- This Worker forwards traffic and manages upgrades; backend Xray/sing-box remains the authority for authentication and protocol policy.
- Enable `RATE_LIMIT_ENABLED=true` to reduce abusive connection churn without limiting normal tunnel payload traffic.
- Use `UUID_MAX_CONNECTIONS` to cap concurrent usage per UUID (`0` keeps the feature disabled).
- Prefer private/internal backend addresses and restrict backend ingress to expected sources (Cloudflare egress or trusted networks).
- Use `BACKEND_LIST` to isolate failure domains across multiple backend instances.
- Keep `DEBUG=false` in normal production operation to reduce log exposure.

## Landing page

When `SUBSCRIPTION_ENABLED=false`:

- `GET /` and `GET /index.html` document requests are served by `src/landing.ts` with cache header:

```text
Cache-Control: public, max-age=3600
```

When `SUBSCRIPTION_ENABLED=true`:

- root requests return subscription info text instead of the HTML landing page.

## License

[MIT](/LICENSE)
