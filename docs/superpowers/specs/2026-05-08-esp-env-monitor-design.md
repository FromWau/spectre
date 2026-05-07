# Spectre: ESP environment monitor and PC power control

**Date:** 2026-05-08
**Status:** approved by user, ready for implementation planning
**Stack:** Rust on ESP-IDF (via `esp-idf-svc` + `esp-idf-hal`), ESP32-C3 SuperMini

## 1. Goal

Build a small homelab fleet of ESP32-C3 nodes that expose environment readings and a remote PC power button over plain HTTP on the home Wi-Fi.

Four nodes total:

* Three identical battery + solar powered environment nodes, each carrying a BME280 (temperature, humidity, pressure), a BH1750 (lux), and an LM393 reed module (door/window state).
* One USB powered PC power node, carrying a single optocoupler wired across the PC's front panel power button header. Lets a client "press" the power button remotely.

The fifth ESP32-C3 stays as a spare.

## 2. Architecture

```
                    Home Wi-Fi
   +----------+  +----------+  +----------+  +------------+
   | env1     |  | env2     |  | env3     |  | pc-power   |
   | .local   |  | .local   |  | .local   |  | .local     |
   +----------+  +----------+  +----------+  +------------+
   | ESP32-C3 |  | ESP32-C3 |  | ESP32-C3 |  | ESP32-C3   |
   | BME280   |  | BME280   |  | BME280   |  | Optocoupler|
   | BH1750   |  | BH1750   |  | BH1750   |  |   to PC PWR|
   | Reed     |  | Reed     |  | Reed     |  |   header   |
   | 18650+sun|  | 18650+sun|  | 18650+sun|  | USB 5V SB  |
   +----------+  +----------+  +----------+  +------------+
```

Key choices:

* **Cargo workspace** rooted at `/home/fromml/Projects/esp/`, project name `spectre`. Three crates: `spectre-common` (lib, dir `common/`), `spectre-env` (bin, dir `env-node/`), `spectre-pc-power` (bin, dir `pc-power-node/`).
* **Stack:** Rust on top of ESP-IDF via `esp-idf-svc` + `esp-idf-hal`. Async optional, blocking handlers fine for v1.
* **Network:** Wi-Fi STA, DHCP, mDNS hostname per node (`spectre-env1.local`, `spectre-env2.local`, `spectre-env3.local`, `spectre-pc.local`).
* **Transport:** plain HTTP/1.1 over TCP, no TLS. JSON bodies. No auth. Trusted home LAN.
* **Pull-only for v1.** No background sampler, no cache. Each request reads fresh sensor values.
* **Reed door state** is read at request time (no ISR for v1). Trades real-time edge detection for simplicity. Future push or SSE endpoint will revisit this.

### Deviations from `rest-api-design` skill (explicit)

| Skill rule | Deviation | Rationale |
|---|---|---|
| Always use HTTPS | Plain HTTP only | Trusted home LAN. ESP-IDF TLS adds about 60 KB flash + cert management; not worth it for v1. Revisit if nodes ever leave LAN. |
| Auth required | No auth | User decision. PC-power endpoint is the only sensitive one; mitigation is home LAN trust + LAN segmentation. |
| Rate limiting | None | Not adversarial. Single client. |
| Pagination | n/a | v1 has no list endpoints. Will apply to a future `GET /power-events` and any historical-measurements endpoint. |

## 3. Crate layout

```
esp/
+-- Cargo.toml                         (workspace)
+-- rust-toolchain.toml                (nightly + esp riscv32imc target)
+-- .cargo/config.toml                 (target = riscv32imc-esp-espidf, runner = espflash)
+-- sdkconfig.defaults                 (ESP-IDF tuning: logs, partition table, etc.)
|
+-- common/                            (shared library crate)
|   +-- Cargo.toml
|   +-- src/
|       +-- lib.rs                     (re-exports)
|       +-- wifi.rs                    (connect_wifi(ssid, pass) -> EspWifi)
|       +-- mdns.rs                    (advertise(hostname))
|       +-- http.rs                    (build_server() + helper to register JSON routes)
|       +-- error.rs                   (AppError enum, Result<T> alias)
|       +-- config.rs                  (NodeConfig: hostname, ssid/pass via cfg.toml + env)
|
+-- env-node/                          (binary)
|   +-- Cargo.toml
|   +-- build.rs
|   +-- cfg.toml                       (NodeConfig values)
|   +-- src/
|       +-- main.rs                    (boot: wifi -> mdns -> http server -> park)
|       +-- sensors/
|       |   +-- mod.rs                 (struct Sensors, read_all())
|       |   +-- bme280.rs              (I2C driver)
|       |   +-- bh1750.rs              (I2C driver, one-shot high-res)
|       |   +-- reed.rs                (GPIO read on the LM393 DO line)
|       +-- handlers.rs                (GET /api/v1/measurements -> JSON)
|
+-- pc-power-node/                     (binary)
    +-- Cargo.toml
    +-- build.rs
    +-- cfg.toml
    +-- src/
        +-- main.rs                    (boot: wifi -> mdns -> http server -> park)
        +-- relay.rs                   (OptoRelay { gpio }, press(duration))
        +-- handlers.rs                (POST /api/v1/power-events)
```

### Crate responsibilities

* **`common`** owns anything both binaries do identically: Wi-Fi join (with retry), mDNS announce, HTTP server bring-up, JSON helper that wraps `esp-idf-svc::http::server::Response`, structured logging via `log` + `esp-idf-svc::log::EspLogger`, error type. Roughly 150 to 250 lines, single purpose.
* **`env-node`** owns one I2C bus (`Mutex<I2cDriver>` shared across the two sensor modules), one GPIO for the reed module's DO line, one HTTP route. Roughly 400 to 600 lines.
* **`pc-power-node`** owns one GPIO driving an optocoupler LED, one HTTP route handling two event types. Roughly 100 to 200 lines.

### Workspace dependencies (pinned)

* `esp-idf-svc`, `esp-idf-hal`, `esp-idf-sys`
* `embedded-hal`, `embedded-hal-bus` (sharing the I2C bus between BME280 and BH1750)
* `bme280` or `bme280-rs` (whichever supports `embedded-hal` 1.0)
* `bh1750` driver crate (one-shot high-res mode)
* `serde`, `serde_json`
* `log`, `thiserror`
* `toml-cfg` for `cfg.toml` to const generation

## 4. HTTP API

All paths under `/api/v1/`. URLs are noun resources. snake_case throughout. Single error envelope shape across all responses.

### Env nodes (`spectre-env1.local`, `spectre-env2.local`, `spectre-env3.local`)

`GET /api/v1/measurements` returns `200 OK`:

```json
{
  "data": {
    "node_id": "env1",
    "uptime_s": 12453,
    "temperature_c": 21.43,
    "humidity_pct": 47.2,
    "pressure_hpa": 1013.8,
    "lux": 312.5,
    "door_open": false,
    "battery_v": 3.92
  }
}
```

* `data` envelope leaves room for a `meta` peer in v2.
* `door_open` is a live GPIO read at request time. `true` means no magnet near the reed.
* `battery_v` is `null` until ADC wiring is added (see hardware section).

### PC-power node (`spectre-pc.local`)

`POST /api/v1/power-events` returns `201 Created`.

Request body (required):

```json
{ "type": "press" }
```

or

```json
{ "type": "force_shutdown" }
```

Response:

```json
{
  "data": {
    "type": "press",
    "duration_ms": 200,
    "uptime_s": 8421
  }
}
```

* `type: "press"` closes the optocoupler for 200 ms (normal power on/off tap).
* `type: "force_shutdown"` closes the optocoupler for 6000 ms (PSU cut).
* Synchronous: response sent only after the press completes.
* No `Location` header. Events are not persisted, so no retrievable URI for an individual event. A future `GET /api/v1/power-events?since=...` could return a RAM-only ring buffer.

### All nodes

`GET /api/v1/health` returns `200 OK`:

```json
{ "data": { "ok": true, "uptime_s": 12453, "node_id": "env1" } }
```

### Status codes

| Code | Meaning | Where |
|---|---|---|
| `200` | OK | `GET` responses |
| `201` | Created | `POST /power-events` |
| `400` | Malformed JSON | any `POST` |
| `404` | Unknown route | any |
| `422` | Validation error (unknown `type`, missing field) | `POST /power-events` |
| `429` | Press already in progress | `POST /power-events` |
| `500` | Unexpected internal error | any |
| `503` | Sensor read failed | env nodes only |

### Error envelope

```json
{ "error": { "code": 422, "message": "unknown type 'foo'" } }
```

* `code` is numeric and matches the HTTP status (per the rest-api-design skill's example shape).
* `message` is human-readable, may evolve.

## 5. Hardware

### BOM (confirmed in hand)

| Part | Qty | Notes |
|---|---|---|
| ESP32-C3 SuperMini | 5 | USB-C, dual button (RST + BOOT), on-board AMS1117 LDO, 3.3 V available on `3V3` pin |
| BME280 (bare 6-pin GY-BME/BMP280) | 3 | needs CSB tied to VCC for I2C mode, SDO tied to GND for address `0x76` |
| GY-302 BH1750 (5-pin) | 5 | on-board LDO accepts 3.3 to 5 V, ADDR floating or to GND for `0x23` |
| LM393 reed module (3-pin) | 5 | clean DO output, about 10 mA continuous idle (comparator + LEDs) |
| 10 x 3 mm neodym disc magnets | 20 | pair with reed; may need spacer to prevent stuck-closed |
| Samsung INR18650-35E cell | 20 | 3500 mAh nominal, 8 A max discharge, button-top |
| 18650 single-cell holder with leads | 5 | red = +, black = - to TP4056 B+/B- |
| TP4056 + DW01 protection board (USB-C) | 5 | combined charge + protect; pads `B+ B- OUT+ OUT-`, USB-C input also acts as IN+/IN- via the connector |
| 1 W 5 V solar panel (107 x 61 mm) | 6 | bare back, no leads attached, no on-board diode |
| Sharp PC817 optocoupler (4-pin DIP) | 10 | CTR rank C (200 to 400%), one needed for PC-power node |
| XINWEI 1/4 W resistor kit (220 to 100K all present) | 1 set | covers opto current limit (220 or 330 ohm) and future battery divider (100K) |
| 20 cm dupont jumpers (M-M, M-F, F-F mix) | 1 set | for breadboarding env nodes |

### BOM (still to arrive, ordered)

| Part | Qty | Source | ETA |
|---|---|---|---|
| MT3608 boost module | 3 | Conrad | 2026-05-12 |
| 1N5817 Schottky diode (Diotec, DO-15) | 3 | Conrad | 2026-05-12 |

### Env-node power topology

```
Solar 1W (107x61mm)
   |
   +--> 1N5817 (optional, blocks reverse current at night)
   |
   v
TP4056 IN+/IN- (under the USB-C connector, or solder direct to IN pads)
   |
   v
Samsung 35E cell (B+/B-)
   |
   v
TP4056 OUT+/OUT-
   |
   v
MT3608 boost (output set to 5.0 V)
   |
   v
ESP32-C3 SuperMini 5V pin -> on-board AMS1117 LDO -> 3V3 pin -> sensors
```

USB-C on the SuperMini stays usable for flashing without back-feed concerns because the boost output isolates the cell from the USB rail.

### Env-node pin map (ESP32-C3 SuperMini)

| Function | GPIO | Notes |
|---|---|---|
| I2C SDA | 5 | also SPI MISO; not used as SPI |
| I2C SCL | 6 | also SPI CLK; not used as SPI |
| Reed input (LM393 DO) | 4 | non-strapping; module already provides clean digital |
| Battery sense (future) | 0 | ADC1_CH0; needs 100K + 100K divider so 4.2 V scales to 2.1 V |
| Status LED | 8 | on-board blue LED, active LOW; strapping pin but safe after boot |
| BOOT button | 9 | already wired with pull-up; future "factory reset" hook |
| 5 V supply in | 5V pin | from MT3608 boost output |

### Env-node sensor wiring

BME280 (bare 6-pin variant):

| BME280 pin | Goes to |
|---|---|
| VCC | ESP `3V3` |
| GND | ESP `GND` |
| SCL | ESP GPIO 6 |
| SDA | ESP GPIO 5 |
| CSB | ESP `3V3` (must be HIGH to select I2C mode) |
| SDO | ESP `GND` (sets address `0x76`) |

GY-302 BH1750:

| BH1750 pin | Goes to |
|---|---|
| VCC | ESP `3V3` |
| GND | ESP `GND` |
| SCL | ESP GPIO 6 (shared with BME280) |
| SDA | ESP GPIO 5 (shared with BME280) |
| ADDR | ESP `GND` (or floating; default `0x23`) |

LM393 reed module:

| Module pin | Goes to |
|---|---|
| VCC | ESP `3V3` (modules accept 3.3 to 5 V) |
| GND | ESP `GND` |
| DO | ESP GPIO 4 |

I2C addresses are auto-probed at boot: BME280 tries `0x76` then `0x77`, BH1750 tries `0x23` then `0x5C`. Detected address is logged and used for the rest of the run. Reed DO polarity (active-LOW vs active-HIGH) is set by a config constant after a one-time bench check.

### PC-power node power topology

```
PC internal USB 2.0 header (5 V SB, GND)
   |
   v
ESP32-C3 5V pin / GND -> on-board AMS1117 LDO -> 3.3 V to MCU
```

5 V SB (standby) is alive whenever the PSU is plugged in; the ESP stays online even when the PC is powered off, same source Wake-on-LAN uses.

### PC-power node optocoupler wiring (PC817)

```
                                   Optocoupler (PC817)
                  +----------+------+------+
   ESP GPIO 4 ->--+ 330 ohm  | 1   4|  ---> PC PWRBTN+ (+3.3 V side, mobo)
                  +----------+ LED  +
                             | 2   3|  ---> PC PWRBTN- (GND side, mobo)
                             +------+
                          ISOLATED: no shared GND between ESP and PC mobo
```

Do NOT tie ESP GND to PC mobo GND. Galvanic isolation is the whole point.

To find which PWRBTN pin is +3.3 V vs GND: with PC plugged in but off, multimeter in DC mode between the two pins on the front panel header. Mobo holds one pin at +3.3 V, the other at GND. Wire the optocoupler collector (pin 4) to the +3.3 V side, emitter (pin 3) to GND side. Get this backwards and the press will not register, but nothing will be damaged.

Pulse durations driven from firmware: 200 ms (`type: press`), 6000 ms (`type: force_shutdown`). Both use the same GPIO; `force_shutdown` just holds it longer.

### Power budget (env node)

| Mode | Avg current (incl. reed module) | Runtime per charge |
|---|---|---|
| WiFi always-on, no sleep | about 90 mA | about 39 h |
| WiFi modem-sleep (default in ESP-IDF) | about 40 mA | about 87 h |
| Light-sleep between requests (future) | about 17 mA | about 205 h |

For v1 (modem-sleep only, no light-sleep), about 3.6 days per charge. Solar tops it back up almost any day with reasonable light. Comfortable margin.

LM393 reed module idle draw (about 10 mA continuous) is the dominant non-radio load. Future optimization: de-solder the indicator LEDs on the LM393 module, or bypass the comparator entirely and tap directly off the reed switch leads.

### Hardware checklist before flashing

* [ ] Receive MT3608 boost + 1N5817 Schottky on 2026-05-12
* [ ] Multimeter the PC PWRBTN polarity on the target motherboard before connecting the optocoupler
* [ ] Confirm the motherboard's internal USB header pinout from its manual

## 6. Data flow and error handling

### Env-node request flow (`GET /api/v1/measurements`)

```
HTTP request
    |
    v
+------------------------------------------------+
| http handler                                    |
|  1. acquire I2C bus mutex                       |
|  2. read BME280  -> temp_c, hum_pct, press_hpa  |
|  3. read BH1750  -> lux                         |
|  4. read GPIO 4  -> reed.DO                     |
|  5. release mutex                               |
|  6. build JSON response                         |
+------------------------------------------------+
    |
    v
HTTP 200 + JSON
```

* Single I2C bus mutex owned by the handler. BME280 read takes about 30 ms (forced mode + readout); BH1750 one-shot high-res takes about 120 ms. Reed is sub-ms GPIO read. Total about 150 ms per request, fine for any pull rate up to 5 Hz.
* No background task. No cache. Each request is independent.
* Concurrent requests are serialized by the mutex; the second one waits about 150 ms for the first.

### PC-power request flow (`POST /api/v1/power-events`)

```
HTTP request body: {"type": "press" | "force_shutdown"}
    |
    v
+------------------------------------------------+
| http handler                                    |
|  1. parse body, validate type field             |
|     -> 422 if missing/unknown                   |
|  2. acquire press-in-progress mutex             |
|     -> 429 if already pressing                  |
|  3. set GPIO 4 HIGH                             |
|  4. sleep N ms (200 or 6000)                    |
|  5. set GPIO 4 LOW                              |
|  6. release mutex                               |
|  7. build JSON response                         |
+------------------------------------------------+
    |
    v
HTTP 201 + JSON
```

* Press-in-progress mutex prevents two concurrent presses from overlapping. Returns `429` if a press is already underway.
* Synchronous response: sent only after GPIO returns LOW. Client knows the press completed when it gets the 2xx.

### Error handling matrix

| Failure | HTTP code | `error.code` | Retry? |
|---|---|---|---|
| Malformed JSON in request body | `400` | `400` | No (fix client) |
| Unknown / missing `type` field | `422` | `422` | No (fix client) |
| Press already in progress | `429` | `429` | Yes, after duration |
| BME280 NACK / I2C bus lock | `503` | `503` | Yes, after 1 s |
| BH1750 NACK / I2C bus lock | `503` | `503` | Yes, after 1 s |
| WiFi disconnected mid-request | TCP times out, no response | n/a | Yes |
| Internal panic / unexpected | `500` | `500` | Investigate |

### Failure-recovery behavior

* **Wi-Fi disconnect:** background task in `common::wifi` retries every 5 s with backoff up to 60 s. mDNS re-announces on reconnect. HTTP server stays up; new requests succeed once Wi-Fi is back.
* **I2C bus stuck low** (rare; can happen with hot-plugged modules): driver attempts a 9-clock-pulse bus recovery on first failed read; on second failed read returns `503`. Doesn't crash the node.
* **Panic / unhandled fault:** ESP-IDF core dump enabled, watchdog reboots the node within about 5 s. Comes back up cleanly. Lost in-flight requests fail at the TCP layer.
* **Sensor probe at boot:** firmware tries `0x76` then `0x77` for BME280, and `0x23` then `0x5C` for BH1750. Found address is logged and used for the rest of the run. If neither responds, log loud error and respond `503` to all `GET /measurements` until reboot. `GET /health` still works.

### Logging

* ESP-IDF logger via the `log` crate, level `info` for normal operation, `debug` available via `cfg.toml`.
* Each request logs: `node=env1 method=GET path=/api/v1/measurements status=200 duration_ms=147`.
* Boot logs include detected sensor addresses, Wi-Fi connect time, mDNS announce result.

## 7. Testing

Three layers, increasing in fidelity and cost-to-run.

### Layer 1: host-side unit tests (`cargo test`)

Pure logic, runs in seconds on the host, no ESP needed.

| Crate | What's tested |
|---|---|
| `common::http` | JSON envelope shape, HTTP status code mapping, content-type header |
| `common::config` | `cfg.toml` parsing, required-field validation |
| `pc-power-node` body parser | Valid types parse; missing `type` returns `422`; unknown `type` returns `422`; malformed JSON returns `400` |
| `env-node` response builder | Field rounding (2 decimals), `null` battery_v handling |
| Sensor decoders | `common::sensors::bme280` and `bh1750` decode functions: feed canned register byte sequences, assert decoded values. Use `embedded-hal-mock` for the I2C bus. |

These live in `#[cfg(test)] mod tests` blocks inside each crate. Run with `cargo test -p common`, etc.

### Layer 2: on-device smoke checks (manual, scripted)

After flashing, run from the dev machine:

```fish
# Reachability
curl -s http://spectre-env1.local/api/v1/health | jq .

# Measurement shape
curl -s http://spectre-env1.local/api/v1/measurements | jq .

# Door state: open the door, re-curl, expect door_open=true

# Sensor failure simulation: unplug BME280 SDA, expect 503
curl -i http://spectre-env1.local/api/v1/measurements

# PC-power tap (with opto driving an LED on the bench, NOT yet the PC)
curl -i -X POST http://spectre-pc.local/api/v1/power-events \
     -H 'Content-Type: application/json' \
     -d '{"type":"press"}'

# Validation
curl -i -X POST http://spectre-pc.local/api/v1/power-events \
     -H 'Content-Type: application/json' \
     -d '{"type":"foo"}'
# expect 422

# Concurrent press
curl -X POST http://spectre-pc.local/api/v1/power-events -d '{"type":"press"}' &
curl -X POST http://spectre-pc.local/api/v1/power-events -d '{"type":"press"}'
# expect first 201, second 429
```

Bundle these into `scripts/smoke.sh` (or `just smoke env1`) so it's one command.

### Layer 3: hardware-in-the-loop (optional, post-MVP)

A Rust test binary in `tests/hil.rs` (or a separate `xtask` workspace member) that:

1. Builds the firmware (`cargo build -p env-node --release --target riscv32imc-esp-espidf`)
2. Flashes via `espflash flash`
3. Waits for Wi-Fi join (parse serial logs)
4. Hits the endpoints with `reqwest`, asserts on status + body shape

Out of scope for v1. Layer 2 smoke script gets 95 % of the value at 5 % of the effort. Defer until firmware churn justifies the automation.

### What's deliberately NOT tested

* Hardware quirks (which BME280 address you got, whether the reed module's DO is active-LOW or active-HIGH): handled at runtime via probing + a config constant.
* Solar charging behavior: observable only over hours/days. Logged via `battery_v` ADC once that's wired.
* Optocoupler wiring polarity on the PC: only checkable on a real motherboard. Smoke-test phase 2.

### Tooling

| Tool | Purpose |
|---|---|
| `cargo test` | Layer 1 |
| `cargo nextest` (optional) | Faster runner, prettier output |
| `embedded-hal-mock` | I2C bus simulation in unit tests |
| `espflash` / `cargo-espflash` | Flashing |
| `cargo-espmonitor` | Serial log viewing |
| `curl` + `jq` | Layer 2 |
| `just` (optional) | Recipe runner: `just flash env1`, `just smoke env1`, `just monitor env1` |

## 8. Out of scope for v1

* Push or SSE endpoints from env nodes to a remote DB (planned: v2).
* Battery voltage ADC reading (planned: once divider is wired; firmware code path stubbed in v1, returns `null`).
* Light-sleep / deep-sleep power optimization (planned: v3 if battery + solar margin proves insufficient).
* TLS, auth, rate limiting (intentionally deferred; trusted home LAN).
* Persistent event log on the PC-power node (would need RAM ring buffer + retrievable URIs; v2).
* Web dashboard (any HTTP client works for v1; build a UI later if useful).
* Reed interrupt / edge detection (poll-on-request only for v1).
* OTA updates (flash via USB-C for v1).

## 9. Open questions for implementation phase

* Which BME280 driver crate (`bme280` vs `bme280-rs`) plays nicest with `embedded-hal` 1.0 on the current `esp-idf-hal` release? Pick at planning time.
* Confirm `MaybeUninit`-free static initialization pattern works for the shared `Mutex<I2cDriver>`, or fall back to `lazy_static`.
* Decide whether to use `esp-idf-svc` HTTP server or a slimmer alternative (`picoserve` on top of std). Default: built-in HTTP server, lowest friction.
