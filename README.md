# HAKI (Hardware-Agnostic Kinetic Integrity) Simulator
Live Simulation: https://cjay254.github.io/HAKI-architecture-simulator/
HAKI (Hardware-Agnostic Kinetic Integrity) is a software safety net protecting data centers from Silent Data Corruption. As chips degrade, they make hidden errors. HAKI uses a mathematical framework to verify calculation integrity in real-time with low overhead. This repo holds the architectural blueprints and behavioral simulation.

This is an interactive proof-of-concept simulator demonstrating real-time Physical Proof-of-State (PPoS) detection across 7 CPU cores, with live threat injection, anomaly scoring, and an automated alert chain.

## Why HAKI Exists

Every software system ever written makes one silent, absolute assumption: the CPU gives the correct answer to the correct input. At sub-5nm transistor scales, this assumption is 'empirically false'.

Quantum tunnelling, electromagnetic interference, cosmic rays, and deliberate physical attacks can cause a CPU to silently compute the wrong answer — with no error flag, no crash, no warning. This is called Silent Data Corruption (SDC).

- Meta (2021): ~1 SDC event per few thousand servers **per day** at hyperscale
- Unity Technologies: $110M quarterly revenue loss from corrupted data flowing undetected
- Amazon S3 (2008): a single bit-flip cascaded into a multi-hour outage costing tens of millions per hour
- Toyota (2013): cosmic-ray bit-flips in safety-critical ECUs — a $1.2B legal settlement

The existing defensive stack (ECC RAM, TPMs, ZFS checksums) is reactive, post-hoc, and structurally unable to detect physical attacks *during* computation.

HAKI fills that gap. Inspired by the strain gauge used in civil engineering — a sensor that detects microscopic structural stress before a bridge cracks — HAKI fires cryptographically insignificant "Ghost Instructions" through the CPU every 10ms, measures their sub-nanosecond timing echo, and generates a signed Physical Proof-of-State (PPoS) token for every computation window.

---

## What You Can Do With This Simulator

This simulator is a fully functional browser-based demonstration of the HAKI protocol. No installation required — open `index.html` in any browser.

### Explore the core protocol
- Watch 7 CPU cores continuously emit Ghost Probe results in real time
- See `ΔT` (timing delta), z-score, and INT/FP consistency values update per probe cycle
- Observe the '5-state Integrity Finite State Machine' transition in the State Machine tab: `BOOT → HEALTHY → SUSPECT → DEGRADED → COMPROMISED → QUARANTINE`

### Inject real threat scenarios
Use the 'threat injection panel' (bottom-left) to simulate:

| Threat | What it simulates | HAKI detection method |
|--------|------------------|-----------------------|
| Normal | Baseline healthy operation | ΔT within ±1σ, INT == FP |
| Rowhammer | DRAM row electromagnetic hammering | ΔT spike > 3σ, cache miss surge |
| Voltage Glitch | Momentary voltage drop / hardware implant | ΔT deviation + INT ≠ FP mismatch |
| Mercurial Core | CPU returns wrong answer for specific operations | INT result ≠ FP result on repeated ghost ops |
| Degradation | Long-term hardware wear / thermal drift | Monotonic ΔT upward trend over time |

### Inspect PPoS tokens
The 'Tokens tab' shows live Ed25519-signed Physical Proof-of-State tokens with all fields: timestamp (nanosecond precision), ΔT measurement, cache miss rate, cross-architecture consistency result, thermal/voltage stability classification, and integrity state.

### Trace the alert and response chain
- The **Alert Log** (right panel) records every state transition with severity, z-score, threat classification, and confidence level
- Click any alert to open a 'forensic detail modal': state transition, INT/FP consistency, auto-actions taken, and which stakeholders were notified
- The 'Response Protocol panel' shows who gets alerted at each severity level — from SIEM to PagerDuty to the CTO — and what automatic actions fire (transaction rollback, core quarantine, forensic snapshot)

### Understand the maths
The 'How It Works tab' explains the full detection pipeline in plain language, including the three simultaneous signal cross-checks, the CUSUM change-point detector for gradual wear, and the BLAKE3 cryptographic hash-chain audit log.

---

## How to Use It

### Running locally

```bash
# No dependencies — just open in a browser
open index.html
# or
python3 -m http.server 8080
# then visit http://localhost:8080/index.html
```

### Controls

| Control | Location | What it does |
|---------|----------|-------------|
| **Simulation toggle** | Top of left panel | Start / pause the probe loop |
| **Speed selector** | Next to toggle | Set probe interval: 250ms → 5s |
| **Core list** | Left panel | Click any core to focus its waveform |
| **Threat buttons** | Bottom-left | Inject a threat scenario across all cores |
| **Reset** | Threat panel | Clear all alerts and return all cores to BOOT |
| **Tabs** | Center panel | Switch between Waveform, State Machine, Tokens, How It Works |

### Reading the waveform

The center waveform plots `ΔT` (CPU cycles, y-axis) over the last 60 probe samples (x-axis). The amber dashed line is the calibrated baseline mean (`μ`). Coloured bands mark the ±1σ and ±3σ boundaries. Spikes outside ±3σ trigger the `COMPROMISED` state transition.

### Understanding z-scores

Each probe result is scored as `z = (ΔT − μ) / σ`. The defaults:

- `|z| < 1σ` → **HEALTHY**
- `|z| > 1σ` sustained → **SUSPECT** (deep probe mode activates at 1 kHz)
- `|z| > 3σ` for ≥3 consecutive probes → **COMPROMISED** (data invalidated, alert dispatched)
- INT ≠ FP result → **COMPROMISED** regardless of ΔT (mercurial core confirmed)
- 10+ COMPROMISED events → **QUARANTINE** (core isolated via cgroup)
