# nomon — Project Context

Consolidated architecture reference for the nomon robot fleet development agents.

---

## System Overview

**nomon** is a fleet of intelligent, semi-autonomous robots providing utility to working- and middle-class people. The system is split across three tightly integrated repositories:

| Repository | Language | Role |
|------------|----------|------|
| **nomopractic** | Rust | Low-latency HAT hardware daemon on Raspberry Pi. All hardware register knowledge lives here. |
| **nomothetic** | Python | Fleet API package: REST, camera, telemetry, HAT client. No hardware knowledge — delegates to Rust via IPC. |
| **nomourgoi** | Markdown | Development infrastructure: agents, prompts, shared standards. |

---

## Runtime Architecture

```
[ Mobile / Remote client ]
        │  HTTPS
        ▼
[ nomothetic — FastAPI on Pi ]
  ├── camera.py     (picamera2 OV5647)
  ├── streaming.py  (MJPEG Flask)
  ├── telemetry.py  (paho-mqtt)
  ├── audio.py      (ALSA USB mic + HifiBerry DAC)
  └── hat.py        (IPC client)
        │  Unix socket NDJSON
        ▼
[ nomopractic — Rust daemon on Pi ]
  ├── hat/servo.rs      (PWM servo, TTL lease)
  ├── hat/motor.rs      (TC1508S DC motor)
  ├── hat/battery.rs    (ADC voltage)
  ├── hat/gpio.rs       (named GPIO pins)
  ├── hat/ultrasonic.rs (HC-SR04 distance)
  └── hat/pwm.rs        (I2C PWM register protocol)
        │  rppal I2C
        ▼
[ SunFounder Robot HAT V4 ]
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
| Motor 0 | 12 | 24 (D4) | false |
| Motor 1 | 13 | 23 (D5) | false |

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
| `src/reset.rs` | MCU reset (assert BCM5 low ≥ 10 ms) |

### nomothetic (Python)

| Module | Purpose |
|--------|---------|
| `nomothetic/__init__.py` | Package exports |
| `nomothetic/api.py` | FastAPI HTTPS REST — primary control surface |
| `nomothetic/camera.py` | OV5647 capture via picamera2 |
| `nomothetic/streaming.py` | MJPEG Flask server for local LAN streaming |
| `nomothetic/telemetry.py` | MQTT background publisher (paho-mqtt) |
| `nomothetic/hat.py` | IPC client for nomopractic daemon (`HatClient`) |
| `nomothetic/audio.py` | USB microphone + HifiBerry DAC control |

---

## Development Status (Phase 8 Complete)

- **nomopractic**: 149 tests (123 unit + 26 integration)
- **nomothetic**: 262 tests
- Phases complete: GPIO, ADC/battery, servo, motor, CI/CD, Python client, audio, peripheral expansion
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
pip install -e .
pytest
black .
ruff check .
```

---

## Pi Hardware

- SSH: `perceptua@perceptua`, key `~/.ssh/id_ed25519`
- USB mic: ALSA card 2 (Texas Instruments PCM2902)
- Speaker DAC: HifiBerry (ALSA card 1, `sndrpihifiberry`)
- Speaker enable: BCM20 (`spk_en` on Robot HAT V4)
- Ultrasonic: D2=BCM27 (TRIG), D3=BCM22 (ECHO), HC-SR04 compatible
