# PiPower — Raspberry Pi UPS HAT

**A Raspberry Pi 4 / 5 UPS HAT. Lives inside a 3rd-party aluminium thermal case alongside the Pi. Keeps the homelab alive through brownouts.**

[![Status](https://img.shields.io/badge/status-design_doc-blue)](docs/requirements.pdf)
[![License](https://img.shields.io/badge/license-CERN--OHL--P--2.0-blue)](LICENSE)
[![Target](https://img.shields.io/badge/target-Pi_4_%2F_Pi_5-c51a4a)](https://www.raspberrypi.com/)
[![Designed With](https://img.shields.io/badge/Open_Hardware-OSHW-orange)](https://www.oshwa.org/)

<p align="center">
  <em>TODO: hero shot — board render + a populated photo inside the aluminium case</em><br/>
  <code>docs/photos/hero.jpg</code>
</p>

---

> **Project status: design doc, no hardware yet.**
> Requirements locked. Topology and parts still being chosen. See [`docs/requirements.pdf`](docs/requirements.pdf) for the source-of-truth requirements document.

---

## Why I'm Building This

My homelab runs on a Raspberry Pi 4 — Caddy + Authelia + Cloudflare Tunnel + Pi-hole + Home Assistant, all on a single SD card and a single 5V rail. Bangalore's grid is mostly fine and occasionally not. A brownout drops the Pi mid-write, the SD card corrupts a little more each time, and recovery is a Sunday afternoon of restoring from backups. Commercial Pi UPS HATs exist but are either toy-grade (no real current capacity, no telemetry) or overpriced for what they do.

PiPower is the boring, careful version: a UPS HAT that meets the actual electrical requirements of a Pi 4 / Pi 5 (5V at 3A continuous, 4A peak, <100 mV ripple, zero-delay switchover), with real battery management and I²C telemetry to the Pi so it can shut down gracefully when the battery gets low.

The other half of the motivation is **fitting it into the existing thermal solution**. India gets hot. My Pi already lives inside a 3rd-party aluminium case that doubles as a passive heatsink (the Argon ONE / GeeekPi / DeskPi family). The UPS HAT has to live inside that same case — alongside the Pi, in the empty space above or beside it, sharing thermal mass. So this isn't a generic UPS HAT design; it's a UPS HAT designed to coexist with a thermal case.

---

## Initial Requirements

Source: [`docs/requirements.pdf`](docs/requirements.pdf) (preliminary, v0).

| # | Requirement | Target | Status |
|---|---|---|---|
| 1 | Compatible with Pi 4 and Pi 5 | Both, same board | 🎯 Pin compatibility to be verified |
| 2 | Output voltage | 5 V ±5% all loads | 🎯 Spec'd |
| 3 | Continuous output current | ≥3 A | 🎯 Spec'd |
| 4 | Peak transient current | ≥4 A | 🎯 Spec'd |
| 5 | Output ripple | <100 mV pk-pk | 🎯 Spec'd |
| 6 | Switchover delay (AC → battery) | 0 ms (true load-share, no relay) | 🎯 Spec'd — non-negotiable |
| 7 | Battery topology | Open | ⏳ TBD — single-cell + boost vs. 2S + buck |
| 8 | Battery format | Replaceable, common cell | 🎯 18650 or 21700 (per requirements doc) |
| 9 | Overcharge protection | ~4.2 V/cell | 🎯 Spec'd |
| 10 | Deep discharge cutoff | 2.8–3.0 V/cell | 🎯 Spec'd |
| 11 | Thermal monitoring | NTC on battery | 🎯 Spec'd |
| 12 | I²C telemetry to Pi | V<sub>bat</sub>, I<sub>chg</sub>, I<sub>dis</sub>, V<sub>in</sub> presence | 🎯 Spec'd |
| 13 | Fuel gauge (state of charge) | Optional | 🎯 Stretch goal |
| 14 | Input | 5 V 3 A adapter (official Pi supply) via barrel jack and/or USB-C | ⏳ TBD — connector choice open |
| 15 | Output to Pi | GPIO header (pogo pins) and/or USB-C-out | ⏳ TBD — output topology open |
| 16 | Form factor | Full-size HAT, 65 × 56 mm, 2 × 20 GPIO | 🎯 Locked |
| 17 | Thermal coexistence | Fits inside Argon ONE / GeeekPi / DeskPi class metal case alongside Pi | 🎯 Locked |
| 18 | Protection | Reverse polarity, ESD (TVS), input polyfuse, short-circuit | 🎯 Spec'd |
| 19 | Graceful shutdown trigger | Pi receives shutdown signal before battery cutoff | 🎯 Spec'd |
| 20 | Charger | CC/CV, configurable I<sub>in</sub> limit, simultaneous load + charge | 🎯 Spec'd |
| 21 | Charger IC | Open | ⏳ TBD — research pending |

---

## Open Design Decisions

These are the things still up for research and trade study. The README will lock them as the design progresses.

### Battery topology

| Option | Implication | Pros | Cons |
|---|---|---|---|
| **Single-cell Li-ion (18650/21700) + boost** | 3.0–4.2 V → 5 V regulated | Standard cell, easy sourcing, single-cell BMS is well-trodden | Requires high-efficiency boost converter at 4 A peak — non-trivial |
| **2S Li-ion (2 × 18650 in series) + buck** | 6.0–8.4 V → 5 V regulated | Buck converters at this current are easier and more efficient than equivalent boosts | Requires balancing circuit, 2× the battery cost, larger footprint |
| **2S + parallel banks (more capacity)** | Same as above, more runtime | Longer runtime | Even larger; thermal considerations for the case |

The preliminary requirements doc specifies single-cell + boost. I'm keeping the 2S option open until I do the boost-converter trade study at 4 A peak — if the candidate ICs look marginal in efficiency or thermals, 2S becomes the safer path.

### Charger IC

Candidates to evaluate:

- **TI BQ25895** — popular, I²C, supports USB-C PD input negotiation, fuel-gauge integration via companion IC. Default starting point for single-cell.
- **TI BQ24295** — older, simpler, well-documented. Single-cell.
- **MP2731** — Monolithic Power, 3 A charge current, cheaper than TI. Less documented.
- **For 2S:** the part landscape shifts entirely — BQ25703, BQ25713, or similar 2S-capable charger controllers.

Selection criteria: I²C control, configurable input current limit, simultaneous load + charge, integrated power-path management (so the topology naturally satisfies the "0 ms switchover" requirement without external arbitration logic).

### Input connector

- **USB-C (preferred):** matches the Pi 4 / Pi 5 native power connector, eliminates one cable type, supports PD negotiation if the charger IC speaks PD.
- **DC barrel jack (alternative):** lower-impedance connector, cheaper, doesn't need PD logic, but adds a different cable type to the desk.
- **Both:** OR'd at the input. Wastes board space but maximises flexibility for travel / different environments.

### Output to Pi

- **Pogo pins on GPIO header:** clean, no cable, the Pi powers through the 5V pins of the 2×20 header (which is well-documented for power injection up to 3 A on Pi 4, and similar on Pi 5).
- **USB-C-out to the Pi's USB-C input:** uses the Pi's native power path, including its onboard protection. Adds a short USB-C cable inside the case.
- **Both:** with a jumper or solder bridge to select. Adds complexity but supports tinkering.

GPIO-pin power injection is the typical Pi UPS HAT path. USB-C-out is rarer but cleaner.

---

## Architecture (target)

```
                       ┌────────────────────────────────┐
                       │  Pi 4 / Pi 5                   │
                       │  (lives in same metal case)    │
                       └─────────────┬──────────┬───────┘
                                     │ 5V       │ I²C
                                     │ (GPIO    │ (SDA/SCL on
                                     │  pogo or │  GPIO header)
                                     │  USB-C)  │
                                     ▲          ▲
                       ┌─────────────┴──────────┴───────┐
                       │  PiPower HAT                   │
                       │                                │
                       │  ┌──────────────────────┐      │
   USB-C / barrel  ►───┼─►│  Protection front-end│      │
   (5 V, 3 A)          │  │  Polyfuse + ESD TVS  │      │
                       │  │  + reverse-polarity  │      │
                       │  └──────────┬───────────┘      │
                       │             │ V_in             │
                       │             ▼                  │
                       │  ┌──────────────────────────┐  │
                       │  │  Power-path charger IC   │  │
                       │  │  (e.g. BQ25895)          │  │
                       │  │  - CC/CV charge          │  │
                       │  │  - I_in limit            │  │
                       │  │  - simultaneous load+chg │  │
                       │  │  - 0 ms switchover       │◄─┼──── NTC (battery temp)
                       │  └──┬───────────────┬───────┘  │
                       │     │ V_sys         │ V_bat    │
                       │     │ (4.5–5 V)     ▼          │
                       │     │       ┌──────────────┐   │
                       │     │       │  Battery     │   │
                       │     │       │  18650/21700 │   │
                       │     │       │  (or 2S)     │   │
                       │     │       └──────────────┘   │
                       │     ▼                          │
                       │  ┌──────────────────────────┐  │
                       │  │  5 V converter           │  │
                       │  │  Boost (single-cell)     │  │
                       │  │   OR Buck (2S)           │  │
                       │  │  >4 A peak, >85% η       │  │
                       │  └──────────┬───────────────┘  │
                       │             │ 5 V regulated    │
                       │             ▼                  │
                       │  ┌──────────────────────────┐  │
                       │  │  Output rail to Pi       │  │
                       │  │  (pogo pins / USB-C-out) │  │
                       │  └──────────────────────────┘  │
                       │                                │
                       │  ┌──────────────────────────┐  │
                       │  │  I²C telemetry           │  │
                       │  │  - V_bat                 │  │
                       │  │  - I_charge / I_discharge│  │
                       │  │  - V_in presence         │  │
                       │  │  - SoC (optional fuel    │  │
                       │  │    gauge)                │  │
                       │  └──────────────────────────┘  │
                       └────────────────────────────────┘
```

The **zero-delay switchover** requirement is met by power-path topology, not by a relay. A power-path charger IC keeps both the input and the battery connected to a shared `V_sys` node through internal MOSFETs that arbitrate continuously — when AC power vanishes, `V_sys` is held up by the battery within microseconds because the MOSFETs were already conducting. No code, no timing, no detection latency. This is the right way to do "0 ms switchover" and the wrong way is everything else.

---

## Why X over Y

Some early design choices, captured before they become invisible.

**Power-path charger, not relay switching.** Relay-based UPS designs introduce a switching delay (typically 1–10 ms) during which the Pi's input rail collapses. The Pi has bulk capacitance on its 5 V rail but it's not enough to ride through a real switchover. A power-path charger IC keeps the load continuously powered from whichever source has higher voltage — input or battery — through internal MOSFETs that arbitrate continuously. The "0 ms" requirement is achievable only via this topology. If a relay shows up anywhere in this design, something has gone wrong.

**Lives inside the thermal case, not on top of it.** The standard Pi HAT mounting position (above the Pi, on the GPIO header) doesn't work if the Pi is inside an aluminium case that uses the Pi's SoC as a thermal interface to the case top. PiPower has to fit *alongside* the Pi, sharing the case's internal volume. This drives the form factor: 65 × 56 mm (full HAT footprint) but with mounting options that include pogo-pin contact to the GPIO header from a position offset from the Pi itself — not stacked above it. Mechanical design is unsolved.

**Replaceable cells, not soldered/integrated battery.** 18650s and 21700s are commodity. Cells degrade, get replaced. A soldered LiPo pouch makes the HAT a consumable; a cell holder makes the HAT a durable item. Cell holders are bulkier than pouches, but the case has the space.

**I²C telemetry, not just a "battery low" GPIO.** The Pi needs to know not only "battery is low, shut down" but also "battery is at 60%, fine" and "current draw is 2.3 A right now." I²C gives a structured telemetry stream that can be polled by a userspace daemon, logged, graphed in Grafana, and fed back into the homelab dashboard. A single GPIO interrupt is a 1985-vintage solution.

**No display, no buttons, no fan control.** Scope discipline. The HAT does power, period. Status LEDs are fine. Anything else (charging-current selection, runtime estimate, manual shutdown button) goes via I²C and is implemented in the Pi-side daemon. The HAT itself is dumb hardware with a clean digital interface.

---

## Pi-side Software (planned)

A Python daemon running on the Pi reads telemetry over I²C and reacts:

```
pipowerd  (systemd service)
├─ Polls HAT over I²C at 1 Hz
├─ Logs telemetry to a local sqlite or to MQTT
├─ Triggers `systemctl poweroff` when:
│   - V_bat < threshold (configurable, default 3.2 V/cell)
│   - State-of-charge < threshold (if fuel gauge present)
│   - Battery temperature out of bounds
├─ Publishes to Home Assistant via MQTT for dashboard
└─ Refuses to shut down repeatedly within N minutes (debounce against
   brownout-bounce)
```

Repo will live at `firmware/pipowerd/` once the hardware is closer to bring-up. Language: Python 3.11+. No firmware *on* the HAT — the charger IC is the brain, and it speaks I²C natively.

---

## Form Factor & Mechanical

Locked:
- **Full-size HAT:** 65 × 56 mm, standard 2 × 20 GPIO header position.
- **Coexists with Pi inside an aluminium thermal case** of the Argon ONE / GeeekPi / DeskPi style.

Open:
- Vertical stack vs. horizontal beside-the-Pi placement inside the case (depends on which case I commit to).
- Cell holder type: through-hole spring contacts vs. surface-mount 18650 holders. Through-hole is more reliable for replacement cycles.
- Cooling: the boost / buck converter at 4 A peak will produce real heat. The aluminium case helps if I can get a thermal pad to the case lid; otherwise a small copper pour + thermal vias on the PCB has to do the job. The thermal solution is part of the PCB design, not bolted on later.

---

## Phase Plan

### Phase 0 — Research (current)
- Battery topology trade study (single-cell + boost vs. 2S + buck)
- Charger IC selection (BQ25895 vs. alternatives)
- Aluminium case selection (Argon ONE M.2 vs. GeeekPi vs. DeskPi)
- Pi 4 / Pi 5 GPIO header pin-compatibility audit (specifically: I²C pins, power injection pins, anything that moved between generations)

### Phase 1 — Schematic
- Draw the full schematic in KiCad
- Define I²C register map (or adopt the charger IC's native one)
- Define mechanical envelope based on case selection
- BOM with LCSC part numbers

### Phase 2 — Layout
- 2-layer PCB, solid ground plane, ≥2 mm wide high-current traces
- Thermal pours under switching IC, thermal vias to ground plane
- Decoupling caps placed close to IC pins (charger, boost, MCU if any)
- Minimal switching loop area
- DRC clean **before export**. No exceptions. (See [OakBridge_MkI lessons learned](https://github.com/HighCarlSagan/OakBridge_MkI/blob/main/docs/lessons-learned.md) for why I now treat this as non-negotiable.)

### Phase 3 — Bring-up
- Power-on sequence with dummy load (electronic load or resistive)
- Verify 5 V rail under 0 A, 1 A, 3 A, 4 A
- Verify ripple under load with scope
- Pull input mid-load; verify zero-delay switchover on scope
- Verify charge profile (CC/CV)
- Verify I²C telemetry
- Thermal soak under continuous 3 A operation

### Phase 4 — Software + Integration
- `pipowerd` daemon
- systemd unit, MQTT publishing
- Integration with existing Home Assistant + Grafana on the homelab
- Graceful-shutdown verification (pull input, watch Pi shut down cleanly)

### Phase 5 — Ship
- Public README with hero photo, demo video, BOM, gerbers, STLs, build guide

---

## Risks

| Risk | Severity | Mitigation |
|---|---|---|
| 4 A peak through a 5 V boost is thermally tight on a HAT footprint | HIGH | Trade study with 2S + buck as alternative; thermal pours; case as heatsink |
| Pi 4 / Pi 5 GPIO pin differences break dual-target compatibility | MED | Audit headers early; if conflict, two PCB variants or jumpered straps |
| Cell holder reliability over many replacement cycles | MED | Use through-hole spring contacts, not SMD; mechanical retention |
| Indian-grid brownouts may include voltage sags (not just full cuts) — charger IC must tolerate input sag | MED | Spec charger with low V_in lockout; verify on bench |
| Aluminium case selection drives mechanical envelope — wrong choice means redesign | MED | Lock case choice in Phase 0 before schematic finalisation |
| Boost / buck switching noise couples to Pi audio output / SD card | LOW | Standard layout discipline; ferrite on output if needed |

---

## Repository Structure (planned)

```
PiPower/
├── README.md
├── LICENSE
├── hardware/
│   ├── pipower.kicad_pro
│   ├── pipower.kicad_sch
│   ├── pipower.kicad_pcb
│   ├── Libraries/
│   └── Exports/
│       ├── Gerbers/
│       ├── pipower_bom.csv
│       ├── pipower_top.jpg
│       └── pipower_bot.jpg
├── mechanical/
│   ├── case_fit_mockup.stl
│   └── source/
├── firmware/
│   └── pipowerd/        # Pi-side Python daemon
│       ├── pipowerd.py
│       ├── pipowerd.service
│       └── config.example.yaml
├── docs/
│   ├── requirements.pdf       # Source-of-truth requirements (preliminary)
│   ├── trade-study.md         # Battery topology + charger IC selection
│   ├── i2c-register-map.md
│   └── photos/
└── tests/
    └── bringup-checklist.md
```

Most folders empty for now. They populate as the project moves through phases.

---

## When to Use This / When NOT to Use This

**Use this if:**
- You run a homelab on a Pi 4 or Pi 5 and lose power often enough to care
- You want real telemetry (battery V, charge/discharge current, runtime estimate) not just a "battery low" LED
- Your Pi is in an aluminium thermal case and a stack-on-top HAT doesn't fit
- You like replaceable 18650/21700 cells, not soldered LiPo pouches

**Don't use this if:**
- You need server-grade UPS with hot-swap. This is a small DC UPS for one Pi, not a rack thing.
- You need APC-compatible NUT integration. The telemetry path is I²C → Python daemon → MQTT, not USB-HID.
- You need it now. This is a design doc. No PCB exists yet.

---

## License

**Hardware:** [CERN Open Hardware Licence Version 2 — Permissive (CERN-OHL-P-2.0)](LICENSE)
**Firmware** (when it exists): MIT
**Documentation:** CC-BY-4.0

---

## Related Projects

- **[Carls_Homelab_Public](https://github.com/HighCarlSagan/Carls_Homelab_Public)** — the homelab that PiPower exists to protect.
- **[OakBridge_MkI](https://github.com/HighCarlSagan/OakBridge_MkI)** — sibling hardware project. Source of the "always run DRC before export" lesson that PiPower benefits from for free.

---

## Author

**Mak (Mayank Shrivastava)** — [GitHub @HighCarlSagan](https://github.com/HighCarlSagan) · [highcarlsagan.dev](https://highcarlsagan.dev)
