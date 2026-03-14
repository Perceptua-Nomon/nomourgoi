# nomon — Coding Standards

Unified style guide for both repositories. Follow these conventions in all new code.

---

## Universal Principles

1. **Clarity over cleverness.** Code is read far more than it is written. Optimise for the next reader.
2. **Minimal dependencies.** Every new crate or library requires justification. Prefer std-lib solutions.
3. **No production shortcuts.** Tests are not optional. Lints are not optional. Documentation is not optional.
4. **Fail loudly at boundaries, quietly in steady state.** Validate inputs early; log errors with context; never swallow failures silently.
5. **Security by default.** Validate all inputs at IPC and API boundaries. No secrets in source. No `unsafe` without written invariant.

---

## Rust (nomopractic)

### Error Handling

Use `thiserror` for custom error types. Propagate with `?`. Never `unwrap()` or `expect()` in production code.

```rust
#[derive(thiserror::Error, Debug)]
pub enum MotorError {
    #[error("motor channel {channel} out of range (valid: 0–3)")]
    InvalidChannel { channel: u8 },

    #[error("HAT hardware error: {0}")]
    Hat(#[from] HatError),
}

pub fn set_speed(channel: u8, speed_pct: f64) -> Result<(), MotorError> {
    if channel > 3 {
        return Err(MotorError::InvalidChannel { channel });
    }
    // ...
    Ok(())
}
```

`unwrap()` is **only** allowed inside `#[cfg(test)]` blocks.

### Logging

Use `tracing` macros with structured fields. Include relevant context in every log event.

```rust
tracing::info!(channel, pulse_us, "servo pulse set");
tracing::warn!(motor, speed_pct, elapsed_ms, "motor TTL watchdog fired");
tracing::error!(err = %e, "I2C write failed");
```

Levels:
- `error!` — hardware failures, IPC parse errors
- `warn!` — TTL watchdog fires, unexpected-but-recoverable states
- `info!` — successful operations at IPC method granularity
- `debug!` — register-level detail, useful for hardware debugging

### Async

Use `tokio`. Spawn one task per IPC client connection. Use `Arc<tokio::sync::Mutex<T>>` for shared state.

**Critical:** Drop `Mutex` guards before any `.await` point to prevent deadlocks.

```rust
// CORRECT
let value = {
    let guard = mutex.lock().await;
    guard.value
};
some_async_fn().await;

// WRONG — guard held across await
let _guard = mutex.lock().await;
some_async_fn().await;  // deadlock risk
```

### Naming

- Functions and variables: `snake_case`
- Types, traits, enums: `PascalCase`
- Constants: `SCREAMING_SNAKE_CASE`
- Module names: `snake_case`
- No leading `_` except to suppress unused warnings in test helpers

### Documentation

All public items (`pub fn`, `pub struct`, `pub enum`, `pub trait`) must have `///` doc comments.
Modules must have a `//!` module-level comment explaining purpose.

```rust
//! Hat GPIO control: named pin abstraction over rppal.
//!
//! Maps logical HAT pin names (D4, D5, MCURST, SW, LED) to BCM numbers.
//! Hardware access goes through the `GpioBus` trait, enabling test mocking.

/// Sets a GPIO output pin high or low.
///
/// # Errors
/// Returns `GpioError::Rppal` if the underlying rppal write fails.
pub fn write_gpio_pin(gpio: &HatGpio, pin: GpioPin, high: bool) -> Result<(), GpioError> {
```

### Unsafe

No `unsafe` unless absolutely required for hardware access. Always add a `// SAFETY:` comment immediately before the block.

```rust
// SAFETY: `ptr` is non-null and valid for the duration of this function;
//         ownership is not transferred.
unsafe { some_ffi_call(ptr) }
```

### Testing

- Unit tests: `#[cfg(test)] mod tests` inside each module
- Use mock I2C/GPIO traits — never real hardware in unit tests
- Integration tests: `tests/` directory, use Unix socket
- All tests must pass on x86_64 Linux without a Raspberry Pi

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::hat::i2c::MockI2c;

    #[test]
    fn test_set_motor_speed_clamps() {
        let i2c = MockI2c::new();
        // ...
        assert_eq!(result, Ok(()));
    }
}
```

### Build Checks

All Rust code must pass:
```bash
cargo test
cargo clippy -- -D warnings
cargo fmt --check
```

---

## Python (nomothetic)

### Type Hints

All public functions must have complete type hints. Use `Optional[T]` for nullable params and `-> None` when nothing is returned.

```python
def capture_frame(path: str, width: int = 1920, height: int = 1080) -> bytes:
    ...

def set_servo_angle(channel: int, angle: float) -> ServoResult:
    ...
```

### Error Handling

Use specific exception classes. Validate inputs early and raise with descriptive messages.

```python
class ServoError(Exception):
    def __init__(self, channel: int, reason: str) -> None:
        self.channel = channel
        self.reason = reason
        super().__init__(f"Servo {channel}: {reason}")

def set_servo(channel: int, angle: float) -> None:
    if not 0 <= channel <= 11:
        raise ServoError(channel, f"channel out of range (0–11)")
    if not -90.0 <= angle <= 90.0:
        raise ServoError(channel, f"angle {angle} out of range (-90–90)")
```

### Conditional Imports (Pi-only Libraries)

All hardware libraries that are unavailable outside the Pi must be imported conditionally.

```python
try:
    from picamera2 import Picamera2
    _PICAMERA2_AVAILABLE = True
except ImportError:
    Picamera2 = None  # type: ignore
    _PICAMERA2_AVAILABLE = False
```

Test code confirms the guard works on non-Pi by running without the library installed.

### Secrets and Configuration

- NO hardcoded passwords, API keys, tokens, or certificate paths in source
- Use environment variables or config files in `.gitignore`
- TLS certificates: paths from config, not embedded strings

### Testing

```python
# Mock hardware at the module import level
@patch('nomothetic.camera.Picamera2')
def test_capture_returns_bytes(mock_cam_class):
    mock_cam = MagicMock()
    mock_cam_class.return_value = mock_cam
    mock_cam.capture_array.return_value = np.zeros((480, 640, 3), dtype=np.uint8)
    # ...
```

Every new public function needs at least one test. New API endpoints need tests for success, validation failure, and hardware-unavailable cases.

Run test suite:
```bash
pytest -v
black --check .
ruff check .
```

### Documentation

All public functions and classes must have docstrings.

```python
def read_ultrasonic() -> UltrasonicResult:
    """Read distance from the HC-SR04 ultrasonic sensor via nomopractic IPC.

    Returns:
        UltrasonicResult with distance_cm field.

    Raises:
        HatError: if the IPC call fails or the daemon is unreachable.
    """
```

### Formatting

- Formatter: `black` (line length 88, default config)
- Linter: `ruff` (default config, no disables without justification)
- No bare `except:` — always catch specific exception types

---

## Cross-Repo: IPC Schema Changes

When modifying the IPC interface, updates are required in **both repos simultaneously**:

| Change | nomopractic file | nomothetic file(s) |
|--------|------------------|--------------------|
| New method | `src/ipc/schema.rs`, `src/ipc/handler.rs` | `nomothetic/src/nomothetic/hat.py`, `docs/hat_ipc_schema.md` |
| New param field | `src/ipc/schema.rs` | `nomothetic/src/nomothetic/hat.py`, `docs/hat_ipc_schema.md` |
| New error code | `src/ipc/schema.rs` | `nomothetic/src/nomothetic/hat.py` |
| New result field | `src/ipc/schema.rs` | `nomothetic/src/nomothetic/hat.py`, `docs/hat_ipc_schema.md` |

The `hat_ipc_schema.md` file is the **authoritative human-readable schema**. It must always reflect the current implementation.

---

## Architecture Decision Records (ADRs)

Create an ADR in `docs/adr/` when:
- Choosing between two reasonable approaches (e.g. rppal vs pigpio)
- Accepting a known trade-off (e.g. TTL lease for hardware safety)
- Making a choice that future developers might question

Filename: `NNN-short-title.md`  
Format: Status, Context, Decision, Consequences.
