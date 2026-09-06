# llm-proxy — stateless mTLS reverse proxy for Anthropic / OpenAI / DeepSeek

A minimal reverse proxy that sits between a mobile agent and the LLM
providers. The proxy holds **no credentials**: every provider API key lives
only on the mobile device and travels inside each request. Devices
authenticate to the proxy with **mTLS client certificates**, so stolen
proxy hardware or a leaked proxy image reveals nothing.

```
┌────────────────┐   mTLS (client cert)    ┌───────────┐   TLS    ┌──────────────────┐
│  Mobile agent   │ ──────────────────────▶ │ llm-proxy │ ───────▶ │ api.anthropic.com │
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

| Prefix        | Upstream                    |
|---------------|-----------------------------|
| `/anthropic/…`| `https://api.anthropic.com/…` |
| `/openai/…`   | `https://api.openai.com/…`    |
| `/deepseek/…` | `https://api.deepseek.com/…`  |

The prefix is stripped; everything else — method, query string, headers
(minus hop-by-hop), body — is forwarded verbatim, and responses stream
back chunk-by-chunk, so SSE (`"stream": true`) works end to end.

## Security properties

- **No credentials at rest on the proxy.** The proxy's disk holds only its
  own TLS keypair and the CA *public* certificate. Provider keys exist in
  memory for the lifetime of a request.
- **mTLS at the handshake.** A connection without a valid client
  certificate signed by your CA is dropped before any HTTP is spoken.
- **No key logging.** Access logs record method, path, status, and the
  device certificate's CN — never headers or bodies.
- **Revocation.** `generate-certs.sh revoke <device>` produces a CRL;
  point the proxy at it (`TLS_CRL` in Node; for Python/Go, regenerating
  the CA per fleet or short cert lifetimes is the simpler lever).
- Keep `ca.key` **offline** — it mints device identities. It is never
  needed on the proxy host.

## Quick start

```bash
# 1. Certificates
cd certs
./generate-certs.sh init proxy.example.com   # or the proxy's IP
./generate-certs.sh client my-phone          # creates client-my-phone.p12

# 2. Run one implementation
node node/server.js                          # Node
cd python && pip install -r requirements.txt && ./run.sh   # Python
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

## Mobile app integration

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
- **Bare process (systemd)** — `deploy/systemd/`: one hardened unit per
  implementation; comments at the top of each file give the install steps.
- **Serverless** — `deploy/serverless/`: Cloudflare Worker + wrangler
  config; mTLS is enforced at Cloudflare's edge. See
  `deploy/serverless/README-serverless.md`, including the AWS/GCP notes.

## Adding a provider

Add one line to the `UPSTREAMS` map in whichever implementation you run
(and in `worker.js` for serverless). Nothing else changes.
