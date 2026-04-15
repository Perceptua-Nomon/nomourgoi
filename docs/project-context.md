# nomon — Project Context

Consolidated architecture reference for the nomon robot fleet development agents.

---

## System Overview

**nomon** is a fleet of intelligent, semi-autonomous robots providing utility to working- and middle-class people. The system is split across five integrated repositories:

| Repository | Language | Role |
|------------|----------|------|
| **nomopractic** | Rust | Low-latency HAT hardware daemon on Raspberry Pi. All hardware register knowledge lives here. |
| **nomothetic** | Python | Fleet API package: REST, camera, telemetry, HAT client. Runs in device mode (on Pi) or central mode (fleet server). |
| **nomotactic** | TypeScript | User-facing Expo (React Native) app: Android, iOS, and web from a single codebase. |
| **nomographic** | SQL | ArcadeDB graph database schemas and ArcadeDB-native migrations. Central (fleet-wide) and local (per-device) instances. |
| **nomourgoi** | Markdown | Development infrastructure: agents, prompts, shared standards. |

---

## Runtime Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│  User Layer                                                            │
│                                                                        │
│   nomotactic (Expo)                                                    │
│   ├── Android / iOS (mobile)                                           │
│   └── Web (browser)                                                    │
│        │                               │                               │
│        │ HTTPS :443                    │ HTTPS :8443                    │
│        ▼                               ▼                               │
│   ┌──────────────────┐         ┌──────────────────────────────────┐    │
│   │ nomothetic        │         │ nomothetic (device mode, on Pi)  │    │
│   │ (central mode)    │         │                                  │    │
│   │ Auth, Fleet,      │         │ JWT Auth (pairing), Camera, HAT, │    │
│   │ User mgmt         │         │ Audio, Stream, Calibration,      │    │
│   └────────┬──────────┘         │ Routines                         │    │
│            │                    └─────────────┬───────────────────┘    │
│            │                                   │                       │
│            ▼                                   │ Unix socket NDJSON    │
│   ┌──────────────────┐                         ▼                       │
│   │ ArcadeDB (central)│         ┌──────────────────────────────────┐   │
│   │ User, Vehicle,    │         │ nomopractic (Rust daemon on Pi)  │   │
│   │ Telemetry         │         │ Servo, Motor, ADC, GPIO,         │   │
│   └──────────────────┘         │ Ultrasonic, Speaker, BLE GATT   │   │
│                                 └─────────────┬───────────────────┘   │
│                                                │ rppal I2C + bluer BLE│
│                                                ▼                       │
│                                 ┌──────────────────────────────────┐   │
│                                 │ SunFounder Robot HAT V4          │   │
│                                 └──────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Hardware Constants

| Constant | Value | Notes |
|----------|-------|-------|
| I2C bus | 1 | Via rppal |
| HAT I2C address | `0x14` | |
| `REG_CHN` | `0x20` | PWM channel base, stride 1 |
| `REG_PSC` | `0x40` | Prescaler group 1 |
| `REG_ARR` | `0x44` | Auto-reload group 1 |
| `REG_PSC2` | `0x50` | Prescaler group 2 |
| `REG_ARR2` | `0x54` | Auto-reload group 2 |
| Clock | 72 MHz | |
| PWM period | 4095 | |
| Servo freq | 50 Hz | |
| Motor freq | 100 Hz | |
| Servo channels | 0–11 | |
| Motor channels | 12–15 | |

### GPIO Pin Map

| HAT Name | BCM pin | Direction | Purpose |
|----------|---------|-----------|---------|
| D2 | 27 | Output | Ultrasonic TRIG |
| D3 | 22 | Input  | Ultrasonic ECHO |
| D4 | 23 | Output | Motor0 DIR |
| D5 | 24 | Output | Motor1 DIR |
| MCURST | 5 | Output | MCU hardware reset |
| SW | 19 | Input  | User button |
| LED | 26 | Output | Status LED |
| SpeakerEn | 20 | Output | HifiBerry amp enable |

### BLE GATT UUIDs (Phase 13)

All vendor-specific 128-bit UUIDs use base `e3a1XXXX-7b2a-4b9c-8f5a-2b7d6e4f1a3c`.

| Service | UUID |
|---------|------|
| nomon Pairing | `e3a10001-7b2a-4b9c-8f5a-2b7d6e4f1a3c` |
| nomon Command | `e3a10002-7b2a-4b9c-8f5a-2b7d6e4f1a3c` |
| nomon WiFi Provisioning | `e3a10003-7b2a-4b9c-8f5a-2b7d6e4f1a3c` |
| nomon Status | `e3a10004-7b2a-4b9c-8f5a-2b7d6e4f1a3c` |

See nomopractic ADR-002 for the full GATT characteristic table and binary protocol.

### PicarX Motor Defaults

| Motor | PWM channel | DIR pin BCM | Reversed |
|-------|-------------|-------------|---------|
| Motor 0 | 12 | 24 (D5) | false |
| Motor 1 | 13 | 23 (D4) | true |

---

## IPC Contract

**Transport:** Unix domain socket at `/run/nomopractic/nomopractic.sock`  
**Framing:** Newline-delimited JSON (NDJSON), max 4096 bytes/message  
**Connection:** Persistent `SOCK_STREAM`; closing the connection idles all leased resources

### Request

```json
{"id": "1", "method": "method_name", "params": {...}}
```

### Success Response

```json
{"id": "1", "ok": true, "result": {...}}
```

### Error Response

```json
{"id": "1", "ok": false, "error": {"code": "ERROR_CODE", "message": "human description"}}
```

### Error Codes

| Code | Meaning |
|------|---------|
| `UNKNOWN_METHOD` | Unrecognised method name |
| `INVALID_PARAMS` | Missing or malformed params |
| `HARDWARE_ERROR` | I2C or GPIO failure |
| `NOT_READY` | Daemon not initialised |
| `SERVO_LEASE_EXPIRED` | TTL watchdog fired |
| `INTERNAL_ERROR` | Unexpected Rust error |

### IPC Methods (as of Phase 8)

| Method | Params | Result fields |
|--------|--------|---------------|
| `health` | `{}` | `status: "ok"` |
| `get_battery_voltage` | `{}` | `voltage_v: f64` |
| `set_servo_pulse_us` | `channel: u8, pulse_us: u32` | `channel, pulse_us` |
| `set_servo_angle` | `channel: u8, angle: f64` | `channel, angle_deg` |
| `reset_mcu` | `{}` | `reset: true` |
| `read_gpio` | `pin: str` | `pin, value: bool` |
| `write_gpio` | `pin: str, value: bool` | `pin, value` |
| `set_motor_speed` | `motor: u8, speed_pct: f64` | `motor, speed_pct` |
| `stop_all_motors` | `{}` | `stopped: true` |
| `get_motor_status` | `{}` | `motors: [{motor, speed_pct, active}]` |
| `read_ultrasonic` | `{}` | `distance_cm: f64` |
| `enable_speaker` | `{}` | `enabled: true` |
| `disable_speaker` | `{}` | `disabled: true` |

**Authoritative schema doc:** `nomothetic/docs/hat_ipc_schema.md`

---

## Module Maps

### nomopractic (Rust)

| Path | Responsibility |
|------|----------------|
| `src/main.rs` | CLI parsing, config load, tracing init, tokio runtime |
| `src/lib.rs` | Library root — re-exports modules |
| `src/config.rs` | TOML + env config with defaults |
| `src/ipc/mod.rs` | Unix socket listener, per-client task spawning, watchdog |
| `src/ipc/schema.rs` | Serde types: `HatRequest`, `HatResponse`, `ErrorBody` |
| `src/ipc/handler.rs` | Method dispatch to HAT driver functions |
| `src/hat/mod.rs` | HAT abstraction — re-exports sub-modules |
| `src/hat/i2c.rs` | Low-level I2C read/write via rppal, `I2cBus` trait |
| `src/hat/pwm.rs` | PWM register protocol (prescaler, channel writes) |
| `src/hat/adc.rs` | ADC read (command byte + 2-byte result) |
| `src/hat/servo.rs` | Servo control with TTL lease watchdog |
| `src/hat/motor.rs` | DC motor speed/direction control |
| `src/hat/battery.rs` | Battery voltage via ADC A4 |
| `src/hat/gpio.rs` | Named GPIO pins — `GpioPin` enum, `GpioBus` trait |
| `src/hat/ultrasonic.rs` | HC-SR04 distance sensor (TRIG/ECHO GPIO timing) |
| `src/ble/mod.rs` | BLE GATT server lifecycle, advertising (behind `ble` feature) |
| `src/ble/protocol.rs` | Binary frame codec (opcode/seq/length/payload) |
| `src/ble/services.rs` | GATT service + characteristic registration |
| `src/ble/session.rs` | Pairing, HKDF key derivation, AES-CCM encryption |
| `src/ble/bridge.rs` | BLE binary command → IPC handler dispatch |
| `src/ble/wifi.rs` | WiFi provisioning: nmcli scan/connect/status |
| `src/reset.rs` | MCU reset (assert BCM5 low ≥ 10 ms) |

### nomothetic (Python)

| Module | Purpose |
|--------|---------|
| `nomothetic/__init__.py` | Package exports |
| `nomothetic/api.py` | FastAPI HTTPS REST — primary control surface (device + central modes) |
| `nomothetic/mode.py` | API mode enum (`device` / `central`) and config-driven selection |
| `nomothetic/auth.py` | JWT auth: token issuance, validation, password hashing (central + device modes) |
| `nomothetic/auth_routes.py` | `/api/auth/*` endpoints (register, login, refresh, logout, profile) — central mode |
| `nomothetic/device_auth_routes.py` | `/api/device/auth/*` endpoints (pairing, refresh, profile) — device mode |
| `nomothetic/pairing.py` | Device pairing secret lifecycle (generate, verify, consume) |
| `nomothetic/fleet.py` | Fleet data: ArcadeDB queries for device management (central mode) |
| `nomothetic/fleet_routes.py` | `/api/fleet/*` endpoints (device CRUD) — central mode |
| `nomothetic/rate_limit.py` | Sliding-window rate limiting for auth and pairing endpoints |
| `nomothetic/db.py` | ArcadeDB HTTP API client with Gremlin query support |
| `nomothetic/user_store.py` | User persistence (Protocol + InMemory + Gremlin backends) |
| `nomothetic/fleet_store.py` | Fleet device persistence (Protocol + InMemory + Gremlin backends) |
| `nomothetic/token_store.py` | Refresh token persistence (Protocol + InMemory + Gremlin backends) |
| `nomothetic/gremlin_utils.py` | Shared Gremlin value sanitiser |
| `nomothetic/camera.py` | OV5647 capture via picamera2 |
| `nomothetic/streaming.py` | MJPEG Flask server for local LAN streaming |
| `nomothetic/telemetry.py` | MQTT background publisher (paho-mqtt) |
| `nomothetic/hat.py` | IPC client for nomopractic daemon (`HatClient`) |
| `nomothetic/audio.py` | USB microphone + HifiBerry DAC control |

### nomotactic (TypeScript / Expo)

| Path | Purpose |
|------|---------|
| `app/_layout.tsx` | Root layout: AuthProvider, StatusBar, theme |
| `app/index.tsx` | Smart entry: landing (web) / redirect (mobile) / device pairing prompt |
| `app/login.tsx` | Login / register screen |
| `app/(app)/_layout.tsx` | Auth guard, CommandInput bar |
| `app/(app)/index.tsx` | Device control dashboard (expandable cards) |
| `lib/api.ts` | Typed API client (fetch wrapper, per-URL auth headers) |
| `lib/auth.tsx` | AuthContext: central + device JWT management, pairing, expo-secure-store |
| `lib/ble.ts` | BLE service interface, mock + real implementations |
| `lib/ble-protocol.ts` | Binary frame codec for BLE GATT (Phase 2) |
| `lib/ble-session.ts` | AES-128-CCM session encryption + HKDF key derivation (Phase 2) |
| `lib/transport.tsx` | Hybrid transport provider: BLE ↔ HTTPS switching (Phase 2) |
| `lib/theme.ts` | Colour palette, spacing, typography constants |
| `constants/config.ts` | API URLs (DEVICE_API_URL, CENTRAL_API_URL) |
| `components/CommandInput.tsx` | AI-ready command input bar |

### nomographic (SQL / ArcadeDB Migrations)

| Path | Purpose |
|------|---------|
| `central/sql/V1__create_vehicle_schema.sql` | Vehicle + TelemetryReading vertices, HasTelemetry edge |
| `central/sql/V2__add_user_schema.sql` | User vertex + OwnsDevice edge |
| `central/sql/V3__create_refresh_token.sql` | RefreshToken vertex with token_hash, email, expiry |
| `local/sql/V1__create_device_schema.sql` | DeviceState + OperationLog vertices, Performed edge |
| `docker-compose.yml` | ArcadeDB with Gremlin Server plugin |
| `scripts/lib/migrate-common.sh` | Shared lineage tracking library (MetaType vertices + Supersedes edges) |
| *Auto-generated* | `{Type}Meta` vertices and `Supersedes` edges created by post-migration lineage hook |

---

## Development Status (BLE Pairing & Hybrid Connectivity Complete)

- **nomopractic**: 258 tests, Phases 1–11 + Phase 13 (BLE GATT Server) complete
- **nomothetic**: 531 tests, Phases 1–11 + Phases 13–18 complete
- **nomotactic**: Phase 1 (App Foundation & Auth) + Phase 2 (BLE Integration) complete — auth flow, device dashboard, web landing, device pairing, BLE connectivity, hybrid transport, command input.
- **nomographic**: V1 central (Vehicle) + V1 local (DeviceState) + V2 central (User & OwnsDevice) + V3 central (RefreshToken) schemas complete. Docker Compose with Gremlin Server plugin.
- Phases complete: GPIO, ADC/battery, servo, motor, CI/CD, Python client, audio, peripheral expansion, calibration, routines, central mode auth, user-facing app, ArcadeDB integration, deploy hardening, security hardening, device-mode auth, BLE GATT server, BLE pairing coordination, BLE client integration
- Target platform: Raspberry Pi Zero 2W, Debian trixie (aarch64)
- Dev/CI platform: x86_64 Linux (cross-compile via `cross`)

---

## Key Commands

```bash
# nomopractic build + test (requires source "$HOME/.cargo/env")
cargo build
cargo test
cargo clippy -- -D warnings
cargo fmt --check
cross build --target aarch64-unknown-linux-gnu --release
make check   # fmt + clippy + test

# nomothetic test + lint
uv run pytest tests/
uv run ruff check src/ tests/
uv run black --check src/ tests/
uv run mypy src/ tests/

# nomotactic lint
npx expo lint

# nomographic migration validation
./scripts/migrate-central.sh validate
./scripts/migrate-local.sh validate
```

---

## Pi Hardware

- SSH: `<pi-user>@<pi-host>`, key `~/.ssh/id_ed25519`
- USB mic: ALSA card 2 (Texas Instruments PCM2902)
- Speaker DAC: HifiBerry (ALSA card 1, `sndrpihifiberry`)
- Speaker enable: BCM20 (`spk_en` on Robot HAT V4)
- Ultrasonic: D2=BCM27 (TRIG), D3=BCM22 (ECHO), HC-SR04 compatible
