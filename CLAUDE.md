# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

edgetunnel is a Cloudflare Workers/Pages edge-compute proxy that supports VLESS, Trojan, and Shadowsocks protocols over WebSocket, gRPC, and XHTTP transports. It includes an admin panel, subscription generation for mainstream clients (Clash, Sing-box, Surge, etc.), and chain-proxy capabilities (SOCKS5, HTTP, HTTPS, TURN, SSTP). The entire application is a **single file** — `_worker.js` (~5900 lines) — with no build step, no package.json, and no dependencies.

## Development

There is no build system, linter, or test suite. The file is deployed directly to Cloudflare Workers or Pages.

- **Local dev**: Use [Wrangler](https://developers.cloudflare.com/workers/wrangler/) (`npx wrangler dev`) with `wrangler.toml`.
- **Deploy**: Copy `_worker.js` into the Cloudflare Workers editor, or upload the repo as a Cloudflare Pages project. A KV namespace bound as `KV` is required for persistence.
- **Testing proxy connectivity**: Hit `GET /admin/check?socks5=...` (or `http`, `https`, `turn`, `sstp`) from the deployed worker to validate chain proxies.

## Architecture

`_worker.js` is structured as a single Cloudflare Worker `export default { async fetch(request, env, ctx) }`. Dispatch happens inline by path and method:

| Route pattern | Handler |
|---|---|
| `/?version` (with valid UUID) | Returns version info |
| WebSocket Upgrade (any path, valid admin) | `处理WS请求()` — WS tunneling |
| POST (any path, valid admin) | `处理gRPC请求()` (content-type `application/grpc`) or `处理XHTTP请求()` (x_padding referer) |
| `/login` | Login page / POST form auth; sets `auth` cookie |
| `/admin` and `/admin/*` | Admin APIs (config CRUD, logs, Cloudflare usage, proxy check, custom IPs). Admin UI HTML is served from `Pages静态页面` (`https://edt-pages.github.io`). |
| `/sub` | Subscription generation (mixed/base64, Clash YAML, Sing-box JSON, Surge). Supports both local generation from a preferred-IP pool and delegating to an external subconverter. |
| `/logout` or UUID-format path | Clears auth cookie, redirects to login |
| `/KEY` (KEY env var value) | Quick subscription redirect |
| Everything else | Camouflage page (reverse proxy to `URL` env var, or built-in nginx/1101 error page) |

### Key subsystems (in file order)

1. **Configuration & initialization** (lines 1–45): Reads env vars (`ADMIN`, `KEY`, `UUID`, `PROXYIP`, `GO2SOCKS5`, `DEBUG`, `BEST_SUB`, `PRELOAD_RACE_DIAL`, `OFF_LOG`), derives `userID` UUID from `MD5MD5(ADMIN + KEY)`, sets proxy IP (custom or CF-colo-based default).

2. **Proxy transports** (lines 513–1540+):
   - `处理XHTTP请求()` — Xray-core XHTTP stream-one transport
   - `处理gRPC请求()` — gRPC-web tunneling
   - `处理WS请求()` — WebSocket tunneling with early-data decoding
   - All three funnel into shared TCP/UDP forwarding (`forwardataTCP`, `forwardataudp`)

3. **TCP connection & upstream** (lines 1855–2470):
   - `forwardataTCP()` — connects to origin, supports SOCKS5/HTTP/HTTPS/TURN/SSTP chain proxies, WS upload queue with backpressure (`创建上行写入队列`), downstream Grain-based framing (`创建下行Grain发送器`)
   - `创建请求TCP连接器()` — creates raw TCP sockets via `connect()` (CF Workers runtime API)

4. **Chain proxy clients** (lines 2487–4120):
   - `socks5Connect()` — SOCKS5 handshake (no-auth and user/password)
   - `httpConnect()` / `httpsConnect()` — HTTP CONNECT tunnel
   - `turnConnect()` — TURN (Traversal Using Relays around NAT) with STUN auth
   - `sstpConnect()` — SSTP (Secure Socket Tunneling Protocol) over SOCKS5

5. **Custom TLS stack** (lines 2879–3360): Pure-JS TLS 1.2/1.3 implementation (`TlsClient` class is instantiated inline). Includes ChaCha20-Poly1305, AES-GCM, ECDH key exchange (P-256), HKDF, TLS 1.2 PRF. Needed because Cloudflare Workers do not expose raw TLS sockets — only `connect()` gives a raw TCP socket that requires application-layer TLS.

6. **Subscription generation** (lines 285–477, 4136–4680):
   - Detects target format from UA/query params: `mixed` (base64 node list), `clash` (YAML), `singbox` (JSON), `surge`, `quanx`, `loon`
   - `Clash订阅配置文件热补丁()` — injects DNS config, ECH opts, gRPC user-agent into Clash YAML
   - `Singbox订阅配置文件热补丁()` — similar patching for Sing-box JSON
   - `Surge订阅配置文件热补丁()` — Surge-specific policy path rewriting

7. **Admin APIs** (lines 66–280):
   - `GET /admin/config.json` — current config
   - `POST /admin/config.json` — save config (validates UUID + HOST required)
   - `POST /admin/cf.json` — Cloudflare API credentials
   - `POST /admin/tg.json` — Telegram bot credentials
   - `POST /admin/ADD.txt` — custom preferred IPs
   - `GET /admin/log.json` — traffic logs from KV
   - `GET /admin/getCloudflareUsage` — CF account usage stats
   - `GET /admin/getADDAPI` — validate preferred-IP API endpoint
   - `GET /admin/check` — proxy connectivity test
   - `GET /admin/init` — reset config to defaults

8. **Utility functions** (lines 4690–5905):
   - `读取config_JSON()` — reads/initializes config from KV with extensive defaults
   - `请求优选API()` — fetches preferred IPs from external APIs with timeout
   - `生成随机IP()` — generates randomized Cloudflare IP ranges
   - `反代参数获取()` — parses proxyip/socks5/http path params
   - `DoH查询()` — DNS-over-HTTPS for A/AAAA records (used for preload race dial)
   - `解析地址端口()` — resolves proxyip hostname to IP:port
   - `base64SecretEncode/Decode()` — XOR-based obfuscated base64
   - `html1101()` / `nginx()` — built-in camouflage pages
   - `请求日志记录()` — KV-backed traffic/request logging (respects `OFF_LOG`)

### Naming convention

The codebase uses **Chinese identifiers** throughout (variables, function names, comments). This is intentional and should be preserved. Key Chinese terms:
- `反代` = reverse proxy
- `优选` = preferred/optimized
- `订阅` = subscription
- `请求` = request
- `处理` = handle/process
- `路径` = path
- `节点` = node
- `配置` = config
- `日志` = log
- `伪装` = camouflage
- `传输协议` = transport protocol
- `加密方式` = encryption method

### Config persistence

All configuration is stored in Cloudflare KV (binding `KV`):
- `config.json` — main proxy config (UUID, HOST, transport, TLS settings, subscription preferences, etc.)
- `cf.json` — Cloudflare API credentials for usage stats
- `tg.json` — Telegram bot credentials
- `ADD.txt` — custom preferred IP list
- `log.json` — traffic log array

### Environment variables (see README for full table)

Critical: `ADMIN` (required), `KEY`, `UUID`, `PROXYIP`, `URL`, `GO2SOCKS5`, `DEBUG`, `OFF_LOG`, `BEST_SUB`, `PRELOAD_RACE_DIAL`. The `KV` binding must exist for admin panel and subscription functionality.

## GitHub workflows

- `.github/workflows/sync.yml` — daily upstream sync for forks.
- `.github/workflows/Auto-close-empty-PRs.yml` — auto-closes empty PRs.
