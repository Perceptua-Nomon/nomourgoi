# nomon — Security Checklist

Risk inventory and verification checklist for both repositories. The review agent applies this on every review pass.

---

## Threat Model

nomon robots expose hardware actuators and sensors to remote callers. The primary attack surfaces are:

1. **IPC socket** (`/run/nomopractic/nomopractic.sock`) — local Unix socket, accessible to any process running as the `nomothetic` service user
2. **REST API** (nomothetic FastAPI over HTTPS) — HTTPS on LAN + remote; authenticated
3. **MQTT telemetry** — outbound only; credentials must stay out of source
4. **Camera/audio streaming** — MJPEG and audio data; stream start/stop is JWT-gated via the API, and the MJPEG server itself (plain HTTP, outside the API) requires a per-run `?token=` access token minted by `/api/stream/start`
5. **Plugin auth** (`/api/plugin/*`) — Ed25519 challenge-response bootstrap for on-device autonomy plugins (autonomon); key registration is localhost-only, nonces are single-use and TTL-bounded (nomothetic ADR-019)

---

## Rust (nomopractic)

### Code Safety

| # | Check | Severity if violated |
|---|-------|----------------------|
| R1 | No `unwrap()` or `expect()` in non-test code | HIGH — panic crashes the daemon |
| R2 | No `unsafe` block without an adjacent `// SAFETY:` comment | HIGH — undefined behaviour risk |
| R3 | `tokio::sync::Mutex` guards dropped before `.await` | HIGH — deadlock risk |
| R4 | All IPC channels have bounded capacity (`mpsc::channel(N)`) | MEDIUM — unbounded memory growth |
| R5 | `tracing` logs never emit secrets or credentials | HIGH — log injection / credential leak |

### Input Validation (IPC boundary)

| # | Check | Severity if violated |
|---|-------|----------------------|
| R6 | Servo channel validated 0–11 before hardware write | HIGH — out-of-range I2C write |
| R7 | Servo pulse_us validated within safe range (500–2500 µs) | HIGH — hardware damage risk |
| R8 | Motor speed_pct clamped to -100.0–100.0 | MEDIUM — uncontrolled actuator |
| R9 | GPIO pin validated against known enum values | HIGH — arbitrary I2C/GPIO write |
| R10 | JSON message size enforced ≤ 4096 bytes at read time | MEDIUM — memory exhaustion |
| R11 | Method name length bounded before dispatch | LOW — log spam / DoS |

### IPC Protocol

| # | Check | Severity if violated |
|---|-------|----------------------|
| R12 | Unrecognised methods return `UNKNOWN_METHOD`, not a panic | HIGH |
| R13 | Malformed JSON returns `INVALID_PARAMS`, not a panic | HIGH |
| R14 | Rate-limit on `reset_mcu` (≥ 1 second between resets) | MEDIUM — MCU brownout |
| R15 | TTL watchdog idles servos and motors on client disconnect | HIGH — unsafe actuator state |

---

## Python (nomothetic)

### Injection Prevention

| # | Check | Severity if violated |
|---|-------|----------------------|
| P1 | No `eval()` or `exec()` on any user-supplied input | CRITICAL — code injection |
| P2 | No shell string interpolation with user data (use `subprocess.run([...])` lists) | CRITICAL — command injection |
| P3 | File paths from user input validated/normalised; no path traversal | HIGH — arbitrary file read/write |
| P4 | FastAPI endpoint params use Pydantic models with type/range validation | HIGH — INVALID_PARAMS reach hardware |
| P14 | Gremlin property keys whitelisted against model fields in update methods | HIGH — arbitrary property injection |
| P15 | `_sanitize_gremlin_value` rejects null bytes and control characters | MEDIUM — parser confusion |

### Secrets Management

| # | Check | Severity if violated |
|---|-------|----------------------|
| P5 | No hardcoded passwords, API keys, or tokens in source | CRITICAL |
| P6 | No hardcoded TLS certificate paths that bypass config | HIGH |
| P7 | MQTT credentials from environment variables or gitignored config | HIGH |
| P8 | `.gitignore` covers config files containing credentials | HIGH |

### Authentication & Access

| # | Check | Severity if violated |
|---|-------|----------------------|
| P9 | REST API uses HTTPS (no plaintext HTTP endpoints) | HIGH |
| P10 | Camera and streaming endpoints are authenticated; the MJPEG stream server (outside the JWT'd API) enforces its per-run `?token=` on `/` and `/stream` | HIGH |
| P11 | CORS policy is restrictive, not `allow_origins=["*"]` on authenticated routes | MEDIUM |
| P16 | Server-side logout endpoint revokes refresh tokens | MEDIUM — token persists after logout |
| P17 | Device-mode startup warns if Tailscale not detected | LOW — silent misconfiguration |
| P18 | Device pairing secret generated with `secrets.token_urlsafe` (≥128 bits) | HIGH — predictable secret |
| P19 | Pairing secret consumed on first use (single-use, constant-time compare) | HIGH — replay attack |
| P20 | Device JWT issuer (`nomon-device`) differs from central (`nomon-central`) | HIGH — cross-mode token reuse |
| P21 | Pairing endpoint rate-limited (3/min per IP) | MEDIUM — brute-force pairing secret |

### Dependency Safety

| # | Check | Severity if violated |
|---|-------|----------------------|
| P12 | No Pi-specific library imported unconditionally (breaks CI silently) | MEDIUM |
| P13 | No outdated dependencies with known CVEs | HIGH |

### WiFi Provisioning

| # | Check | Severity if violated |
|---|-------|----------------------|
| P22 | SSID rejects null bytes and control characters (`\x00–\x1f`, `\x7f`) | MEDIUM — nmcli robustness / DoS |
| P23 | SSID rejects leading `-` character | MEDIUM — nmcli argument injection |
| P24 | `/run/nomothetic/pairing-secret` written with mode `0o600` (owner-read-only) | HIGH — pairing secret world-readable |

### Deployment Guards

| # | Check | Severity if violated |
|---|-------|----------------------|
| P25 | `NOMON_DEVICE_AUTH=false` never appears in production systemd units | CRITICAL — disables ALL device endpoint authentication |

---

## Python (autonomon)

| # | Check | Severity if violated |
|---|-------|----------------------|
| A1 | Plugin Ed25519 private key written `0600`, owned by the service user, atomic write | HIGH — key theft yields device JWTs |
| A2 | Device JWT lives in memory only; never written to disk or logged | HIGH — token exfiltration |
| A3 | NDJSON lifecycle events and stderr logs never carry credentials | HIGH — secrets land in the nomothetic journal |
| A4 | `/etc/autonomon/autonomon.env` contains no secrets (key *path* only) | MEDIUM — file is world-readable by design |
| A5 | All device I/O via the nomothetic REST API — no direct I2C/GPIO (ADR-004) | MEDIUM — bypasses gateway validation |
| A6 | `httpx` `verify=False` is limited to device connections (self-signed certs, nomothetic ADR-001) — never to central/public hosts | HIGH — MITM on real TLS endpoints |

---

## Cross-Repo

| # | Check | Severity if violated |
|---|-------|----------------------|
| X1 | IPC method names match exactly in Rust handler and Python client | HIGH — silent failure |
| X2 | Error codes defined in Rust `ErrorCode` are all handled in Python | MEDIUM — unhandled errors |
| X3 | Hardware constants (BCM pins, I2C address) agree between repos | HIGH — wrong hardware write |
| X4 | `hat_ipc_schema.md` reflects the current implementation | MEDIUM — misleads developers |
| X5 | Integration tests cover round-trip: Python call → IPC → Rust → response | HIGH — schema drift undetected |

### Token Storage (nomotactic)

| # | Check | Severity if violated |
|---|-------|----------------------|
| X6 | Access tokens held in React state only on web (never written to any browser storage) | HIGH — XSS exfiltration |
| X7 | Refresh tokens use `sessionStorage` on web (not `localStorage`; cleared on tab close) | MEDIUM — persistent token after session end |
| X8 | Mobile tokens stored in `expo-secure-store` (OS keychain), not AsyncStorage | HIGH — plaintext token on device storage |

---

## Fleet & Registration

> These items apply to the central-mode nomothetic service and the fleet management API.

| # | Check | Severity if violated |
|---|-------|----------------------|
| FL1 | **Registration proof lacks cryptographic signature verification.** The device VIN proof JWT signature is NOT verified by the central server (device and central use separate secrets). An authenticated central user can forge a proof claiming any VIN. **Planned mitigation**: asymmetric per-device EC certificates. Until then, rely on trust between central-authenticated users. | MEDIUM — multi-tenant VIN squatting |
| FL2 | Fleet API update endpoints whitelist property keys against the device model schema | HIGH — arbitrary property injection into ArcadeDB |
| FL3 | Device-to-central registration requires a valid central-issued JWT (not just structural proof) | HIGH — unauthenticated device registration |

---

## OWASP Top 10 Applicability

| OWASP category | nomon exposure | Primary mitigations |
|---------------|---------------|---------------------|
| A01 Broken Access Control | REST API endpoints, camera feed | HTTPS + auth on all routes |
| A02 Cryptographic Failures | HTTPS certs, MQTT credentials | Self-signed TLS (ADR 001), env-var secrets |
| A03 Injection | IPC method names, API params | Pydantic validation, enum dispatch in Rust |
| A04 Insecure Design | Actuator control over network | TTL watchdog, rate limits, bounded channels |
| A05 Security Misconfiguration | CORS, default credentials | Explicit CORS origins, no defaults in source |
| A06 Vulnerable Components | Python/Rust deps | Regular `cargo audit`, `pip-audit` review |
| A07 Auth Failures | REST API | HTTPS auth required (see nomothetic docs) |
| A08 Software Integrity | OTA firmware updates (future) | Signed releases, verify before flash |
| A09 Logging Failures | HAT daemon, API errors | Structured tracing in Rust, Python logging |
| A10 SSRF | No outbound URL fetching today | N/A (design-time: avoid dynamic URLs from input) |

---

## Verification Commands

```bash
# Check for unwrap in production Rust (should return zero results outside #[cfg(test)])
rg -n "\.unwrap\(\)|\.expect\(" nomopractic/src/ -g "*.rs" | grep -v "#\[cfg(test)\]"

# Check for unsafe blocks in Rust
rg -n "unsafe" nomopractic/src/ -g "*.rs"

# Check for hardcoded secrets in Python
grep -rn "password\s*=\s*['\"].\|api_key\s*=\s*['\"].\|token\s*=\s*['\"]." nomothetic/src/

# Check for eval/exec in Python
grep -rn "eval(\|exec(" nomothetic/src/

# Check pairing secret display file permissions (P24)
grep -n "0o6" nomothetic/src/nomothetic/api.py | grep -i "secret\|pairing"

# Check NOMON_DEVICE_AUTH is never 'false' in systemd units (P25)
grep -rn "NOMON_DEVICE_AUTH=false" nomothetic/systemd/

# Rust security audit
cd nomopractic && cargo audit

# Python dependency check
cd nomothetic && pip-audit
```
