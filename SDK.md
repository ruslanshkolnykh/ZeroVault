# llm-proxy Developer SDK & Documentation

Client SDKs for connecting an AI/mobile agent to the llm-proxy gateway:
**JavaScript/TypeScript** (`sdk/js`), **Python** (`sdk/python`),
**Kotlin/Android** (`sdk/kotlin`), **Swift/iOS** (`sdk/swift`). All four
share the same design and never store provider API keys.

## 1. Concepts

Every request the SDK makes carries two credentials, in two different
layers, and the gateway holds neither:

1. **Device identity (TLS layer).** Your device's client certificate (from
   `certs/generate-certs.sh client <name>`) authenticates the connection
   itself. No valid certificate — no connection. The SDK also *pins* the
   proxy's CA (`ca.crt`), so it will refuse to talk to an impostor gateway.
2. **Provider API key (HTTP layer).** The key for Anthropic/OpenAI/etc.
   travels in the request headers, straight through the gateway to the
   provider. Store it in the platform secure store; hand it to the SDK per
   request or via a `keyProvider` callback so it's fetched lazily and never
   held by the SDK.

Routing is by URL prefix. The SDK's `baseUrl(provider)` returns
`https://<proxy>/<provider>`; everything after that prefix is the
provider's own documented path, forwarded verbatim:

```
client.baseUrl("openai")   -> https://proxy:8443/openai
POST /openai/v1/chat/completions  ==  POST https://api.openai.com/v1/chat/completions
```

The 19 built-in providers, their auth headers and typical paths are in the
README routing table. The SDKs apply the right auth header automatically:
`x-api-key` for Anthropic, `x-goog-api-key` for Gemini, `Bearer` for the
rest.

## 2. Enrollment (all platforms)

1. Operator runs `./generate-certs.sh client my-device` on the CA machine.
2. Deliver `client-my-device.p12` (identity) and `ca.crt` (trust anchor) to
   the device over a trusted channel; delete the local `.p12` copy.
3. The app imports the `.p12` into secure storage (Keychain / Keystore /
   files with restricted permissions on servers) and keeps `ca.crt` beside
   it.
4. Provider API keys go into the platform secret store, never into code.

## 3. Quickstarts

### JavaScript / TypeScript (Node 18+, zero dependencies)

```js
const fs = require('fs');
const { LlmProxyClient } = require('@llm-proxy/sdk'); // sdk/js

const client = new LlmProxyClient({
  proxyUrl: 'https://proxy.example.com:8443',
  cert: fs.readFileSync('client-my-device.crt'),
  key:  fs.readFileSync('client-my-device.key'),
  ca:   fs.readFileSync('ca.crt'),
  keyProvider: (provider) => mySecretStore.get(provider),  // optional
});

// Plain request (any provider, any endpoint)
const res = await client.request('anthropic', '/v1/messages', {
  headers: { 'anthropic-version': '2023-06-01' },
  json: { model: 'claude-sonnet-4-5', max_tokens: 200,
          messages: [{ role: 'user', content: 'hello' }] },
});
console.log(res.json().content);

// Streaming (SSE)
for await (const ev of client.stream('openai', '/v1/chat/completions', {
  json: { model: 'gpt-4o-mini', stream: true,
          messages: [{ role: 'user', content: 'hello' }] },
})) {
  if (ev.data !== '[DONE]') process.stdout.write(JSON.parse(ev.data).choices[0].delta.content ?? '');
}
```

### Python (3.9+, stdlib only)

```python
from llmproxy import LlmProxyClient  # sdk/python

client = LlmProxyClient(
    proxy_url="https://proxy.example.com:8443",
    cert="client-my-device.crt", key="client-my-device.key", ca="ca.crt",
)

r = client.request("mistral", "/v1/chat/completions", api_key=key,
                   json={"model": "mistral-small-latest",
                         "messages": [{"role": "user", "content": "hello"}]})
print(r.json()["choices"][0]["message"]["content"])

for ev in client.stream("deepseek", "/v1/chat/completions", api_key=key,
                        json={"model": "deepseek-chat", "stream": True,
                              "messages": [{"role": "user", "content": "hi"}]}):
    ...  # ev.data is the raw SSE payload
```

### Kotlin / Android (OkHttp)

```kotlin
val client = LlmProxyClient(
    proxyUrl = "https://proxy.example.com:8443",
    p12 = context.openFileInput("client-my-device.p12"),
    p12Password = enrollmentPassword,
    caCert = context.assets.open("ca.crt"),
    keyProvider = { provider -> encryptedPrefs.getString(provider, null)!! },
)

val response = client.request(
    provider = "zhipu",
    path = "/api/paas/v4/chat/completions",
    jsonBody = """{"model":"glm-4.6","messages":[{"role":"user","content":"你好"}]}""",
)
```

For SSE on Android, make the same call with `"stream": true` and read
`response.body!!.source()` line by line (lines starting `data:`), or plug
`client.httpClient` into okhttp-eventsource.

### Swift / iOS (URLSession)

```swift
let client = LlmProxyClient(
    proxyURL: URL(string: "https://proxy.example.com:8443")!,
    identity: enrolledIdentity,        // from SecPKCS12Import at enrollment
    caCertificate: pinnedCA,           // from bundled ca.crt
    keyProvider: { provider in Keychain.apiKey(for: provider) }
)

let (data, _) = try await client.request(
    provider: "anthropic", path: "/v1/messages",
    json: ["model": "claude-sonnet-4-5", "max_tokens": 200,
           "messages": [["role": "user", "content": "hello"]]],
    headers: ["anthropic-version": "2023-06-01"]
)
```

For SSE on iOS, build the same `URLRequest` and use
`URLSession.bytes(for:)`, iterating `lines` and parsing `data:` prefixes.

## 4. Using official provider SDKs

The proxy is transparent, so official SDKs work — give them the proxy base
URL and the mTLS transport:

**Node** (Anthropic / OpenAI SDKs accept `baseURL` + `httpAgent`):

```js
const Anthropic = require('@anthropic-ai/sdk');
const anthropic = new Anthropic({
  baseURL: client.baseUrl('anthropic'),
  apiKey: myKey,
  httpAgent: client.httpsAgent,          // carries the device certificate
});
```

**Python** (SDKs accept `base_url` + an `http_client`; needs `httpx`):

```python
import httpx, anthropic
http_client = httpx.Client(verify=client.ssl_context)   # mTLS-configured
anthropic_client = anthropic.Anthropic(
    base_url=client.base_url("anthropic"), api_key=my_key,
    http_client=http_client,
)
```

Gemini's REST paths work through `/gemini/v1beta/...` with the
`x-goog-api-key` header (the SDK sets it for you via `request()`); Google's
own client libraries vary in base-URL support, so prefer the SDK's raw
`request`/`stream` for Gemini.

## 5. Error reference

| Status | Source | Meaning | Handle by |
|---|---|---|---|
| — TLS handshake failure | proxy | cert missing/expired/revoked, or CA mismatch | re-enroll device; check clock |
| 400 `invalid path` | proxy | path contained `..`, `\`, or bad encoding | fix the path |
| 401/403 (JSON from provider) | upstream | bad/expired API key | rotate the key on the device |
| 403 `device not allowed` | proxy | CN not in `ALLOWED_DEVICES` | contact operator |
| 404 `unknown route` | proxy | unknown provider prefix | use a listed provider or `EXTRA_UPSTREAMS` |
| 413 | proxy | body over `MAX_BODY_BYTES` | shrink payload |
| 429 + `retry-after` | proxy | device rate limit | back off; honor `retry-after` |
| 502 `upstream error` | proxy | provider unreachable | retry with backoff |
| anything else | upstream | the provider's own error, passed through | per provider docs |

Both SDKs throw a typed error (`LlmProxyError`) carrying `status`, `body`,
`provider`, and `path` for statuses ≥ 400.

## 6. Security guidance for app developers

- Keep the `.p12` and API keys in the platform secure store
  (Keychain/Keystore); mark keys non-exportable where supported.
- Ship `ca.crt` inside the app and pin it (the SDKs do this) — never fall
  back to the system trust store for the proxy connection.
- Handle 429 by backing off, not by retrying in a tight loop — the limit
  exists to contain a stolen device certificate.
- Expect certificate expiry after 365 days: surface a re-enrollment flow.
- Never log request headers or bodies on-device; they contain live keys.
