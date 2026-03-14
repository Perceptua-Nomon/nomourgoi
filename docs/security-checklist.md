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

### Dependency Safety

| # | Check | Severity if violated |
|---|-------|----------------------|
| P12 | No Pi-specific library imported unconditionally (breaks CI silently) | MEDIUM |
| P13 | No outdated dependencies with known CVEs | HIGH |

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
