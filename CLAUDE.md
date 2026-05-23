# Spectre

Homelab fleet of ESP32-C3 SuperMini nodes running Rust on ESP-IDF, exposing plain HTTP endpoints over Wi-Fi on the home LAN.

## Topology

* **3 environment nodes** — BME280 + BH1750 + LM393 reed module, battery + solar via TP4056 + MT3608, mDNS `spectre-env1.local`, `spectre-env2.local`, `spectre-env3.local`.
* **1 PC-power node** — PC817 optocoupler across the PC's front-panel power-button header, USB-powered from the PC's internal 5 V SB rail, mDNS `spectre-pc.local`.
* **1 ESP32-C3 spare.**

Pull-only HTTP for v1: no push, no cache, no auth, plain HTTP. User has explicitly accepted the no-auth / plain-HTTP risk on a trusted home LAN (PC-power endpoint is the only sensitive one).

## Workspace

Cargo workspace at the repo root. Crates:

* `common/` — `spectre-common` (lib): error type, config, Wi-Fi join, mDNS, HTTP scaffold.
* `env-node/` — `spectre-env` (binary): reads sensors + serves `GET /api/v1/measurements`.
* `pc-power-node/` — `spectre-pc-power` (binary): drives optocoupler + serves `POST /api/v1/power-events`.

None of these crates exist yet — they're created in plan Task 1+.

## Authoritative docs

* `docs/superpowers/specs/2026-05-08-esp-env-monitor-design.md` — design spec
* `docs/superpowers/plans/2026-05-08-spectre-implementation.md` — 28-task implementation plan

Read those before starting any task.

## Workflow preference

**Wiring before code.** Even though plan Tasks 0-9 are software-only (toolchain + Cargo scaffold + `spectre-common`), the user prefers to wire the breadboard first so there's something physical to flash before any Rust gets written. Don't push to start `cargo` work just because it's technically possible.

**Assist mode** is in effect by default for this project (`/assist`) — Claude should not write code that drops directly into the project, only hints + Socratic review + plans + concepts. User keeps their hands on the keyboard to learn. Escape phrase: "just write it" for a single response.

## Hardware status (as of 2026-05-23)

**In hand:**
* ESP32-C3 SuperMini ×5, BME280 ×3, BH1750 ×5, LM393 reed ×5, magnets, 18650 cells ×20, TP4056 ×5, solar panels ×6, PC817 ×10, dupont jumpers, multimeter.
* Diotec 1N5817 Schottky diode (DO-15, 20 V) — for solar (+) → TP4056 IN+ reverse-current block.

**Ordered, ETA 2026-05-28:**
* GERUI 10× MT3608 boost converter modules (Amazon)
* CY USB 2.0 internal-header → USB-C cable, 50 cm (PC-power node)
* QWORK silicone wire 22 AWG, 5 colors × 10 m (permanent battery/solar joints)
* Elegoo 3× MB-102 breadboards (830-point)
* AZDelivery 3× MB-102 breadboard PSU modules
* Newding 12 V / 2 A universal PSU, multi-tip barrel, Type-F (Schuko)

**Still missing (not yet bought):**
* Heat-shrink tube assortment (2-5 mm) for solder joints on the battery/solar chain. Cheap (~€5). Mention before Task 24.

## Bench-power preference

User does **not** want batteries on the breadboard during prototyping. Bench PSU is wall-wart powered (Newding 12 V → MB-102 module → 3.3 V/5 V rails). 18650 cells stay reserved for the final env-node power chain (TP4056 + MT3608, plan Task 24).

## Where we left off (2026-05-23)

* Plan reviewed, task list set up.
* All 28 plan tasks pending — none started.
* Hardware ordered, ETA 2026-05-28.
* **Next session:** confirm parts arrived, verify wall-wart polarity (center-positive, ⊖―⊙―⊕), then walk through wiring one env node on a breadboard per the wiring guide. After the breadboard is wired and ESP enumerates over USB, start plan Task 0 (toolchain install).

## Pitfalls worth remembering

* **BMP280 vs BME280**: counterfeit modules sometimes ship BMP280 silicon (no humidity). Plan's auto-probe checks chip-ID register `0xD0` — BME280 returns `0x60`, BMP280 returns `0x58`. Catches the swap.
* **ESP32-C3 SuperMini pinout varies between clones.** Trust the silkscreen on the actual board, not online pinout diagrams.
* **Reed module polarity varies.** Plan defaults `active_low = true`; half the LM393 reed modules out there are wired opposite. Verify with multimeter or by inspecting the API response.
* **ErP setting in BIOS** must be disabled to keep USB 5V SB alive while PC is in S5 — required for the PC-power node to stay reachable.
* **`env-node/cfg.toml` and `pc-power-node/cfg.toml` contain Wi-Fi creds** — gitignored from Task 10 onward. Don't commit them.
