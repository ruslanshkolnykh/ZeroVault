# Compliance & DoD-grade hardening

Audit round 2, aligning llm-proxy with the frameworks a DoD / national-security
customer will assess against: NIST SP 800-53 rev 5, CNSA 1.0/2.0, FIPS 140-3,
DISA STIGs, NIST SP 800-207 (zero trust), CMMC. This complements SECURITY.md
(round 1 threat model and fixes).

**Honest scope statement.** No open-source project is "DoD-compliant" by
itself — authorization (ATO) attaches to a *system*: this software plus the
hardened OS beneath it, the enclave it runs in, the people and processes
around it. What the code can do is implement the technical controls cleanly
and document the rest. That is what this round delivers. Claims below marked
*operator* are inherited from the deployment environment, not the code.

## Round-2 findings and fixes (all applied)

**B1 — Crypto below CNSA guidance.** Certificates used P-256/SHA-256. CNSA
1.0 requires P-384, SHA-384 and AES-256 for national-security systems.
*Fixed:* `generate-certs.sh` now defaults to **P-384 keys and SHA-384
signatures** (CA, server, device certs and CRL); AES-256 already protected
the CA key, and TLS_AES_256_GCM_SHA384 is first-preference. `CURVE`/`HASH`
env overrides remain for non-NSS fleets. (CNSA 2.0's post-quantum
algorithms — ML-KEM/ML-DSA — are not yet practical for client-cert PKI in
mainstream TLS stacks; tracked as a residual item.)

**B2 — No FIPS operating mode.** ChaCha20-Poly1305 (not FIPS-approved) was
negotiable, and TLS 1.2 could be enabled by config. *Fixed:* `FIPS_MODE=1`
in all three runtimes forces TLS 1.3 and, in Node, AES-GCM suites only;
`TLS_MIN_VERSION` downgrades are ignored in this mode. Full FIPS 140-3
requires the validated crypto module underneath — Node against an OpenSSL
FIPS provider, Go built with `GOEXPERIMENT=boringcrypto`, Python's ssl
against a FIPS OpenSSL — documented per runtime (*operator*).

**B3 — Revocation required a restart (IA-5/CM windows).** The CRL was read
once at startup, so a revoked device kept access until someone restarted the
service. *Fixed:* Node and Go now **hot-reload the CRL** (default every 60 s;
`CRL_RELOAD_SECS`) — copy the new `crl.pem` over and the revoked device's
next handshake fails, no restart, verified live. Python still requires a
restart (its SSL context is baked at bind time) and its runbook says so.

**B4 — Logs were not audit-grade (AU-3).** Free-text lines lacked machine
structure and consistent event typing. *Fixed:* `LOG_FORMAT=json` emits
structured records — `ts`, `evt` (`startup`, `request`, `access_denied`,
`tls_rejected`, `crl_reloaded`), device CN, source IP, method, path,
status — still never headers, bodies, queries or keys. Stdout → journald →
your SIEM; journald sealing / remote forwarding is the AU-9 integrity leg
(*operator*).

**B5 — No concurrent-connection ceiling (SC-5).** Rate limits bound request
*frequency* but not open sockets; a certificate holder could exhaust file
descriptors with idle connections. *Fixed:* `MAX_CONNS` (default 500) caps
concurrent TLS connections in Node (`server.maxConnections`) and Go (a
semaphore listener); Python already had `limit_concurrency`.

**B6 — Floating dependency versions (CM-2/SR-3).** Python's requirements
used `>=` ranges, so two installs could run different code. *Fixed:* exact
`==` pins with instructions to regenerate hash-pinned lockfiles via
`pip-compile --generate-hashes`. Node/Go implementations and the JS/Python
SDKs have zero third-party dependencies at all — the strongest supply-chain
posture available.

**B7 — SDK input hardening (SI-10).** The JS and Python SDKs passed `path`
into the HTTP request line unvalidated; platform stacks reject CR/LF, but
defense-in-depth belongs in our code, and Python's provider check accepted
Unicode alphanumerics. *Fixed:* both SDKs now reject control characters,
whitespace and `..` in paths at a single choke point, and provider names are
strict ASCII `[a-z0-9-]+`. (Kotlin/Swift build URLs through OkHttp/URL
APIs that normalize and reject these natively.)

## NIST SP 800-53 rev 5 — technical control mapping

| Family | Control | How it's met |
|---|---|---|
| AC | AC-3, AC-4 | mTLS gate before any HTTP; fixed upstream allowlist (no client-controlled egress); `ALLOWED_DEVICES` CN allowlist |
| AC | AC-7/SC-5 | per-device token-bucket rate limits, body caps, `MAX_CONNS`, tight header timeouts |
| AU | AU-2/3/12 | structured JSON audit events with actor (cert CN), action, outcome, source, time |
| AU | AU-9 | append-only via journald; sealing/remote forward (*operator*) |
| IA | IA-2/3 | device authentication by X.509 client certificate, CA-pinned both directions |
| IA | IA-5 | 365-day cert lifetime; CRL revocation hot-reloaded ≤60 s; encrypted offline CA key; keys never at rest on the gateway |
| SC | SC-8/13 | TLS 1.3 (AEAD, forward secrecy); CNSA-aligned P-384/SHA-384/AES-256; `FIPS_MODE` |
| SC | SC-7 | single-port boundary; syscall-filtered systemd / capability-free read-only containers |
| SI | SI-10 | path-ambiguity rejection (`..`, encodings, NUL) at proxy and SDKs |
| CM | CM-2/7 | root-owned code the service cannot modify; pinned deps; one-command reproducible install |
| SR | SR-3 | zero-dependency Node/Go/SDK builds; ~200 auditable lines per runtime |
| IR/CP | IR-4, CP-10 | runbooks: revocation drill, CA-compromise procedure, stateless rebuild |

## Framework notes

- **CNSA 1.0:** met by default after B1 (P-384 / SHA-384 / AES-256-GCM /
  TLS 1.3). **CNSA 2.0** (post-quantum): residual — revisit when ML-DSA
  client certs land in OpenSSL/Go/mobile keystores.
- **FIPS 140-3:** `FIPS_MODE` + a validated module per B2. The *code* never
  implements crypto itself — it only configures platform TLS, which is the
  right shape for FIPS.
- **DISA STIGs:** OS-level (Ubuntu/RHEL STIG) is *operator*; the systemd
  units already exceed the service-hardening items (NoNewPrivileges,
  SystemCallFilter, ProtectSystem=strict, etc.). Apply the OS STIG plus
  `auditd` and FIPS kernel mode on the host.
- **NIST 800-207 zero trust:** per-device identity, per-request credentials,
  no implicit network trust, continuous revocation — the architecture is
  natively zero-trust; document this mapping in your ATO package.
- **CMMC / 800-171:** the same technical controls cover the AC/AU/IA/SC
  families; the rest is organizational.

## Residual items an assessor will ask about

Post-quantum (CNSA 2.0) timelines; OCSP vs CRL (CRL chosen deliberately —
offline-friendly, no third-party call on the handshake path; interval is
60 s); Python runtime's restart-based revocation and IP-keyed rate limits
(prefer Node/Go where this matters); FIPS validation certificates for the
underlying crypto modules belong to Node/OpenSSL/Go/BoringCrypto, not this
project; and continuous monitoring (CA-7) is an operator program — the
structured logs are its feed.
