# nomon — Security Checklist

Risk inventory and verification checklist for both repositories. The review agent applies this on every review pass.

---

## Threat Model

nomon robots expose hardware actuators and sensors to remote callers. The primary attack surfaces are:

1. **IPC socket** (`/run/nomopractic/nomopractic.sock`) — local Unix socket, accessible to any process running as the `nomothetic` service user
2. **REST API** (nomothetic FastAPI over HTTPS) — HTTPS on LAN + remote; authenticated
3. **MQTT telemetry** — outbound only; credentials must stay out of source
4. **Camera/audio streaming** — MJPEG and audio data; access control delegated to API layer

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
| P10 | Camera and streaming endpoints are authenticated | HIGH |
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

---

## BLE (nomopractic + nomotactic)

### BLE Authentication & Encryption

| # | Check | Severity if violated |
|---|-------|----------------------|
| B1 | [SUPERSEDED by ADR-004 — OS link-layer AES-CCM replaces application-layer crypto] ~~BLE pairing secret verified with constant-time compare (`hmac.compare_digest` equivalent in Rust)~~ | HIGH — timing side-channel leaks secret |
| B2 | [SUPERSEDED by ADR-004 — OS link-layer AES-CCM replaces application-layer crypto] ~~BLE pairing secret is single-use (consumed after first successful pairing)~~ | HIGH — replay attack |
| B3 | [SUPERSEDED by ADR-004 — OS link-layer AES-CCM replaces application-layer crypto] ~~BLE session key derived via HKDF-SHA256 (not raw pairing secret)~~ | HIGH — weak key derivation |
| B4 | [SUPERSEDED by ADR-004 — OS link-layer AES-CCM replaces application-layer crypto] ~~All post-pairing BLE commands encrypted with AES-128-CCM~~ | HIGH — command injection / eavesdropping |
| B5 | [SUPERSEDED by ADR-004 — OS link-layer AES-CCM replaces application-layer crypto] ~~AES-CCM nonce uses monotonic counter; server rejects counter ≤ last seen~~ | HIGH — replay attack |
| B6 | [SUPERSEDED by ADR-004 — OS link-layer AES-CCM replaces application-layer crypto] ~~AES-CCM nonce includes direction byte (client→server vs server→client)~~ | MEDIUM — nonce reuse across directions |
| B7 | BLE session state cleared on client disconnect | MEDIUM — stale session key |
| B8 | JWT issued over BLE uses same `NOMON_JWT_SECRET` as HTTPS (no separate weak secret) | HIGH — token forgery |
| B9 | JWT issued over BLE uses `iss: "nomon-device"` (not `nomon-central`) | HIGH — cross-mode token reuse |
| B10 | Shared pairing secret file (`/var/lib/nomon/pairing_secret`) has mode `0640`, owner `root:nomon` | HIGH — unauthorized secret read |
| B13 | `encrypt_authenticated_write: true` is set on the GATT Command Write characteristic | HIGH — unauthenticated writes accepted |
| B14 | BLE bridge NDJSON buffer overflow protection is active (max 8 KB per line) | MEDIUM — memory exhaustion |
| B15 | Passkey file at `pairing_secret_path` has mode `0600` and is owned by the daemon user | HIGH — unauthorized secret read |

### BLE Input Validation

| # | Check | Severity if violated |
|---|-------|----------------------|
| B11 | [SUPERSEDED by ADR-004 — OS link-layer AES-CCM replaces application-layer crypto] ~~BLE binary frame length validated against opcode's expected payload size~~ | HIGH — buffer over-read |
| B12 | [SUPERSEDED by ADR-004 — OS link-layer AES-CCM replaces application-layer crypto] ~~BLE opcode validated before dispatch (unknown opcode → error response, not panic)~~ | HIGH — daemon crash |
| B16 | BLE motor/servo/sensor parameters validated identically to IPC params (same handler) | HIGH — out-of-range hardware write |
| B17 | BLE advertising name length ≤ 29 bytes (BLE spec limit) | LOW — advertising failure |

### BLE Transport

| # | Check | Severity if violated |
|---|-------|----------------------|
| B18 | BLE GATT server does not expose hardware commands without prior pairing (pre-auth characteristics limited to health/status) | HIGH — unauthorized motor control |
| B19 | WiFi password written over BLE is not logged or persisted beyond `nmcli` | HIGH — credential leak |
| B20 | BLE disconnect triggers motor/servo lease cleanup (same as IPC disconnect) | HIGH — unsafe actuator state |
| B21 | `react-native-ble-plx` permissions requested at runtime (Android 12+ Bluetooth permissions) | MEDIUM — app crash or silent failure |

---

## Cross-Repo

| # | Check | Severity if violated |
|---|-------|----------------------|
| X1 | IPC method names match exactly in Rust handler and Python client | HIGH — silent failure |
| X2 | Error codes defined in Rust `ErrorCode` are all handled in Python | MEDIUM — unhandled errors |
| X3 | Hardware constants (BCM pins, I2C address) agree between repos | HIGH — wrong hardware write |
| X4 | `hat_ipc_schema.md` reflects the current implementation | MEDIUM — misleads developers |
| X5 | Integration tests cover round-trip: Python call → IPC → Rust → response | HIGH — schema drift undetected |

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
rg -n "\.unwrap\(\)|\.expect\(" nomopractic/src/ --include="*.rs" | grep -v "#\[cfg(test)\]"

# Check for unsafe blocks in Rust
rg -n "unsafe" nomopractic/src/ --include="*.rs"

# Check for hardcoded secrets in Python
grep -rn "password\s*=\s*['\"].\|api_key\s*=\s*['\"].\|token\s*=\s*['\"]." nomothetic/src/

# Check for eval/exec in Python
grep -rn "eval(\|exec(" nomothetic/src/

# Rust security audit
cd nomopractic && cargo audit

# Python dependency check
cd nomothetic && pip-audit
```
