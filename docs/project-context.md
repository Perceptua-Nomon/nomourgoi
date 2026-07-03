# nomon — Project Context

Consolidated architecture reference for the nomon robot fleet development agents.

---

## System Overview

**nomon** is a fleet of intelligent, semi-autonomous robots providing utility to working- and middle-class people. The system is split across six integrated repositories:

| Repository | Language | Role |
|------------|----------|------|
| **nomopractic** | Rust | Low-latency HAT hardware daemon on Raspberry Pi. All hardware register knowledge lives here. |
| **nomothetic** | Python | Fleet API package: REST, camera, telemetry, HAT client. Runs in device mode (on Pi) or central mode (fleet server). |
| **nomotactic** | TypeScript | User-facing Expo (React Native) app: Android, iOS, and web from a single codebase. |
| **nomographic** | SQL | ArcadeDB graph database schemas and ArcadeDB-native migrations. Central (fleet-wide) and local (per-device) instances. |
| **autonomon** | Python | The brain: four-layer cognitive pipeline (Perception → World Model → Planning → Action) driving a device through the nomothetic REST API. Standalone venv; never imported by nomothetic (autonomon ADR-004/005). |
| **nomourgoi** | Markdown | Development infrastructure: agents, prompts, shared standards. |

**Boundary rules:** all hardware register knowledge lives *below* nomothetic (in nomopractic); all perception processing, modeling, and planning lives *above* it (in autonomon). nomothetic is a thin raw-I/O gateway between them. autonomon publishes its routine catalogue to a shared file (`/var/lib/nomon/routine_catalog.json`, override `NOMON_ROUTINE_CATALOG_PATH`) at deploy time; nomothetic reads that file to list and launch routines — the two never import each other.

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
│   └──────────────────┘         │ Ultrasonic, Speaker   │   │
│                                 └─────────────┬───────────────────┘   │
│                                                │ rppal I2C│
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
| `SERVO_LEASE_EXPIRED` | Servo lease TTL elapsed; channel idled until next command |
| `ALREADY_RUNNING` | A routine is already active; call `stop_routine` first |
| `TIMEOUT` | Ultrasonic ECHO pulse did not arrive within timeout |
| `NO_ECHO` | Ultrasonic measurement outside valid range (2–400 cm) |
| `INTERNAL_ERROR` | Unexpected Rust error |

### IPC Methods (35 methods — Phases 1–11)

| Method | Direction | Purpose |
|--------|-----------|---------|
| `health` | read | Daemon liveness and hardware connection status |
| `get_battery_voltage` | read | Battery voltage via ADC channel A4 |
| `read_adc` | read | Raw 16-bit value from ADC channel 0–7 |
| `set_servo_pulse_us` | write | Set PWM channel pulse width in microseconds |
| `set_servo_angle` | write | Set servo angle in degrees (0–180°) |
| `get_servo_status` | read | Active servo TTL lease list |
| `read_gpio` | read | Read named GPIO pin level |
| `write_gpio` | write | Drive named GPIO output pin high/low |
| `set_motor_speed` | write | Set single DC motor speed (signed %) |
| `stop_all_motors` | write | Immediately stop all motors, clear leases |
| `get_motor_status` | read | Active motor TTL lease list |
| `drive` | write | Set all motors to same speed (atomic) |
| `steer` | write | Set steering servo angle |
| `pan_camera` | write | Set camera pan servo angle |
| `tilt_camera` | write | Set camera tilt servo angle |
| `reset_mcu` | write | Assert/de-assert MCU reset line |
| `get_mcu_status` | read | MCU reset statistics since daemon start |
| `read_ultrasonic` | read | HC-SR04 distance measurement |
| `read_grayscale` | read | Raw ADC values from grayscale sensors |
| `read_grayscale_normalized` | read | Calibration-normalized grayscale (0.0–1.0) |
| `enable_speaker` | write | Assert speaker amplifier enable pin (BCM 20) |
| `disable_speaker` | write | De-assert speaker amplifier enable pin |
| `set_volume` | write | Set HifiBerry DAC output volume (0–100%) |
| `get_volume` | read | Read current output volume |
| `set_mic_gain` | write | Set USB microphone capture gain (0–100%) |
| `get_mic_gain` | read | Read current microphone gain |
| `get_calibration` | read | Full calibration store snapshot |
| `set_motor_calibration` | write | Adjust motor calibration (partial update) |
| `set_servo_calibration` | write | Set servo trim offset (µs) |
| `calibrate_grayscale` | write | Capture live ADC as surface reference value |
| `save_calibration` | write | Persist calibration store to disk |
| `reset_calibration` | write | Revert calibration to factory defaults |
| `start_routine` | write | Start a named autonomous routine |
| `stop_routine` | write | Stop active routine, return telemetry stats |
| `get_routine_status` | read | Query routine engine state |

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
| `src/reset.rs` | MCU reset (assert BCM5 low ≥ 10 ms) |
| `src/calibration.rs` | CalibrationStore: motor/servo/grayscale calibration |
| `src/testing.rs` | Shared test mocks: MockI2c, MockGpio, MockAlsaControl (`#[cfg(test)]`) |
| `src/ipc/params.rs` | Typed IPC parameter extraction helpers (ParamExtractor) |

### nomothetic (Python)

| Module | Purpose |
|--------|---------|
| `nomothetic/__init__.py` | Package exports |
| `nomothetic/api.py` | FastAPI HTTPS REST — primary control surface (device + central modes) |
| `nomothetic/mode.py` | API mode enum (`device` / `central`) and config-driven selection |
| `nomothetic/auth.py` | JWT auth: token issuance, validation, password hashing (central + device modes) |
| `nomothetic/auth_routes.py` | `/api/auth/*` endpoints (register, login, refresh, logout, profile) — central mode |
| `nomothetic/device_auth_routes.py` | `/api/device/auth/*` endpoints (pairing, refresh, profile, identity) — device mode |
| `nomothetic/pairing.py` | Device pairing secret lifecycle (generate, verify, consume) |
| `nomothetic/fleet_routes.py` | `/api/fleet/*` endpoints (device CRUD + registration proof validation) — central mode |
| `nomothetic/rate_limit.py` | Sliding-window rate limiting for auth and pairing endpoints |
| `nomothetic/db.py` | ArcadeDB HTTP API client with Gremlin query support |
| `nomothetic/user_store.py` | User persistence (Protocol + InMemory + Gremlin backends) |
| `nomothetic/fleet_store.py` | Fleet device persistence (Protocol + InMemory + Gremlin backends) |
| `nomothetic/token_store.py` | Refresh token persistence (Protocol + InMemory + Gremlin backends) |
| `nomothetic/device_jwt.py` | Device JWT secret persistence (`DeviceJwtSecretStore`, atomic 0600 write) |
| `nomothetic/db_utils.py` | Shared database query utilities and Gremlin value sanitiser |
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
| `lib/transport.tsx` | HTTPS transport provider |
| `lib/endpoints.ts` | API endpoint string constants |
| `lib/usePolling.ts` | Reusable polling hook |
| `lib/useDeviceCommand.ts` | Transport-switching command hook |
| `lib/theme.ts` | Colour palette, spacing, typography constants |
| `constants/config.ts` | API URLs (DEVICE_API_URL, CENTRAL_API_URL) |
| `components/CommandInput.tsx` | AI-ready command input bar |
| `components/HttpPairingForm.tsx` | HTTP device pairing form (Soft AP pairing flow) |
| `components/WifiProvisionForm.tsx` | Wi-Fi credential form (rendered inline after successful pairing) |
| `components/DeviceRegistrationForm.tsx` | Central fleet registration; discovery-driven flow (direct/ap/needs-pairing) |
| `components/ExpandableCard.tsx` | Expandable dashboard card base component |
| `components/StatusCard.tsx` | Device status card |
| `components/ControlPad.tsx` | Directional control pad |
| `components/MotorCard.tsx` | Motor speed / direction controls |
| `components/PanTiltPad.tsx` | Camera pan/tilt 2D pad |
| `components/CameraCard.tsx` | Live camera feed card |
| `components/RoutineCard.tsx` | Autonomous routine start/stop card |
| `components/SensorCard.tsx` | Sensor readings card |
| `components/SensorBar.tsx` | Horizontal sensor bar gauge |
| `components/VideoFeed.tsx` | MJPEG video feed component |
| `components/ConnectionIndicator.tsx` | Connection status indicator |
| `lib/devices.ts` | Central fleet device list utilities |
| `lib/local-devices.ts` | Local (Soft-AP-paired) device registry |

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

### autonomon (Python — autonomy)

| Path | Purpose |
|------|---------|
| `src/autonomon/messages.py` | `PerceptionEvent` / `WorldStateUpdate` / `ActionPlan` / `ActionResult` dataclasses |
| `src/autonomon/pipeline.py` | `Pipeline` — wires the four layers with typed, bounded asyncio queues |
| `src/autonomon/slot.py` | `LayerSlot` / `SlotState` — owns one layer's asyncio task + queues |
| `src/autonomon/fan_in.py` | `FanInSlot` — multi-source perception fan-in |
| `src/autonomon/plugin_auth.py` | Ed25519 challenge-response device-JWT auth (nomothetic ADR-019); key never leaves disk, JWT never touches it |
| `src/autonomon/perception/` | `PerceptionBase`, `Perceptron` (sensor polling), vision detectors (`yolo-onnx` / `opencv-dnn` / `opencv-hog` / `fake`) |
| `src/autonomon/world_model/` | `ObstacleWorldModel`, `TargetWorldModel` |
| `src/autonomon/planning/` | `AvoidancePlanner`, `FollowPlanner` (`PursuitPlanner` retained but superseded) |
| `src/autonomon/action/` | `VehicleAction` — drive/steer/pan/tilt via the nomothetic REST API |
| `src/autonomon/routines/` | Registry, `explore` / `follow-user` factories, `nomon-autonomon` CLI, catalogue publish, status reporting |

---

## Development Status (Device Fleet Registration Complete)

- **nomopractic**: 239 tests, Phases 1–11, 14–15 complete (Phase 12 planned, Phases 13/13.1 superseded by Phase 15)
- **nomothetic**: 591 tests, Phases 1–11, 13–23 complete (Phase 12 planned, Phases 18/18.1 superseded by Phase 20)
- **nomotactic**: Phases 1.1–1.6, 8, 15 complete — auth flow, device dashboard, web landing, Wi-Fi AP pairing, fleet device registration. BLE phases (1.5, 2, 2.1, 2.2) superseded by Phase 15.
- **nomographic**: Phases 1–7 complete. V3 central (User, Vehicle, RefreshToken) + V1 local (DeviceState) schemas.
- **autonomon**: four-layer pipeline framework with `explore` and `follow-user` routines; Ed25519 plugin auth; file-based catalogue handoff to nomothetic (ADR-005). Phases 3/4/7 deferred; 5/6b/6c active — see `autonomon/docs/roadmap.md`.
- Latest completed work: device fleet registration & identity (nomothetic Phase 23 / nomotactic Phase 8); network provisioning security hardening (SSID control-char/leading-dash validation, pairing secret file permissions 0o600, registration proof FL1 warning)
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

# autonomon check (same Python toolchain as nomothetic)
make check   # ruff + black + mypy + pytest
```

---

## Pi Hardware

- SSH: `<pi-user>@<pi-host>`, key `~/.ssh/id_ed25519`
- USB mic: ALSA card 2 (Texas Instruments PCM2902)
- Speaker DAC: HifiBerry (ALSA card 1, `sndrpihifiberry`)
- Speaker enable: BCM20 (`spk_en` on Robot HAT V4)
- Ultrasonic: D2=BCM27 (TRIG), D3=BCM22 (ECHO), HC-SR04 compatible
