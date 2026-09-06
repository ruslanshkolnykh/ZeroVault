# ZeroVault — stateless mTLS reverse proxy for 21 AI providers

A minimal reverse proxy that sits between an AI/mobile agent and the LLM
providers. The proxy holds **no credentials**: every provider API key lives
only on the agent's device and travels inside each request. Devices
authenticate to the proxy with **mTLS client certificates**, so stolen
proxy hardware or a leaked proxy image reveals nothing.

```
┌────────────────┐   mTLS (client cert)    ┌───────────┐   TLS    ┌──────────────────┐
│ AI/Mobile agent │ ──────────────────────▶ │ llm-proxy │ ───────▶ │ api.anthropic.com │
│  holds:         │  /anthropic/v1/messages │           │          │ api.openai.com    │
│  • client cert  │  x-api-key: sk-ant-...  │ stateless │          │ api.deepseek.com  │
│  • API keys     │  (key forwarded as-is)  │ no keys   │          └──────────────────┘
└────────────────┘                          └───────────┘
```

Three interchangeable implementations (pick one): `node/` (zero
dependencies), `python/` (FastAPI + httpx), `go/` (stdlib only). All three
behave identically: same routing, same mTLS posture, same streaming
behavior.

## Routing

21 providers, verified against each provider's live API documentation
(2026-09). The prefix is stripped and the rest of the path is forwarded
verbatim, so the app sends the provider's documented path after the prefix.
Auth headers pass through untouched, so each provider's scheme just works.

**Major model labs**

| Prefix | Upstream | Auth the app sends | Typical path |
|---|---|---|---|
| `/anthropic/…` | `api.anthropic.com` | `x-api-key` | `/v1/messages` |
| `/openai/…` | `api.openai.com` | `Authorization: Bearer` | `/v1/chat/completions` |
| `/deepseek/…` | `api.deepseek.com` | `Authorization: Bearer` | `/v1/chat/completions` |
| `/gemini/…` | `generativelanguage.googleapis.com` | `x-goog-api-key` | `/v1beta/models/...:generateContent` |
| `/mistral/…` | `api.mistral.ai` | `Authorization: Bearer` | `/v1/chat/completions` |
| `/cohere/…` | `api.cohere.com` | `Authorization: Bearer` | `/v2/chat` |
| `/ai21/…` | `api.ai21.com` | `Authorization: Bearer` | `/studio/v1/chat/completions` |
| `/perplexity/…` | `api.perplexity.ai` | `Authorization: Bearer` | `/chat/completions` |
| `/stability/…` | `api.stability.ai` | `Authorization: Bearer` | `/v2beta/stable-image/generate/...` |

**China-based providers**

| Prefix | Upstream | Auth | Typical path |
|---|---|---|---|
| `/dashscope/…` | `dashscope-intl.aliyuncs.com` | `Authorization: Bearer` | `/compatible-mode/v1/chat/completions` |
| `/moonshot/…` | `api.moonshot.ai` | `Authorization: Bearer` | `/v1/chat/completions` |
| `/zhipu/…` | `open.bigmodel.cn` | `Authorization: Bearer` | `/api/paas/v4/chat/completions` |
| `/zai/…` | `api.z.ai` | `Authorization: Bearer` | `/api/paas/v4/chat/completions` |
| `/siliconflow/…` | `api.siliconflow.com` | `Authorization: Bearer` | `/v1/chat/completions` |

**Russia-based providers**

| Prefix | Upstream | Auth | Typical path |
|---|---|---|---|
| `/yandex/…` | `llm.api.cloud.yandex.net` | `Authorization: Bearer` (+ `OpenAI-Project: <folder_ID>`) | `/v1/chat/completions` (OpenAI-compatible) or `/foundationModels/v1/completion` (native, `Api-Key` auth) |
| `/gigachat/…` | `api.giga.chat` | `Authorization: Bearer` (30-min OAuth token) | `/v1/chat/completions` |
| `/gigachat-auth/…` | `ngw.devices.sberbank.ru:9443` | `Authorization: Basic <auth key>` | `/api/v2/oauth` (token exchange) |

**Embeddings, image & specialized**

| Prefix | Upstream | Auth | Typical path |
|---|---|---|---|
| `/voyage/…` | `api.voyageai.com` | `Authorization: Bearer` | `/v1/embeddings` |
| `/jina/…` | `api.jina.ai` | `Authorization: Bearer` | `/v1/embeddings` |
| `/nomic/…` | `api-atlas.nomic.ai` | `Authorization: Bearer` | `/v1/embedding/text` |
| `/recraft/…` | `external.api.recraft.ai` | `Authorization: Bearer` | `/v1/images/generations` |
| `/upstage/…` | `api.upstage.ai` | `Authorization: Bearer` | `/v1/chat/completions` |

Notes: Google PaLM and 01.AI/Lingyi were deliberately excluded (deprecated /
unverifiable API status). Alibaba also offers workspace-scoped regional hosts
(`{WorkspaceId}.ap-southeast-1.maas.aliyuncs.com` etc.) — add yours with
`EXTRA_UPSTREAMS` (below). Gemini also accepts the key as a `?key=` query
parameter; the proxy never logs query strings, so that stays safe.

GigaChat specifics: the app first exchanges its authorization key for a
30-minute access token via `/gigachat-auth/api/v2/oauth`, then calls
`/gigachat/v1/...` with that token — both hops go through the proxy
(`gigachat-auth` demonstrates non-443 upstream ports). Sber and some
Yandex endpoints serve certificates from the **Russian Trusted Root CA**,
which is not in standard trust stores — if the proxy logs `upstream error`
for these routes, install that root on the proxy host (Debian:
drop the PEM in `/usr/local/share/ca-certificates/` and run
`update-ca-certificates`; Node also honors `NODE_EXTRA_CA_CERTS`). This
affects only the proxy→provider hop; devices are untouched.

**Custom routes:** `EXTRA_UPSTREAMS="alias=host.example.com,other=host2"`
adds operator-defined prefixes at deploy time (all three implementations).
Clients can never add routes, so the gateway remains a closed allowlist.

Everything after the prefix — method, query string, headers (minus
hop-by-hop), body — is forwarded verbatim, and responses stream back
chunk-by-chunk, so SSE (`"stream": true`) works end to end.

## Security properties

Audited and hardened — see **SECURITY.md** for the full threat model,
findings, and residual risks. Highlights:

- **No credentials at rest on the proxy.** The proxy's disk holds only its
  own TLS keypair and the CA *public* certificate. Provider keys exist in
  memory for the lifetime of a request. Logs record method, query-stripped
  path, status and device CN — never headers, bodies or queries.
- **mTLS at the handshake.** A connection without a valid client
  certificate signed by your CA is dropped before any HTTP is spoken.
  TLS 1.3 by default; the optional TLS 1.2 fallback
  (`TLS_MIN_VERSION=TLSv1.2`) allows ECDHE+AEAD suites only, so all
  traffic has forward secrecy — recorded traffic can't be decrypted later.
- **Revocation & kill switch.** `generate-certs.sh revoke <device>`
  produces `crl.pem`; set `TLS_CRL` and restart — works in all three
  implementations. `ALLOWED_DEVICES=phone1,phone2` (Node/Go) is an
  instant application-layer allowlist on top.
- **Abuse containment.** Per-device token-bucket rate limiting
  (`RATE_LIMIT_RPM`, default 120; `RATE_LIMIT_BURST`, default 30 — keyed
  on cert CN in Node/Go, client IP in Python) and a streamed body cap
  (`MAX_BODY_BYTES`, default 25 MB) limit what a stolen cert can do
  before revocation. Set provider-side spend caps on every key as the
  backstop.
- **`ca.key` is AES-256-encrypted and never needed on the proxy host.**
  Keep it offline — it mints device identities.

## Quick start

```bash
# 1. Certificates
cd certs
./generate-certs.sh init proxy.example.com   # or the proxy's IP
./generate-certs.sh client my-phone          # creates client-my-phone.p12

# 2. Run one implementation
node node/server.js                          # Node
cd python && pip install -r requirements.txt && ./run.py   # Python
cd go && go run .                            # Go

# 3. Test with the device's cert
curl --cacert certs/ca.crt \
     --cert certs/client-my-phone.crt --key certs/client-my-phone.key \
     https://proxy.example.com:8443/anthropic/v1/messages \
     -H 'x-api-key: sk-ant-...' \
     -H 'anthropic-version: 2023-06-01' \
     -H 'content-type: application/json' \
     -d '{"model":"claude-sonnet-4-5","max_tokens":64,"messages":[{"role":"user","content":"hi"}]}'
```

OpenAI and DeepSeek go through the same way, with the app's usual
`Authorization: Bearer …` header:

```bash
curl --cacert certs/ca.crt --cert client.crt --key client.key \
     https://proxy.example.com:8443/openai/v1/chat/completions \
     -H 'Authorization: Bearer sk-...' -H 'content-type: application/json' \
     -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"hi"}]}'
```

## AI/Mobile agent integration

The app builds the exact request it would send to the provider, changes
only the base URL, and attaches its client certificate:

- **Base URLs** in the app's SDK/config:
  `https://proxy:8443/anthropic`, `https://proxy:8443/openai`,
  `https://proxy:8443/deepseek`. Official SDKs accept a custom base URL
  (`baseURL` in Anthropic's and OpenAI's SDKs), so no request-building
  code changes.
- **Android:** import `client-<device>.p12` into the app's own keystore
  (bundle at enrollment or fetch once over a trusted channel), then supply
  it via `SSLContext`/OkHttp:
  `KeyManagerFactory` over the PKCS#12 → `sslSocketFactory(...)`. Pin
  `ca.crt` as the trust anchor for the proxy connection.
- **iOS:** import the `.p12` with `SecPKCS12Import`, keep the identity in
  the Keychain, answer `URLSession`'s
  `NSURLAuthenticationMethodClientCertificate` challenge with a
  `URLCredential(identity:…)`, and trust `ca.crt` for the server side.
- Store provider API keys in the platform secure store (Android Keystore /
  iOS Keychain) and add them per request as `x-api-key` (Anthropic) or
  `Authorization: Bearer` (OpenAI, DeepSeek) — exactly as if talking to
  the provider directly.

## Deployment

Each option is self-contained under `deploy/`:

- **Docker** — `deploy/docker/`: one Dockerfile per implementation plus a
  compose file. Certs are bind-mounted read-only, never baked into images.
  `docker compose --profile go up -d` (or `node` / `python`).
- **Bare process (systemd)** — one command from scratch:
  `sudo deploy/install/install-{node,python,go}.sh` creates the service
  user, runtime, hardened layout and unit, then self-checks that mTLS is
  enforced. Units live in `deploy/systemd/`.
- **Serverless** — `deploy/serverless/`: Cloudflare Worker + wrangler
  config; mTLS is enforced at Cloudflare's edge. See
  `deploy/serverless/README-serverless.md`, including the AWS/GCP notes.

## CA Console (certificate management UI)

`tools/ca-console/` is a point-and-click interface for the certificate
lifecycle — enroll devices, download `.p12` bundles, one-click revoke,
expiry dashboard. It runs **on the CA machine only** (binds 127.0.0.1,
per-start access token) and wraps `generate-certs.sh`, so the CA key never
moves:

```bash
cd tools/ca-console
CA_KEY_PASS=<passphrase> node server.js   # open the printed URL
```

## Runbooks

Per-runtime operational guides — fresh install, certificate lifecycle
(enroll/revoke/rotate), configuration reference, updates and rollback,
monitoring, troubleshooting, disaster recovery:
`runbooks/RUNBOOK-node.md`, `runbooks/RUNBOOK-python.md`,
`runbooks/RUNBOOK-go.md`.

## Adding a provider

Set `EXTRA_UPSTREAMS="alias=api.host.com"` at deploy time, or add one line
to the `UPSTREAMS` map in whichever implementation you run (and in
`worker.js` for serverless). Nothing else changes.
