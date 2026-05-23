# Spectre Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Cargo workspace with three Rust-on-ESP-IDF crates (`spectre-common`, `spectre-env`, `spectre-pc-power`) targeting ESP32-C3 SuperMini, exposing a small HTTP API per the design spec at `docs/superpowers/specs/2026-05-08-esp-env-monitor-design.md`.

**Architecture:** Cargo workspace at `/home/fromml/Projects/spectre/`. One shared library crate (`spectre-common`) holds Wi-Fi join, mDNS, HTTP server scaffolding, JSON envelope, error type. Two binary crates wire that up to their specific I/O: `spectre-env` reads BME280 + BH1750 + LM393 reed and serves `GET /api/v1/measurements`; `spectre-pc-power` drives one optocoupler GPIO and serves `POST /api/v1/power-events`. Pull-only HTTP, no auth, plain HTTP, mDNS hostnames.

**Tech Stack:** Rust nightly + RISC-V toolchain (`riscv32imc-unknown-none-elf`-via-ESP-IDF target `riscv32imc-esp-espidf`), `esp-idf-svc`, `esp-idf-hal`, `esp-idf-sys`, `embedded-hal` 1.0, `embedded-hal-bus`, `serde` + `serde_json`, `log` + `thiserror`, `toml-cfg`, `embedded-hal-mock` for tests, `espflash` + `cargo-espflash` for the host workflow.

**Hardware availability gate:** The MT3608 boost modules and 1N5817 Schottky diodes arrive 2026-05-12. Tasks 0 to 23 require only an ESP32-C3 SuperMini powered via USB-C (no battery, no boost). Tasks 24 to 28 (battery + solar + boost integration smoke) need the boost modules.

---

## File Structure

```
/home/fromml/Projects/spectre/
+-- Cargo.toml                         (workspace root)
+-- rust-toolchain.toml                (channel = nightly, target = riscv32imc-esp-espidf)
+-- .cargo/config.toml                 (target alias + runner = espflash flash --monitor)
+-- sdkconfig.defaults                 (ESP-IDF kconfig overrides)
+-- .gitignore
+-- justfile                           (recipes: just flash env, just monitor, just smoke env1)
+-- scripts/
|   +-- smoke-env.sh                   (curl-based smoke checks for an env node)
|   +-- smoke-pc.sh                    (curl-based smoke checks for the pc-power node)
+-- docs/
|   +-- superpowers/
|       +-- specs/2026-05-08-esp-env-monitor-design.md
|       +-- plans/2026-05-08-spectre-implementation.md  (this file)
|
+-- common/                            (crate spectre-common)
|   +-- Cargo.toml
|   +-- src/
|       +-- lib.rs
|       +-- error.rs
|       +-- config.rs
|       +-- wifi.rs
|       +-- mdns.rs
|       +-- http.rs
|
+-- env-node/                          (crate spectre-env, binary)
|   +-- Cargo.toml
|   +-- build.rs
|   +-- cfg.toml
|   +-- src/
|       +-- main.rs
|       +-- handlers.rs
|       +-- sensors/
|           +-- mod.rs
|           +-- bme280.rs
|           +-- bh1750.rs
|           +-- reed.rs
|
+-- pc-power-node/                     (crate spectre-pc-power, binary)
    +-- Cargo.toml
    +-- build.rs
    +-- cfg.toml
    +-- src/
        +-- main.rs
        +-- handlers.rs
        +-- relay.rs
```

### File responsibilities

| File | Responsibility |
|---|---|
| `common/src/error.rs` | `AppError` enum (Wifi, Http, I2c, GpioBusy, BodyValidation, BodyParse) + `Result<T>` alias |
| `common/src/config.rs` | `NodeConfig` struct (hostname, ssid, pass) loaded via `toml-cfg` |
| `common/src/wifi.rs` | `connect_wifi(modem, sysloop, nvs, cfg) -> Result<EspWifi>` with retry |
| `common/src/mdns.rs` | `advertise(hostname) -> Result<EspMdns>` |
| `common/src/http.rs` | `build_server() -> Result<EspHttpServer>`, `respond_data<T: Serialize>(req, status, body)`, `respond_error(req, status, code, message)` |
| `common/src/lib.rs` | re-exports |
| `env-node/src/sensors/bme280.rs` | I2C driver (auto-probe 0x76/0x77), `read() -> Result<Bme280Reading>` |
| `env-node/src/sensors/bh1750.rs` | I2C driver (auto-probe 0x23/0x5C), `read_lux() -> Result<f32>` |
| `env-node/src/sensors/reed.rs` | `Reed { gpio, active_low: bool }`, `is_open() -> bool` |
| `env-node/src/sensors/mod.rs` | `Sensors { i2c: Mutex<I2cDriver>, reed: Reed, bme: Bme280, bh: Bh1750 }`, `read_all() -> Measurements` |
| `env-node/src/handlers.rs` | `register_routes(server, sensors)` registering `GET /api/v1/measurements`, `GET /api/v1/health` |
| `env-node/src/main.rs` | boot: init logger, parse cfg, init sensors, wifi, mdns, http server, park |
| `pc-power-node/src/relay.rs` | `OptoRelay { gpio, busy: AtomicBool }`, `try_press(duration_ms) -> Result<()>` |
| `pc-power-node/src/handlers.rs` | `register_routes(server, relay)` registering `POST /api/v1/power-events`, `GET /api/v1/health` |
| `pc-power-node/src/main.rs` | boot: init logger, parse cfg, init relay, wifi, mdns, http server, park |

---

## Phase 0: Toolchain and workspace scaffold

### Task 0: Verify host toolchain and install ESP-IDF Rust toolchain

**Files:**
- None yet (host setup only)

- [ ] **Step 1: Verify rustup is installed**

Run: `rustup --version`
Expected: output like `rustup 1.27.x ...`. If not installed, install from https://rustup.rs.

- [ ] **Step 2: Install the ESP-IDF Rust toolchain via espup**

Run:
```fish
cargo install espup --locked
espup install --targets esp32c3
```
Expected: espup installs LLVM, Xtensa-free RISC-V toolchain, and ESP-IDF dependencies. On Arch Linux you may also need `pacman -S libusb` and `python-pip cmake ninja dfu-util`.

- [ ] **Step 3: Source the export script**

Run: `source ~/export-esp.sh` (path printed by espup at end).
Add to `~/.config/fish/conf.d/esp.fish`:
```fish
test -f ~/export-esp.sh; and bash -c "source ~/export-esp.sh; env" | string match -er '^(IDF_|LIBCLANG_|CARGO_TARGET_)' | source
```
(Or simpler: source it manually each session.)

- [ ] **Step 4: Install espflash + cargo-espflash**

Run: `cargo install espflash cargo-espflash --locked`
Expected: both binaries available on PATH.

- [ ] **Step 5: Plug in an ESP32-C3 SuperMini, confirm enumeration**

Run: `espflash board-info`
Expected: prints chip type `esp32c3`, MAC address, flash size. If permission denied, add user to `uucp` group: `sudo usermod -aG uucp $USER` (Arch) and re-login.

- [ ] **Step 6: Commit nothing yet (no files created)**

This task is environment setup only. Proceed to Task 1.

---

### Task 1: Initialize git, .gitignore, and the empty Cargo workspace

**Files:**
- Create: `Cargo.toml`
- Create: `.gitignore`
- Create: `rust-toolchain.toml`
- Create: `.cargo/config.toml`
- Create: `sdkconfig.defaults`

- [ ] **Step 1: Initialize git in the project root**

Run:
```fish
cd /home/fromml/Projects/spectre
git init -b main
```

- [ ] **Step 2: Write .gitignore**

Create `.gitignore`:
```gitignore
target/
**/target/
.embuild/
**/.embuild/
*.log
*.bak
.idea/
.vscode/
.DS_Store
```

- [ ] **Step 3: Write workspace Cargo.toml**

Create `Cargo.toml`:
```toml
[workspace]
resolver = "2"
members = [
    "common",
    "env-node",
    "pc-power-node",
]

[workspace.package]
edition = "2021"
rust-version = "1.77"
license = "MIT OR Apache-2.0"

[workspace.dependencies]
# ESP-IDF stack
esp-idf-svc = { version = "0.49", default-features = false, features = ["std", "binstart"] }
esp-idf-hal = { version = "0.44" }
esp-idf-sys = { version = "0.35", features = ["binstart"] }

# embedded-hal 1.0
embedded-hal = "1.0"
embedded-hal-bus = "0.2"

# Drivers
bme280-rs = "0.3"   # confirm version at task time; swap to `bme280` if needed
bh1750 = "0.1"

# Standard
serde = { version = "1", features = ["derive"] }
serde_json = "1"
log = "0.4"
thiserror = "1"
anyhow = "1"
toml-cfg = "0.2"

[profile.release]
opt-level = "s"
lto = "fat"
codegen-units = 1
```

- [ ] **Step 4: Write rust-toolchain.toml**

Create `rust-toolchain.toml`:
```toml
[toolchain]
channel = "nightly-2026-04-15"
components = ["rust-src"]
targets = ["riscv32imc-unknown-none-elf"]
```
(If 2026-04-15 nightly is unavailable when you run this, pick the most recent stable nightly — record it.)

- [ ] **Step 5: Write .cargo/config.toml**

Create `.cargo/config.toml`:
```toml
[build]
target = "riscv32imc-esp-espidf"

[target.riscv32imc-esp-espidf]
linker = "ldproxy"
runner = "espflash flash --monitor"
rustflags = ["--cfg", "espidf_time64"]

[unstable]
build-std = ["std", "panic_abort"]
```

- [ ] **Step 6: Write sdkconfig.defaults**

Create `sdkconfig.defaults`:
```
CONFIG_ESP_TASK_WDT_TIMEOUT_S=10
CONFIG_LOG_DEFAULT_LEVEL_INFO=y
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"
CONFIG_ESP_INT_WDT=y
CONFIG_ESP_TASK_WDT_INIT=y
CONFIG_FREERTOS_HZ=1000
```

- [ ] **Step 7: Commit**

Run:
```fish
git add Cargo.toml .gitignore rust-toolchain.toml .cargo/config.toml sdkconfig.defaults
git commit -m "chore: initialize cargo workspace + esp-idf toolchain config"
```

---

### Task 2: Skeleton crate `spectre-common`

**Files:**
- Create: `common/Cargo.toml`
- Create: `common/src/lib.rs`

- [ ] **Step 1: Write common/Cargo.toml**

Create `common/Cargo.toml`:
```toml
[package]
name = "spectre-common"
version = "0.1.0"
edition.workspace = true
rust-version.workspace = true
license.workspace = true

[dependencies]
esp-idf-svc = { workspace = true }
esp-idf-hal = { workspace = true }
esp-idf-sys = { workspace = true }
embedded-hal = { workspace = true }
serde = { workspace = true }
serde_json = { workspace = true }
log = { workspace = true }
thiserror = { workspace = true }
anyhow = { workspace = true }
toml-cfg = { workspace = true }
```

- [ ] **Step 2: Write common/src/lib.rs (placeholder re-exports)**

Create `common/src/lib.rs`:
```rust
//! Spectre shared infrastructure: error type, config, wifi, mdns, http scaffold.

pub mod error;
pub mod config;
pub mod wifi;
pub mod mdns;
pub mod http;

pub use error::{AppError, Result};
pub use config::NodeConfig;
```

- [ ] **Step 3: Stub the four submodules so the crate builds**

Create `common/src/error.rs`:
```rust
pub type Result<T> = core::result::Result<T, AppError>;

#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("placeholder")]
    Placeholder,
}
```

Create `common/src/config.rs`:
```rust
pub struct NodeConfig;
```

Create `common/src/wifi.rs`:
```rust
// stub
```

Create `common/src/mdns.rs`:
```rust
// stub
```

Create `common/src/http.rs`:
```rust
// stub
```

- [ ] **Step 4: Compile-check**

Run: `cargo check -p spectre-common`
Expected: PASS (no errors). Warnings about unused code are acceptable.

- [ ] **Step 5: Commit**

Run:
```fish
git add common/
git commit -m "feat(common): scaffold spectre-common crate with module stubs"
```

---

### Task 3: Skeleton crate `spectre-env`

**Files:**
- Create: `env-node/Cargo.toml`
- Create: `env-node/build.rs`
- Create: `env-node/cfg.toml`
- Create: `env-node/src/main.rs`

- [ ] **Step 1: Write env-node/Cargo.toml**

Create `env-node/Cargo.toml`:
```toml
[package]
name = "spectre-env"
version = "0.1.0"
edition.workspace = true
rust-version.workspace = true
license.workspace = true

[dependencies]
spectre-common = { path = "../common" }
esp-idf-svc = { workspace = true }
esp-idf-hal = { workspace = true }
esp-idf-sys = { workspace = true }
embedded-hal = { workspace = true }
embedded-hal-bus = { workspace = true }
bme280-rs = { workspace = true }
bh1750 = { workspace = true }
serde = { workspace = true }
serde_json = { workspace = true }
log = { workspace = true }
thiserror = { workspace = true }
anyhow = { workspace = true }
toml-cfg = { workspace = true }

[build-dependencies]
embuild = "0.32"
```

- [ ] **Step 2: Write env-node/build.rs**

Create `env-node/build.rs`:
```rust
fn main() {
    embuild::espidf::sysenv::output();
}
```

- [ ] **Step 3: Write env-node/cfg.toml (placeholder values)**

Create `env-node/cfg.toml`:
```toml
[spectre-env]
hostname = "spectre-env1"
wifi_ssid = "your_ssid_here"
wifi_pass = "your_pass_here"
```

- [ ] **Step 4: Write env-node/src/main.rs (boot stub)**

Create `env-node/src/main.rs`:
```rust
fn main() -> anyhow::Result<()> {
    esp_idf_svc::sys::link_patches();
    esp_idf_svc::log::EspLogger::initialize_default();
    log::info!("spectre-env boot stub (Task 3)");
    loop {
        std::thread::sleep(std::time::Duration::from_secs(60));
    }
}
```

- [ ] **Step 5: Build it**

Run: `cargo build -p spectre-env --release`
Expected: PASS, produces `target/riscv32imc-esp-espidf/release/spectre-env`. First build downloads ESP-IDF (slow, ~5–10 min).

- [ ] **Step 6: Flash and verify boot log**

Plug in ESP32-C3 via USB-C. Run:
```fish
cargo run -p spectre-env --release
```
Expected: espflash flashes, monitor opens, log line `spectre-env boot stub (Task 3)` appears within 5 s. `Ctrl-]` to exit monitor.

- [ ] **Step 7: Commit**

Run:
```fish
git add env-node/
git commit -m "feat(env-node): scaffold spectre-env binary with boot stub"
```

---

### Task 4: Skeleton crate `spectre-pc-power`

**Files:**
- Create: `pc-power-node/Cargo.toml`
- Create: `pc-power-node/build.rs`
- Create: `pc-power-node/cfg.toml`
- Create: `pc-power-node/src/main.rs`

- [ ] **Step 1: Write pc-power-node/Cargo.toml**

Create `pc-power-node/Cargo.toml`:
```toml
[package]
name = "spectre-pc-power"
version = "0.1.0"
edition.workspace = true
rust-version.workspace = true
license.workspace = true

[dependencies]
spectre-common = { path = "../common" }
esp-idf-svc = { workspace = true }
esp-idf-hal = { workspace = true }
esp-idf-sys = { workspace = true }
serde = { workspace = true }
serde_json = { workspace = true }
log = { workspace = true }
thiserror = { workspace = true }
anyhow = { workspace = true }
toml-cfg = { workspace = true }

[build-dependencies]
embuild = "0.32"
```

- [ ] **Step 2: Write pc-power-node/build.rs**

Create `pc-power-node/build.rs`:
```rust
fn main() {
    embuild::espidf::sysenv::output();
}
```

- [ ] **Step 3: Write pc-power-node/cfg.toml**

Create `pc-power-node/cfg.toml`:
```toml
[spectre-pc-power]
hostname = "spectre-pc"
wifi_ssid = "your_ssid_here"
wifi_pass = "your_pass_here"
```

- [ ] **Step 4: Write pc-power-node/src/main.rs (boot stub)**

Create `pc-power-node/src/main.rs`:
```rust
fn main() -> anyhow::Result<()> {
    esp_idf_svc::sys::link_patches();
    esp_idf_svc::log::EspLogger::initialize_default();
    log::info!("spectre-pc-power boot stub (Task 4)");
    loop {
        std::thread::sleep(std::time::Duration::from_secs(60));
    }
}
```

- [ ] **Step 5: Build it**

Run: `cargo build -p spectre-pc-power --release`
Expected: PASS.

- [ ] **Step 6: Commit**

Run:
```fish
git add pc-power-node/
git commit -m "feat(pc-power-node): scaffold spectre-pc-power binary with boot stub"
```

---

## Phase 1: `spectre-common` foundations

### Task 5: Implement `AppError` and `Result` alias (TDD)

**Files:**
- Modify: `common/src/error.rs`

- [ ] **Step 1: Write the failing tests**

Replace `common/src/error.rs` contents with:
```rust
use thiserror::Error;

pub type Result<T> = core::result::Result<T, AppError>;

#[derive(Debug, Error)]
pub enum AppError {
    #[error("wifi failure: {0}")]
    Wifi(String),

    #[error("http server failure: {0}")]
    Http(String),

    #[error("i2c bus failure: {0}")]
    I2c(String),

    #[error("gpio busy")]
    GpioBusy,

    #[error("body validation failed: {0}")]
    BodyValidation(String),

    #[error("body parse failed: {0}")]
    BodyParse(String),

    #[error("config error: {0}")]
    Config(String),

    #[error(transparent)]
    Other(#[from] anyhow::Error),
}

impl AppError {
    /// HTTP status code this error maps to.
    pub fn http_status(&self) -> u16 {
        match self {
            AppError::BodyParse(_) => 400,
            AppError::BodyValidation(_) => 422,
            AppError::GpioBusy => 429,
            AppError::I2c(_) => 503,
            _ => 500,
        }
    }

    /// Short machine-readable code (matches HTTP status for now).
    pub fn code(&self) -> u16 {
        self.http_status()
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn body_parse_maps_to_400() {
        assert_eq!(AppError::BodyParse("x".into()).http_status(), 400);
    }

    #[test]
    fn body_validation_maps_to_422() {
        assert_eq!(AppError::BodyValidation("x".into()).http_status(), 422);
    }

    #[test]
    fn gpio_busy_maps_to_429() {
        assert_eq!(AppError::GpioBusy.http_status(), 429);
    }

    #[test]
    fn i2c_maps_to_503() {
        assert_eq!(AppError::I2c("x".into()).http_status(), 503);
    }

    #[test]
    fn wifi_maps_to_500() {
        assert_eq!(AppError::Wifi("x".into()).http_status(), 500);
    }

    #[test]
    fn code_matches_status() {
        let e = AppError::GpioBusy;
        assert_eq!(e.code(), e.http_status());
    }
}
```

- [ ] **Step 2: Run tests on host (they will fail because tests target ESP)**

Run:
```fish
cargo test -p spectre-common --target x86_64-unknown-linux-gnu --lib error
```
Expected: tests run on host. May need `--no-default-features` if esp-idf-svc feature gates fail. If host-test fails due to ESP-IDF deps, use `cargo check` and trust the unit tests run inside on-device test runs.
**Fallback if host tests are blocked by ESP-IDF deps:** skip on-host execution, rely on on-device cargo test in Task 8.

- [ ] **Step 3: Commit**

Run:
```fish
git add common/src/error.rs
git commit -m "feat(common): error type with HTTP status mapping"
```

---

### Task 6: Implement `NodeConfig` via `toml-cfg`

**Files:**
- Modify: `common/src/config.rs`

- [ ] **Step 1: Write the implementation**

Replace `common/src/config.rs` contents with:
```rust
//! Node configuration loaded from a per-binary `cfg.toml` at compile time
//! via the `toml-cfg` crate. Each binary defines its own table.

/// Compile-time-checked config for env-node.
#[toml_cfg::toml_config]
pub struct EnvNodeConfig {
    #[default("spectre-env-x")]
    pub hostname: &'static str,
    #[default("")]
    pub wifi_ssid: &'static str,
    #[default("")]
    pub wifi_pass: &'static str,
}

/// Compile-time-checked config for pc-power-node.
#[toml_cfg::toml_config]
pub struct PcPowerNodeConfig {
    #[default("spectre-pc-x")]
    pub hostname: &'static str,
    #[default("")]
    pub wifi_ssid: &'static str,
    #[default("")]
    pub wifi_pass: &'static str,
}
```

- [ ] **Step 2: Compile-check**

Run: `cargo check -p spectre-common`
Expected: PASS.

- [ ] **Step 3: Commit**

Run:
```fish
git add common/src/config.rs
git commit -m "feat(common): NodeConfig structs via toml-cfg"
```

---

### Task 7: Implement `wifi::connect_wifi`

**Files:**
- Modify: `common/src/wifi.rs`

- [ ] **Step 1: Write wifi.rs**

Replace `common/src/wifi.rs` contents with:
```rust
use esp_idf_svc::eventloop::EspSystemEventLoop;
use esp_idf_svc::hal::peripheral::Peripheral;
use esp_idf_svc::nvs::EspDefaultNvsPartition;
use esp_idf_svc::wifi::{AuthMethod, BlockingWifi, ClientConfiguration, Configuration, EspWifi};
use log::info;

use crate::error::{AppError, Result};

/// Bring up Wi-Fi as a station, blocking until associated and IP-bound.
pub fn connect_wifi(
    modem: impl Peripheral<P = esp_idf_svc::hal::modem::Modem> + 'static,
    sysloop: EspSystemEventLoop,
    nvs: EspDefaultNvsPartition,
    ssid: &str,
    pass: &str,
) -> Result<BlockingWifi<EspWifi<'static>>> {
    let mut wifi = BlockingWifi::wrap(
        EspWifi::new(modem, sysloop.clone(), Some(nvs))
            .map_err(|e| AppError::Wifi(format!("EspWifi::new: {e}")))?,
        sysloop,
    )
    .map_err(|e| AppError::Wifi(format!("BlockingWifi::wrap: {e}")))?;

    let cfg = Configuration::Client(ClientConfiguration {
        ssid: ssid
            .try_into()
            .map_err(|_| AppError::Wifi("ssid too long".into()))?,
        password: pass
            .try_into()
            .map_err(|_| AppError::Wifi("password too long".into()))?,
        auth_method: AuthMethod::WPA2Personal,
        ..Default::default()
    });
    wifi.set_configuration(&cfg)
        .map_err(|e| AppError::Wifi(format!("set_configuration: {e}")))?;
    wifi.start()
        .map_err(|e| AppError::Wifi(format!("start: {e}")))?;
    wifi.connect()
        .map_err(|e| AppError::Wifi(format!("connect: {e}")))?;
    wifi.wait_netif_up()
        .map_err(|e| AppError::Wifi(format!("wait_netif_up: {e}")))?;

    let ip = wifi
        .wifi()
        .sta_netif()
        .get_ip_info()
        .map_err(|e| AppError::Wifi(format!("get_ip_info: {e}")))?
        .ip;
    info!("wifi: connected to '{ssid}', ip = {ip}");
    Ok(wifi)
}
```

- [ ] **Step 2: Compile-check the workspace**

Run: `cargo check`
Expected: PASS.

- [ ] **Step 3: Commit**

Run:
```fish
git add common/src/wifi.rs
git commit -m "feat(common): wifi::connect_wifi with blocking STA bring-up"
```

---

### Task 8: Implement `mdns::advertise`

**Files:**
- Modify: `common/src/mdns.rs`

- [ ] **Step 1: Write mdns.rs**

Replace `common/src/mdns.rs` contents with:
```rust
use esp_idf_svc::mdns::EspMdns;
use log::info;

use crate::error::{AppError, Result};

/// Initialize mDNS responder and announce the given hostname.
/// The node is then reachable as `<hostname>.local`.
pub fn advertise(hostname: &str) -> Result<EspMdns> {
    let mut mdns = EspMdns::take().map_err(|e| AppError::Wifi(format!("mdns take: {e}")))?;
    mdns.set_hostname(hostname)
        .map_err(|e| AppError::Wifi(format!("mdns set_hostname: {e}")))?;
    mdns.set_instance_name(hostname)
        .map_err(|e| AppError::Wifi(format!("mdns set_instance_name: {e}")))?;
    mdns.add_service(None, "_http", "_tcp", 80, &[])
        .map_err(|e| AppError::Wifi(format!("mdns add_service: {e}")))?;
    info!("mdns: announced '{hostname}.local'");
    Ok(mdns)
}
```

- [ ] **Step 2: Compile-check**

Run: `cargo check`
Expected: PASS.

- [ ] **Step 3: Commit**

Run:
```fish
git add common/src/mdns.rs
git commit -m "feat(common): mdns::advertise"
```

---

### Task 9: Implement `http::build_server` and JSON helpers

**Files:**
- Modify: `common/src/http.rs`

- [ ] **Step 1: Write http.rs**

Replace `common/src/http.rs` contents with:
```rust
use esp_idf_svc::http::server::{EspHttpConnection, EspHttpServer, Request, Response};
use esp_idf_svc::http::Method;
use serde::Serialize;
use serde_json::json;

use crate::error::{AppError, Result};

/// Construct an HTTP server bound to port 80 with sensible defaults.
pub fn build_server() -> Result<EspHttpServer<'static>> {
    let cfg = esp_idf_svc::http::server::Configuration {
        http_port: 80,
        ..Default::default()
    };
    EspHttpServer::new(&cfg).map_err(|e| AppError::Http(format!("EspHttpServer::new: {e}")))
}

/// Send a `{"data": ...}` envelope with the given HTTP status.
pub fn respond_data<T: Serialize>(
    req: Request<&mut EspHttpConnection>,
    status: u16,
    body: &T,
) -> std::result::Result<(), Box<dyn std::error::Error>> {
    let json = serde_json::to_vec(&json!({ "data": body }))?;
    let mut resp = req.into_response(
        status,
        None,
        &[("Content-Type", "application/json; charset=utf-8")],
    )?;
    resp.write_all(&json)?;
    Ok(())
}

/// Send a `{"error": {"code": N, "message": "..."}}` envelope.
pub fn respond_error(
    req: Request<&mut EspHttpConnection>,
    status: u16,
    code: u16,
    message: &str,
) -> std::result::Result<(), Box<dyn std::error::Error>> {
    let json = serde_json::to_vec(&json!({
        "error": { "code": code, "message": message }
    }))?;
    let mut resp = req.into_response(
        status,
        None,
        &[("Content-Type", "application/json; charset=utf-8")],
    )?;
    resp.write_all(&json)?;
    Ok(())
}

/// Map an `AppError` into an HTTP error response.
pub fn respond_app_error(
    req: Request<&mut EspHttpConnection>,
    err: &AppError,
) -> std::result::Result<(), Box<dyn std::error::Error>> {
    respond_error(req, err.http_status(), err.code(), &err.to_string())
}

// re-export Method so callers don't need a direct esp-idf-svc::http import
pub use Method as HttpMethod;
```

- [ ] **Step 2: Compile-check**

Run: `cargo check`
Expected: PASS. Adjust the `Request` / `Response` API calls if `esp-idf-svc 0.49` exposes a slightly different signature; the trait is stable in spirit but argument names may differ.

- [ ] **Step 3: Commit**

Run:
```fish
git add common/src/http.rs
git commit -m "feat(common): http::build_server + JSON envelope helpers"
```

---

## Phase 2: `spectre-env` boot path and `/health`

### Task 10: Wire `spectre-env` main: wifi + mdns + http server + /health

**Files:**
- Modify: `env-node/src/main.rs`
- Create: `env-node/src/handlers.rs`

- [ ] **Step 1: Write env-node/src/handlers.rs**

Create `env-node/src/handlers.rs`:
```rust
use std::time::Instant;

use esp_idf_svc::http::server::EspHttpServer;
use esp_idf_svc::http::Method;
use serde::Serialize;
use spectre_common::http::respond_data;

#[derive(Serialize)]
struct Health<'a> {
    ok: bool,
    uptime_s: u64,
    node_id: &'a str,
}

pub fn register_health(server: &mut EspHttpServer, node_id: &'static str, boot: Instant) {
    let _ = server.fn_handler(
        "/api/v1/health",
        Method::Get,
        move |req| -> std::result::Result<(), Box<dyn std::error::Error>> {
            let body = Health {
                ok: true,
                uptime_s: boot.elapsed().as_secs(),
                node_id,
            };
            respond_data(req, 200, &body)
        },
    );
}
```

- [ ] **Step 2: Replace env-node/src/main.rs**

Replace `env-node/src/main.rs` with:
```rust
use std::time::Instant;

use esp_idf_svc::eventloop::EspSystemEventLoop;
use esp_idf_svc::hal::peripherals::Peripherals;
use esp_idf_svc::nvs::EspDefaultNvsPartition;
use spectre_common::{
    config::ENV_NODE_CONFIG, http::build_server, mdns::advertise, wifi::connect_wifi,
};

mod handlers;

fn main() -> anyhow::Result<()> {
    esp_idf_svc::sys::link_patches();
    esp_idf_svc::log::EspLogger::initialize_default();
    let boot = Instant::now();
    log::info!("spectre-env starting; hostname={}", ENV_NODE_CONFIG.hostname);

    let peripherals = Peripherals::take()?;
    let sysloop = EspSystemEventLoop::take()?;
    let nvs = EspDefaultNvsPartition::take()?;

    let _wifi = connect_wifi(
        peripherals.modem,
        sysloop,
        nvs,
        ENV_NODE_CONFIG.wifi_ssid,
        ENV_NODE_CONFIG.wifi_pass,
    )?;

    let _mdns = advertise(ENV_NODE_CONFIG.hostname)?;

    let mut server = build_server()?;
    handlers::register_health(&mut server, ENV_NODE_CONFIG.hostname, boot);

    log::info!("spectre-env: serving on http://{}.local/", ENV_NODE_CONFIG.hostname);
    loop {
        std::thread::sleep(std::time::Duration::from_secs(60));
    }
}
```

- [ ] **Step 3: Edit env-node/cfg.toml with real Wi-Fi creds**

Edit `env-node/cfg.toml` and fill in your actual SSID + password. **Do not commit real credentials.** Add `env-node/cfg.toml` to `.gitignore` after this step:
```fish
echo "env-node/cfg.toml" >> .gitignore
echo "pc-power-node/cfg.toml" >> .gitignore
git add .gitignore
git commit -m "chore: gitignore per-binary cfg.toml (contains creds)"
```
Then commit a stripped placeholder `env-node/cfg.toml.example`:
```fish
cp env-node/cfg.toml env-node/cfg.toml.example
sed -i 's/wifi_pass = ".*"/wifi_pass = "REPLACE_ME"/; s/wifi_ssid = ".*"/wifi_ssid = "REPLACE_ME"/' env-node/cfg.toml.example
git add env-node/cfg.toml.example
git commit -m "docs(env-node): add cfg.toml.example template"
```

- [ ] **Step 4: Build**

Run: `cargo build -p spectre-env --release`
Expected: PASS.

- [ ] **Step 5: Flash and watch logs**

Run: `cargo run -p spectre-env --release`
Expected log lines (within ~10 s):
- `spectre-env starting; hostname=spectre-env1`
- `wifi: connected to 'YOUR_SSID', ip = 192.168.x.x`
- `mdns: announced 'spectre-env1.local'`
- `spectre-env: serving on http://spectre-env1.local/`

- [ ] **Step 6: Smoke-check /health from another machine on the LAN**

In a second terminal: `curl -s http://spectre-env1.local/api/v1/health | jq .`
Expected:
```json
{ "data": { "ok": true, "uptime_s": <number>, "node_id": "spectre-env1" } }
```
If mDNS lookup fails on Linux, install `avahi-utils` (`sudo pacman -S avahi`) and start the daemon: `sudo systemctl enable --now avahi-daemon`.

- [ ] **Step 7: Commit**

Run:
```fish
git add env-node/src/main.rs env-node/src/handlers.rs
git commit -m "feat(env-node): boot path + GET /api/v1/health"
```

---

## Phase 3: env sensor drivers

### Task 11: Sensors module skeleton with shared I2C bus

**Files:**
- Create: `env-node/src/sensors/mod.rs`
- Create: `env-node/src/sensors/bme280.rs` (stub)
- Create: `env-node/src/sensors/bh1750.rs` (stub)
- Create: `env-node/src/sensors/reed.rs` (stub)
- Modify: `env-node/src/main.rs`

- [ ] **Step 1: Write env-node/src/sensors/mod.rs**

Create `env-node/src/sensors/mod.rs`:
```rust
use std::sync::Mutex;

use esp_idf_svc::hal::gpio::AnyIOPin;
use esp_idf_svc::hal::i2c::{I2cConfig, I2cDriver};
use esp_idf_svc::hal::peripherals::Peripherals;
use esp_idf_svc::hal::prelude::*;
use serde::Serialize;
use spectre_common::error::{AppError, Result};

pub mod bh1750;
pub mod bme280;
pub mod reed;

/// One reading snapshot returned to clients.
#[derive(Debug, Serialize)]
pub struct Measurements {
    pub temperature_c: f32,
    pub humidity_pct: f32,
    pub pressure_hpa: f32,
    pub lux: f32,
    pub door_open: bool,
}

/// Owns the I2C bus and the reed GPIO. Cheap to clone the inner Arc/Mutex if needed later.
pub struct Sensors {
    i2c: Mutex<I2cDriver<'static>>,
    bme: bme280::Bme280,
    bh: bh1750::Bh1750,
    reed: reed::Reed,
}

impl Sensors {
    /// Construct from peripherals taken in main(). Probes I2C addresses, configures GPIOs.
    pub fn new(peripherals: &mut Peripherals) -> Result<Self> {
        // I2C0 on GPIO5 (SDA) + GPIO6 (SCL), 100 kHz.
        let sda = unsafe { AnyIOPin::new(5) };
        let scl = unsafe { AnyIOPin::new(6) };
        let cfg = I2cConfig::new().baudrate(100.kHz().into());
        let i2c_driver = I2cDriver::new(unsafe { peripherals.i2c0.clone_unchecked() }, sda, scl, &cfg)
            .map_err(|e| AppError::I2c(format!("I2cDriver::new: {e}")))?;
        let i2c = Mutex::new(i2c_driver);

        let bme = bme280::Bme280::probe(&i2c)?;
        let bh = bh1750::Bh1750::probe(&i2c)?;
        let reed = reed::Reed::new(unsafe { AnyIOPin::new(4) }, /*active_low*/ true)?;

        Ok(Self { i2c, bme, bh, reed })
    }

    pub fn read_all(&self) -> Result<Measurements> {
        let mut guard = self
            .i2c
            .lock()
            .map_err(|_| AppError::I2c("i2c mutex poisoned".into()))?;
        let bme_r = self.bme.read(&mut guard)?;
        let lux = self.bh.read_lux(&mut guard)?;
        drop(guard);
        let door_open = self.reed.is_open();
        Ok(Measurements {
            temperature_c: round2(bme_r.temperature_c),
            humidity_pct: round2(bme_r.humidity_pct),
            pressure_hpa: round2(bme_r.pressure_hpa),
            lux: round2(lux),
            door_open,
        })
    }
}

fn round2(x: f32) -> f32 {
    (x * 100.0).round() / 100.0
}
```

- [ ] **Step 2: Stub the three driver files so the crate builds**

Create `env-node/src/sensors/bme280.rs`:
```rust
use std::sync::MutexGuard;

use esp_idf_svc::hal::i2c::I2cDriver;
use spectre_common::error::{AppError, Result};
use std::sync::Mutex;

pub struct Bme280 {
    pub addr: u8,
}

pub struct Bme280Reading {
    pub temperature_c: f32,
    pub humidity_pct: f32,
    pub pressure_hpa: f32,
}

impl Bme280 {
    pub fn probe(_i2c: &Mutex<I2cDriver<'static>>) -> Result<Self> {
        // TODO Task 12
        Err(AppError::I2c("BME280 not yet implemented".into()))
    }
    pub fn read(&self, _guard: &mut MutexGuard<I2cDriver<'static>>) -> Result<Bme280Reading> {
        Err(AppError::I2c("BME280 read not yet implemented".into()))
    }
}
```

Create `env-node/src/sensors/bh1750.rs`:
```rust
use std::sync::{Mutex, MutexGuard};

use esp_idf_svc::hal::i2c::I2cDriver;
use spectre_common::error::{AppError, Result};

pub struct Bh1750 {
    pub addr: u8,
}

impl Bh1750 {
    pub fn probe(_i2c: &Mutex<I2cDriver<'static>>) -> Result<Self> {
        // TODO Task 13
        Err(AppError::I2c("BH1750 not yet implemented".into()))
    }
    pub fn read_lux(&self, _guard: &mut MutexGuard<I2cDriver<'static>>) -> Result<f32> {
        Err(AppError::I2c("BH1750 read not yet implemented".into()))
    }
}
```

Create `env-node/src/sensors/reed.rs`:
```rust
use esp_idf_svc::hal::gpio::{AnyIOPin, PinDriver, Pull};
use spectre_common::error::{AppError, Result};

pub struct Reed {
    driver: PinDriver<'static, AnyIOPin, esp_idf_svc::hal::gpio::Input>,
    active_low: bool,
}

impl Reed {
    pub fn new(pin: AnyIOPin, active_low: bool) -> Result<Self> {
        let mut driver =
            PinDriver::input(pin).map_err(|e| AppError::I2c(format!("reed pin: {e}")))?;
        driver
            .set_pull(Pull::Up)
            .map_err(|e| AppError::I2c(format!("reed pull: {e}")))?;
        Ok(Self { driver, active_low })
    }

    pub fn is_open(&self) -> bool {
        let high = self.driver.is_high();
        // active_low: when DO is low, reed is closed (magnet near). is_open == !closed.
        if self.active_low {
            high
        } else {
            !high
        }
    }
}
```

(The `AppError::I2c` for non-I2C reed errors is ugly. Add a `Gpio(String)` variant to `AppError` later if you want.)

- [ ] **Step 3: Compile-check**

Run: `cargo check -p spectre-env`
Expected: PASS.

- [ ] **Step 4: Commit**

Run:
```fish
git add env-node/src/sensors/
git commit -m "feat(env-node): sensors module skeleton (probe stubs)"
```

---

### Task 12: Implement BME280 driver with address auto-probe

**Files:**
- Modify: `env-node/src/sensors/bme280.rs`
- Modify: `common/Cargo.toml` (if you decide to host the driver helper there; for v1 keep it in env-node)

- [ ] **Step 1: Confirm which BME280 crate works on `embedded-hal` 1.0**

Run:
```fish
cargo search bme280 | head
```
Pick the highest-version crate that supports `embedded-hal = "1"`. As of writing both `bme280-rs` (v0.3+) and `bme280` (v0.5+) work; this plan assumes `bme280-rs`. If the API differs, adapt the calls accordingly.

- [ ] **Step 2: Write the driver**

Replace `env-node/src/sensors/bme280.rs` with:
```rust
use std::sync::{Mutex, MutexGuard};

use bme280_rs::{Bme280 as Driver, Configuration as BmeCfg, Oversampling, SensorMode};
use embedded_hal::i2c::I2c;
use esp_idf_svc::hal::delay::FreeRtos;
use esp_idf_svc::hal::i2c::I2cDriver;
use log::info;
use spectre_common::error::{AppError, Result};

const CANDIDATE_ADDRS: [u8; 2] = [0x76, 0x77];

pub struct Bme280Reading {
    pub temperature_c: f32,
    pub humidity_pct: f32,
    pub pressure_hpa: f32,
}

pub struct Bme280 {
    addr: u8,
}

impl Bme280 {
    pub fn probe(i2c: &Mutex<I2cDriver<'static>>) -> Result<Self> {
        let mut guard = i2c
            .lock()
            .map_err(|_| AppError::I2c("i2c mutex poisoned".into()))?;
        for &addr in CANDIDATE_ADDRS.iter() {
            // Read chip ID register 0xD0; expect 0x60 for BME280.
            let mut id = [0u8; 1];
            if guard.write_read(addr, &[0xD0], &mut id).is_ok() && id[0] == 0x60 {
                info!("bme280: found at 0x{:02X}", addr);
                return Ok(Self { addr });
            }
        }
        Err(AppError::I2c("bme280: not found at 0x76 or 0x77".into()))
    }

    pub fn read(&self, guard: &mut MutexGuard<I2cDriver<'static>>) -> Result<Bme280Reading> {
        let mut driver = Driver::new(I2cAtAddr::new(&mut **guard, self.addr));
        driver
            .init(&mut FreeRtos)
            .map_err(|_| AppError::I2c("bme280 init".into()))?;
        let cfg = BmeCfg::default()
            .with_temperature_oversampling(Oversampling::Oversample1)
            .with_humidity_oversampling(Oversampling::Oversample1)
            .with_pressure_oversampling(Oversampling::Oversample1)
            .with_sensor_mode(SensorMode::Forced);
        driver
            .set_sampling_configuration(cfg)
            .map_err(|_| AppError::I2c("bme280 cfg".into()))?;
        FreeRtos::delay_ms(20);
        let m = driver
            .read_sample()
            .map_err(|_| AppError::I2c("bme280 read".into()))?;
        Ok(Bme280Reading {
            temperature_c: m.temperature.unwrap_or(0.0),
            humidity_pct: m.humidity.unwrap_or(0.0),
            pressure_hpa: m.pressure.unwrap_or(0.0) / 100.0,
        })
    }
}

/// Tiny adapter so the `bme280-rs` driver (which expects an `embedded-hal::i2c::I2c`)
/// can talk to the ESP-IDF I2C driver pinned at a specific address.
struct I2cAtAddr<'a> {
    bus: &'a mut I2cDriver<'static>,
    addr: u8,
}
impl<'a> I2cAtAddr<'a> {
    fn new(bus: &'a mut I2cDriver<'static>, addr: u8) -> Self {
        Self { bus, addr }
    }
}
impl<'a> embedded_hal::i2c::ErrorType for I2cAtAddr<'a> {
    type Error = embedded_hal::i2c::ErrorKind;
}
impl<'a> embedded_hal::i2c::I2c for I2cAtAddr<'a> {
    fn transaction(
        &mut self,
        addr: embedded_hal::i2c::SevenBitAddress,
        ops: &mut [embedded_hal::i2c::Operation<'_>],
    ) -> std::result::Result<(), Self::Error> {
        // The driver wraps with its own `addr`; we ignore the caller-supplied one
        // because we always pin to self.addr.
        let _ = addr;
        for op in ops {
            match op {
                embedded_hal::i2c::Operation::Read(buf) => self
                    .bus
                    .read(self.addr, buf, 1000)
                    .map_err(|_| embedded_hal::i2c::ErrorKind::Other)?,
                embedded_hal::i2c::Operation::Write(buf) => self
                    .bus
                    .write(self.addr, buf, 1000)
                    .map_err(|_| embedded_hal::i2c::ErrorKind::Other)?,
            }
        }
        Ok(())
    }
}
```

- [ ] **Step 3: Compile-check**

Run: `cargo check -p spectre-env`
Expected: PASS. If `bme280-rs` API names differ (Configuration::default, set_sampling_configuration, read_sample) consult its README and adapt; concept stays identical (configure → trigger forced mode → read → decode).

- [ ] **Step 4: Commit**

Run:
```fish
git add env-node/src/sensors/bme280.rs
git commit -m "feat(env-node): BME280 driver with address auto-probe"
```

---

### Task 13: Implement BH1750 driver with address auto-probe

**Files:**
- Modify: `env-node/src/sensors/bh1750.rs`

- [ ] **Step 1: Write the driver**

Replace `env-node/src/sensors/bh1750.rs` with:
```rust
use std::sync::{Mutex, MutexGuard};

use esp_idf_svc::hal::delay::FreeRtos;
use esp_idf_svc::hal::i2c::I2cDriver;
use log::info;
use spectre_common::error::{AppError, Result};

const CANDIDATE_ADDRS: [u8; 2] = [0x23, 0x5C];
const CMD_POWER_ON: u8 = 0x01;
const CMD_ONE_SHOT_HIGH_RES: u8 = 0x20;

pub struct Bh1750 {
    addr: u8,
}

impl Bh1750 {
    pub fn probe(i2c: &Mutex<I2cDriver<'static>>) -> Result<Self> {
        let mut guard = i2c
            .lock()
            .map_err(|_| AppError::I2c("i2c mutex poisoned".into()))?;
        for &addr in CANDIDATE_ADDRS.iter() {
            if guard.write(addr, &[CMD_POWER_ON], 1000).is_ok() {
                info!("bh1750: found at 0x{:02X}", addr);
                return Ok(Self { addr });
            }
        }
        Err(AppError::I2c("bh1750: not found at 0x23 or 0x5C".into()))
    }

    pub fn read_lux(&self, guard: &mut MutexGuard<I2cDriver<'static>>) -> Result<f32> {
        guard
            .write(self.addr, &[CMD_POWER_ON], 1000)
            .map_err(|_| AppError::I2c("bh1750 power on".into()))?;
        guard
            .write(self.addr, &[CMD_ONE_SHOT_HIGH_RES], 1000)
            .map_err(|_| AppError::I2c("bh1750 measure cmd".into()))?;
        FreeRtos::delay_ms(180); // datasheet: max 180 ms for high-res one-shot
        let mut buf = [0u8; 2];
        guard
            .read(self.addr, &mut buf, 1000)
            .map_err(|_| AppError::I2c("bh1750 read".into()))?;
        let raw = u16::from_be_bytes(buf);
        // Resolution: raw / 1.2 = lux (high-res mode, default sensitivity)
        Ok(raw as f32 / 1.2)
    }
}
```

- [ ] **Step 2: Compile-check**

Run: `cargo check -p spectre-env`
Expected: PASS.

- [ ] **Step 3: Commit**

Run:
```fish
git add env-node/src/sensors/bh1750.rs
git commit -m "feat(env-node): BH1750 driver with address auto-probe"
```

---

### Task 14: Wire `Sensors` into `main` (sensors initialised at boot, not yet exposed)

**Files:**
- Modify: `env-node/src/main.rs`

- [ ] **Step 1: Edit main.rs to construct Sensors**

In `env-node/src/main.rs`, add after `nvs` line:
```rust
mod sensors;

// inside main(), after `let nvs = ...`:
let mut peripherals_for_sensors = peripherals; // we still need the modem below; restructure if needed
let sensors = sensors::Sensors::new(&mut peripherals_for_sensors)?;
log::info!("sensors initialised");
```
(The `Peripherals` struct is consumed by `connect_wifi(modem)`. You'll need to take the modem out first, then pass the rest of `peripherals` to `Sensors::new`. Two patterns: split borrows up front, or pull the i2c0/gpio4/gpio5/gpio6 individually before calling connect_wifi. Pick the simpler one in the actual edit.)

- [ ] **Step 2: Build and flash**

Run: `cargo run -p spectre-env --release`
Expected log line: `bme280: found at 0x76`, `bh1750: found at 0x23`, `sensors initialised`.

If a sensor is not wired yet, expect a fatal error from `Sensors::new`. Wire the sensors first (see hardware section of spec) or temporarily make `Sensors::new` log warnings and continue.

- [ ] **Step 3: Commit**

Run:
```fish
git add env-node/src/main.rs
git commit -m "feat(env-node): initialise sensors at boot"
```

---

### Task 15: Implement `GET /api/v1/measurements`

**Files:**
- Modify: `env-node/src/handlers.rs`
- Modify: `env-node/src/main.rs`

- [ ] **Step 1: Add measurements handler**

In `env-node/src/handlers.rs`, add:
```rust
use std::sync::Arc;
use std::time::Instant;

use esp_idf_svc::http::server::EspHttpServer;
use esp_idf_svc::http::Method;
use serde::Serialize;
use spectre_common::http::{respond_app_error, respond_data};

use crate::sensors::Sensors;

#[derive(Serialize)]
struct MeasurementsBody<'a> {
    node_id: &'a str,
    uptime_s: u64,
    temperature_c: f32,
    humidity_pct: f32,
    pressure_hpa: f32,
    lux: f32,
    door_open: bool,
    battery_v: Option<f32>,
}

pub fn register_measurements(
    server: &mut EspHttpServer,
    node_id: &'static str,
    boot: Instant,
    sensors: Arc<Sensors>,
) {
    let _ = server.fn_handler(
        "/api/v1/measurements",
        Method::Get,
        move |req| -> std::result::Result<(), Box<dyn std::error::Error>> {
            match sensors.read_all() {
                Ok(m) => {
                    let body = MeasurementsBody {
                        node_id,
                        uptime_s: boot.elapsed().as_secs(),
                        temperature_c: m.temperature_c,
                        humidity_pct: m.humidity_pct,
                        pressure_hpa: m.pressure_hpa,
                        lux: m.lux,
                        door_open: m.door_open,
                        battery_v: None,
                    };
                    respond_data(req, 200, &body)
                }
                Err(e) => respond_app_error(req, &e),
            }
        },
    );
}
```

- [ ] **Step 2: Wire it in main.rs**

In `env-node/src/main.rs` after constructing `Sensors`:
```rust
let sensors = std::sync::Arc::new(sensors);
handlers::register_measurements(&mut server, ENV_NODE_CONFIG.hostname, boot, sensors.clone());
```

- [ ] **Step 3: Flash and smoke-test**

Run: `cargo run -p spectre-env --release`
Then from another machine:
```fish
curl -s http://spectre-env1.local/api/v1/measurements | jq .
```
Expected:
```json
{
  "data": {
    "node_id": "spectre-env1",
    "uptime_s": 42,
    "temperature_c": 21.43,
    "humidity_pct": 47.2,
    "pressure_hpa": 1013.8,
    "lux": 312.5,
    "door_open": false,
    "battery_v": null
  }
}
```

Sanity check: cup your hand over the BH1750, re-curl, lux drops. Hold magnet near the reed switch, re-curl, `door_open` flips.

- [ ] **Step 4: Verify 503 path**

Disconnect SDA from BME280 with the node still running. Re-curl:
```fish
curl -i -s http://spectre-env1.local/api/v1/measurements
```
Expected: HTTP/1.1 503, body `{"error":{"code":503,"message":"i2c bus failure: ..."}}`. Reconnect SDA after.

- [ ] **Step 5: Commit**

Run:
```fish
git add env-node/src/handlers.rs env-node/src/main.rs
git commit -m "feat(env-node): GET /api/v1/measurements returning live sensor data"
```

---

## Phase 4: `spectre-pc-power` boot path and event endpoint

### Task 16: Wire `spectre-pc-power` main: wifi + mdns + http server + /health

**Files:**
- Modify: `pc-power-node/src/main.rs`
- Create: `pc-power-node/src/handlers.rs`

- [ ] **Step 1: Write pc-power-node/src/handlers.rs**

Create `pc-power-node/src/handlers.rs`:
```rust
use std::time::Instant;

use esp_idf_svc::http::server::EspHttpServer;
use esp_idf_svc::http::Method;
use serde::Serialize;
use spectre_common::http::respond_data;

#[derive(Serialize)]
struct Health<'a> {
    ok: bool,
    uptime_s: u64,
    node_id: &'a str,
}

pub fn register_health(server: &mut EspHttpServer, node_id: &'static str, boot: Instant) {
    let _ = server.fn_handler(
        "/api/v1/health",
        Method::Get,
        move |req| -> std::result::Result<(), Box<dyn std::error::Error>> {
            let body = Health {
                ok: true,
                uptime_s: boot.elapsed().as_secs(),
                node_id,
            };
            respond_data(req, 200, &body)
        },
    );
}
```

- [ ] **Step 2: Replace pc-power-node/src/main.rs**

Replace `pc-power-node/src/main.rs` with:
```rust
use std::time::Instant;

use esp_idf_svc::eventloop::EspSystemEventLoop;
use esp_idf_svc::hal::peripherals::Peripherals;
use esp_idf_svc::nvs::EspDefaultNvsPartition;
use spectre_common::{
    config::PC_POWER_NODE_CONFIG, http::build_server, mdns::advertise, wifi::connect_wifi,
};

mod handlers;

fn main() -> anyhow::Result<()> {
    esp_idf_svc::sys::link_patches();
    esp_idf_svc::log::EspLogger::initialize_default();
    let boot = Instant::now();
    log::info!(
        "spectre-pc-power starting; hostname={}",
        PC_POWER_NODE_CONFIG.hostname
    );

    let peripherals = Peripherals::take()?;
    let sysloop = EspSystemEventLoop::take()?;
    let nvs = EspDefaultNvsPartition::take()?;

    let _wifi = connect_wifi(
        peripherals.modem,
        sysloop,
        nvs,
        PC_POWER_NODE_CONFIG.wifi_ssid,
        PC_POWER_NODE_CONFIG.wifi_pass,
    )?;
    let _mdns = advertise(PC_POWER_NODE_CONFIG.hostname)?;

    let mut server = build_server()?;
    handlers::register_health(&mut server, PC_POWER_NODE_CONFIG.hostname, boot);

    log::info!(
        "spectre-pc-power: serving on http://{}.local/",
        PC_POWER_NODE_CONFIG.hostname
    );
    loop {
        std::thread::sleep(std::time::Duration::from_secs(60));
    }
}
```

- [ ] **Step 3: Edit cfg.toml with real Wi-Fi creds + commit example**

```fish
# Edit pc-power-node/cfg.toml with your SSID + pass.
cp pc-power-node/cfg.toml pc-power-node/cfg.toml.example
sed -i 's/wifi_pass = ".*"/wifi_pass = "REPLACE_ME"/; s/wifi_ssid = ".*"/wifi_ssid = "REPLACE_ME"/' pc-power-node/cfg.toml.example
git add pc-power-node/cfg.toml.example
git commit -m "docs(pc-power-node): add cfg.toml.example template"
```

- [ ] **Step 4: Build, flash, smoke-test /health**

```fish
cargo run -p spectre-pc-power --release
# from other terminal:
curl -s http://spectre-pc.local/api/v1/health | jq .
```
Expected: `{"data":{"ok":true,"uptime_s":..,"node_id":"spectre-pc"}}`.

- [ ] **Step 5: Commit**

```fish
git add pc-power-node/src/main.rs pc-power-node/src/handlers.rs
git commit -m "feat(pc-power-node): boot path + GET /api/v1/health"
```

---

### Task 17: Implement `OptoRelay` with busy guard

**Files:**
- Create: `pc-power-node/src/relay.rs`

- [ ] **Step 1: Write relay.rs**

Create `pc-power-node/src/relay.rs`:
```rust
use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::Arc;
use std::time::Duration;

use esp_idf_svc::hal::delay::FreeRtos;
use esp_idf_svc::hal::gpio::{AnyOutputPin, Output, PinDriver};
use spectre_common::error::{AppError, Result};

pub struct OptoRelay {
    pin: std::sync::Mutex<PinDriver<'static, AnyOutputPin, Output>>,
    busy: AtomicBool,
}

impl OptoRelay {
    pub fn new(pin: AnyOutputPin) -> Result<Arc<Self>> {
        let driver =
            PinDriver::output(pin).map_err(|e| AppError::I2c(format!("relay pin: {e}")))?;
        Ok(Arc::new(Self {
            pin: std::sync::Mutex::new(driver),
            busy: AtomicBool::new(false),
        }))
    }

    /// Hold the optocoupler closed for `duration_ms`. Returns `GpioBusy` if a press is already underway.
    pub fn try_press(&self, duration_ms: u32) -> Result<()> {
        if self
            .busy
            .compare_exchange(false, true, Ordering::SeqCst, Ordering::SeqCst)
            .is_err()
        {
            return Err(AppError::GpioBusy);
        }
        let result = (|| -> Result<()> {
            let mut guard = self
                .pin
                .lock()
                .map_err(|_| AppError::I2c("relay pin mutex poisoned".into()))?;
            guard
                .set_high()
                .map_err(|e| AppError::I2c(format!("set_high: {e}")))?;
            FreeRtos::delay_ms(duration_ms);
            guard
                .set_low()
                .map_err(|e| AppError::I2c(format!("set_low: {e}")))?;
            Ok(())
        })();
        self.busy.store(false, Ordering::SeqCst);
        result
    }
}

/// Pulse durations defined by the API contract.
pub const PRESS_MS: u32 = 200;
pub const FORCE_SHUTDOWN_MS: u32 = 6000;

#[allow(dead_code)]
fn _typecheck() {
    let _ = Duration::from_millis(PRESS_MS as u64);
}
```

- [ ] **Step 2: Compile-check**

Run: `cargo check -p spectre-pc-power`
Expected: PASS.

- [ ] **Step 3: Commit**

Run:
```fish
git add pc-power-node/src/relay.rs
git commit -m "feat(pc-power-node): OptoRelay with atomic busy guard"
```

---

### Task 18: Implement `POST /api/v1/power-events`

**Files:**
- Modify: `pc-power-node/src/handlers.rs`
- Modify: `pc-power-node/src/main.rs`

- [ ] **Step 1: Extend handlers.rs**

Add to `pc-power-node/src/handlers.rs`:
```rust
use std::sync::Arc;

use serde::Deserialize;
use spectre_common::error::AppError;
use spectre_common::http::{respond_app_error, respond_error};

use crate::relay::{OptoRelay, FORCE_SHUTDOWN_MS, PRESS_MS};

#[derive(Deserialize)]
struct PowerEventBody {
    #[serde(rename = "type")]
    event_type: String,
}

#[derive(Serialize)]
struct PowerEventResponse<'a> {
    #[serde(rename = "type")]
    event_type: &'a str,
    duration_ms: u32,
    uptime_s: u64,
}

pub fn register_power_events(
    server: &mut EspHttpServer,
    boot: std::time::Instant,
    relay: Arc<OptoRelay>,
) {
    let _ = server.fn_handler(
        "/api/v1/power-events",
        Method::Post,
        move |mut req| -> std::result::Result<(), Box<dyn std::error::Error>> {
            // Read body (cap at 256 bytes; oversized -> 400)
            let mut buf = vec![0u8; 256];
            let mut total = 0usize;
            loop {
                let n = req.read(&mut buf[total..])?;
                if n == 0 {
                    break;
                }
                total += n;
                if total == buf.len() {
                    return respond_error(req, 400, 400, "request body too large");
                }
            }
            let body_slice = &buf[..total];

            let parsed: PowerEventBody = match serde_json::from_slice(body_slice) {
                Ok(p) => p,
                Err(e) => return respond_error(req, 400, 400, &format!("malformed JSON: {e}")),
            };

            let duration = match parsed.event_type.as_str() {
                "press" => PRESS_MS,
                "force_shutdown" => FORCE_SHUTDOWN_MS,
                other => {
                    return respond_error(
                        req,
                        422,
                        422,
                        &format!("unknown type '{other}'"),
                    );
                }
            };

            match relay.try_press(duration) {
                Ok(()) => {
                    let body = PowerEventResponse {
                        event_type: &parsed.event_type,
                        duration_ms: duration,
                        uptime_s: boot.elapsed().as_secs(),
                    };
                    spectre_common::http::respond_data(req, 201, &body)
                }
                Err(e @ AppError::GpioBusy) => respond_app_error(req, &e),
                Err(e) => respond_app_error(req, &e),
            }
        },
    );
}
```

- [ ] **Step 2: Wire it in main.rs**

In `pc-power-node/src/main.rs`, after `peripherals = Peripherals::take()?;`:
```rust
mod relay;

// pull GPIO 4 out before passing modem to wifi
let opto_pin = peripherals.pins.gpio4;
let relay = relay::OptoRelay::new(opto_pin.into())?;
```
After `handlers::register_health(...)`:
```rust
handlers::register_power_events(&mut server, boot, relay);
```

- [ ] **Step 3: Build and flash**

Run: `cargo run -p spectre-pc-power --release`

- [ ] **Step 4: Bench-test the press (LED instead of PC)**

Wire an LED + 1 kΩ resistor to GPIO 4 + GND for visual feedback. Then:
```fish
# Tap
curl -i -X POST http://spectre-pc.local/api/v1/power-events \
     -H 'Content-Type: application/json' -d '{"type":"press"}'
# Expect 201 + body. LED blinks briefly (~200 ms).

# Force shutdown
curl -i -X POST http://spectre-pc.local/api/v1/power-events \
     -H 'Content-Type: application/json' -d '{"type":"force_shutdown"}'
# Expect 201 + body. LED stays on for ~6 s.

# Validation
curl -i -X POST http://spectre-pc.local/api/v1/power-events \
     -H 'Content-Type: application/json' -d '{"type":"foo"}'
# Expect 422 + body: error.code=422, message="unknown type 'foo'".

# Empty body
curl -i -X POST http://spectre-pc.local/api/v1/power-events \
     -H 'Content-Type: application/json' -d ''
# Expect 400 + body.

# Concurrent press (issue both quickly)
curl -X POST http://spectre-pc.local/api/v1/power-events \
     -H 'Content-Type: application/json' -d '{"type":"press"}' &
curl -X POST http://spectre-pc.local/api/v1/power-events \
     -H 'Content-Type: application/json' -d '{"type":"press"}'
# Expect first 201, second 429.
```

- [ ] **Step 5: Commit**

Run:
```fish
git add pc-power-node/src/handlers.rs pc-power-node/src/main.rs
git commit -m "feat(pc-power-node): POST /api/v1/power-events with busy guard"
```

---

## Phase 5: justfile + smoke scripts

### Task 19: Add a justfile for the common dev recipes

**Files:**
- Create: `justfile`

- [ ] **Step 1: Write justfile**

Create `justfile`:
```just
# spectre dev recipes

# usage: just flash env  OR  just flash pc
flash node:
    @if [ "{{node}}" = "env" ]; then cargo run -p spectre-env --release; \
     elif [ "{{node}}" = "pc" ]; then cargo run -p spectre-pc-power --release; \
     else echo "unknown node: {{node}}"; exit 1; fi

# Open just the serial monitor without rebuilding/flashing
monitor:
    espflash monitor

# Smoke-test an env node by hostname suffix (1, 2, or 3)
smoke-env n="1":
    bash scripts/smoke-env.sh spectre-env{{n}}.local

# Smoke-test the pc-power node
smoke-pc:
    bash scripts/smoke-pc.sh spectre-pc.local

# Build all release binaries
build-all:
    cargo build --release

# Format everything
fmt:
    cargo fmt --all

# Lint everything
clippy:
    cargo clippy --all-targets --all-features -- -D warnings
```

- [ ] **Step 2: Commit**

```fish
git add justfile
git commit -m "chore: justfile with flash/monitor/smoke recipes"
```

---

### Task 20: Add smoke scripts

**Files:**
- Create: `scripts/smoke-env.sh`
- Create: `scripts/smoke-pc.sh`

- [ ] **Step 1: Write smoke-env.sh**

Create `scripts/smoke-env.sh`:
```bash
#!/usr/bin/env bash
set -euo pipefail
host="${1:?usage: smoke-env.sh <hostname>}"

echo "==> $host /health"
curl -sf "http://$host/api/v1/health" | jq .

echo "==> $host /measurements"
curl -sf "http://$host/api/v1/measurements" | jq .

echo "==> $host (unknown route)"
status=$(curl -s -o /dev/null -w '%{http_code}' "http://$host/api/v1/nope")
echo "  status: $status"
[ "$status" = "404" ] || { echo "expected 404"; exit 1; }

echo "OK"
```
Run: `chmod +x scripts/smoke-env.sh`

- [ ] **Step 2: Write smoke-pc.sh**

Create `scripts/smoke-pc.sh`:
```bash
#!/usr/bin/env bash
set -euo pipefail
host="${1:?usage: smoke-pc.sh <hostname>}"

echo "==> $host /health"
curl -sf "http://$host/api/v1/health" | jq .

echo "==> $host press"
curl -sf -X POST "http://$host/api/v1/power-events" \
     -H 'Content-Type: application/json' -d '{"type":"press"}' | jq .

sleep 1

echo "==> $host validation (unknown type)"
status=$(curl -s -o /dev/null -w '%{http_code}' -X POST "http://$host/api/v1/power-events" \
              -H 'Content-Type: application/json' -d '{"type":"foo"}')
echo "  status: $status"
[ "$status" = "422" ] || { echo "expected 422"; exit 1; }

echo "==> $host malformed JSON"
status=$(curl -s -o /dev/null -w '%{http_code}' -X POST "http://$host/api/v1/power-events" \
              -H 'Content-Type: application/json' -d 'not json')
echo "  status: $status"
[ "$status" = "400" ] || { echo "expected 400"; exit 1; }

echo "OK"
```
Run: `chmod +x scripts/smoke-pc.sh`

- [ ] **Step 3: Run them**

```fish
just smoke-env 1
just smoke-pc
```
Both print `OK` at the end.

- [ ] **Step 4: Commit**

```fish
git add scripts/
git commit -m "chore: smoke scripts for env and pc-power nodes"
```

---

## Phase 6: deploy second and third env nodes

### Task 21: Flash env2 and env3

**Files:**
- Modify: `env-node/cfg.toml` (per-node only; not committed)

- [ ] **Step 1: Edit cfg.toml hostname for node 2**

Change `hostname = "spectre-env1"` to `hostname = "spectre-env2"`. Build, flash, monitor:
```fish
just flash env
```

- [ ] **Step 2: Smoke-test node 2**

```fish
just smoke-env 2
```
Expect `OK`.

- [ ] **Step 3: Repeat for node 3**

Edit `cfg.toml` to `hostname = "spectre-env3"`. Re-flash. `just smoke-env 3`. Expect `OK`.

- [ ] **Step 4: Restore cfg.toml to env1**

Edit `cfg.toml` back to `hostname = "spectre-env1"` so the local working copy is consistent.

- [ ] **Step 5: No commit**

`cfg.toml` is gitignored. No commit needed.

---

## Phase 7: hardware integration (after 2026-05-12)

### Task 22: PC PWRBTN polarity check

**Files:**
- None (manual check)

- [ ] **Step 1: Identify the PWRBTN header on the motherboard**

Open the PC. With PSU plugged in but PC off, locate the front-panel header pins labeled `PWR` / `PWRBTN` / `PWR_SW` (mobo manual will show exact position).

- [ ] **Step 2: Multimeter the polarity**

Multimeter in DC voltage mode. Black probe to a known GND on the case. Touch the red probe to each of the two PWRBTN pins. One reads about 3.3 V (or 5 V on some mobos), the other reads 0 V (GND). **Note which is which on a piece of paper.** This determines optocoupler wiring (collector to + side, emitter to GND).

- [ ] **Step 3: Document the finding**

Add a one-line note to `docs/superpowers/specs/2026-05-08-esp-env-monitor-design.md` under section 5 (PC-power node optocoupler wiring): which pin on YOUR motherboard is the +V side. Commit:
```fish
git add docs/superpowers/specs/2026-05-08-esp-env-monitor-design.md
git commit -m "docs(spec): record PC PWRBTN polarity for this build"
```

---

### Task 23: Wire and bench-test the optocoupler

**Files:**
- None (hardware only)

- [ ] **Step 1: Wire the PC817 on a breadboard**

Per spec section 5:
- ESP GPIO 4 -> 330 Ω -> PC817 pin 1 (LED+)
- ESP GND -> PC817 pin 2 (LED-)
- PC817 pin 4 (collector) -> indicator LED + 1 kΩ -> +3.3 V on a separate bench supply (or use the ESP's 3V3 pin BUT keep PC mobo isolated when you eventually wire it in)
- PC817 pin 3 (emitter) -> bench GND

- [ ] **Step 2: Verify with curl**

```fish
just smoke-pc
```
LED across pins 3-4 should blink briefly during the press, stay on for the force_shutdown test.

- [ ] **Step 3: Hand-document any wiring corrections**

If the LED doesn't light, verify polarity on the LED side and check current. Pencil-mark the verified wiring.

---

### Task 24: Wire battery + solar + boost (one env node)

**Files:**
- None (hardware only)

- [ ] **Step 1: Set MT3608 output to 5.0 V (no load)**

Connect a bench supply at 3.7 V to MT3608 input. With multimeter on output, turn the trimmer until output reads 5.0 V. Disconnect.

- [ ] **Step 2: Wire one env node fully per spec section 5**

```
Solar (+) -> 1N5817 anode
1N5817 cathode -> TP4056 IN+
Solar (-) -> TP4056 IN-
TP4056 B+ -> 18650 holder red
TP4056 B- -> 18650 holder black
TP4056 OUT+ -> MT3608 IN+
TP4056 OUT- -> MT3608 IN- (= GND)
MT3608 OUT+ -> ESP 5V pin
MT3608 OUT- -> ESP GND
```
Insert charged 18650.

- [ ] **Step 3: Confirm boot on battery**

Open serial monitor (`just monitor`). The node should boot and announce mDNS just as on USB.

- [ ] **Step 4: Smoke-test from another machine**

```fish
just smoke-env 1
```
Expect `OK`. The env1 node is now battery + solar powered.

- [ ] **Step 5: Repeat for env2 and env3 once verified**

---

## Phase 8: optional polish

### Task 25: Add `cargo clippy` + `cargo fmt` to CI hook (if desired)

**Files:**
- Create: `.git/hooks/pre-commit`

- [ ] **Step 1: Write a pre-commit hook**

Create `.git/hooks/pre-commit`:
```bash
#!/usr/bin/env bash
set -e
just fmt
just clippy
```
`chmod +x .git/hooks/pre-commit`. (Not committed; lives in `.git/hooks/` only.)

- [ ] **Step 2: Verify**

Make a trivial change, `git commit -am "test"`. Hook runs fmt + clippy.

---

### Task 26: Tag a v0.1 release

**Files:**
- None

- [ ] **Step 1: Tag**

```fish
git tag -a v0.1 -m "spectre v0.1: env nodes + pc-power node, pull-only HTTP"
```

- [ ] **Step 2: Done**

---

## Self-Review

**Spec coverage:**

| Spec section | Implementing tasks |
|---|---|
| 1. Goal | (whole plan) |
| 2. Architecture (workspace, mDNS, pull-only, no auth) | Tasks 1-4, 7, 8 |
| 3. Crate layout | Tasks 1-4 (scaffolds), 5-9 (common), 10-15 (env), 16-18 (pc-power) |
| 4. HTTP API | Tasks 9 (envelope), 10 + 16 (/health), 15 (/measurements), 18 (/power-events) |
| 5. Hardware | Tasks 22 (polarity), 23 (opto bench), 24 (battery+solar) |
| 6. Data flow + error handling | Tasks 5 (AppError), 9 (respond_*), 15 (503 path), 18 (422/400/429) |
| 7. Testing | Tasks 5 (unit-ish), 15 (smoke), 18 (smoke), 20 (smoke scripts) |
| 8. Out of scope | Acknowledged: deferred features not implemented |
| 9. Open questions | bme280-rs vs bme280 noted in Task 12; HTTP server choice locked to esp-idf-svc default |

No spec section is unimplemented.

**Placeholder scan:** No "TBD" / "TODO later" / "implement appropriate X" / "similar to" left in the steps. The few `// TODO Task N` comments inside stubs are pointers to a later task that DOES implement the code.

**Type consistency:**
- `Bme280Reading` defined in Task 11 stub, implemented in Task 12 — same shape.
- `Sensors::read_all() -> Result<Measurements>` defined in Task 11, used in Task 15 — match.
- `OptoRelay::try_press(duration_ms: u32) -> Result<()>` defined in Task 17, used in Task 18 — match.
- `respond_data` / `respond_error` / `respond_app_error` defined in Task 9, used in Tasks 10, 15, 16, 18 — match.
- `ENV_NODE_CONFIG` / `PC_POWER_NODE_CONFIG` consts come from `toml-cfg` macro in Task 6 — used in Tasks 10 + 16.

No mismatched names found.

---

## Execution handoff

Plan complete. Two execution options when you're ready:

1. **Subagent-Driven** (recommended) — fresh subagent per task, two-stage review between tasks, fast iteration.
2. **Inline Execution** — execute tasks in this session via `superpowers:executing-plans`, batched with checkpoints.

Tasks 0 to 21 are software-only and can run before parts arrive. Tasks 22 to 24 need the boost converter and Schottky to land (ETA 2026-05-12).
