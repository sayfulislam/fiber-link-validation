# Fiber Link Validation for GPU / AI Data Centers

A practical field guide to validating GPU-to-TOR fiber links in next-generation AI data centers — written from hands-on validation work, in plain English.

> **Who this is for:** data center validation engineers, network technicians, and anyone bringing up GPU clusters who needs to prove that every link is clean, correct, and ready for traffic.

---

## Table of contents

1. [What is validation?](#1-what-is-validation)
2. [What is root cause analysis?](#2-what-is-root-cause-analysis)
3. [Loopback — definition](#3-loopback--definition)
4. [Terms to know for validation work](#4-terms-to-know-for-validation-work)
5. [Tools, cables and connectors to know](#5-tools-cables-and-connectors-to-know)
6. [Error types you will face during validation](#6-error-types-you-will-face-during-validation)
7. [The 7-step fiber link validation framework](#7-the-7-step-fiber-link-validation-framework)
8. [Validation checklist](#8-validation-checklist)
9. [Useful commands](#9-useful-commands)

---

## 1. What is validation?

**Validation** is the process of proving that a link, a device, or a whole rack works the way the design says it should — *before* it carries production traffic.

In a GPU cluster, a "link" is the fiber path between a server NIC (the GPU host) and the top-of-rack (TOR) switch. Validation checks that path end to end:

| Question | What we check |
|---|---|
| Is the link physically up? | Port state on both the switch and the host |
| Is it cabled to the right place? | LLDP neighbor matches the cabling plan |
| Is the signal clean? | Optical power within range, bit error rate below threshold |
| Are the optics healthy? | Transceiver DOM readings (TX/RX power, temperature, voltage) |
| Is security up? | MACsec / 802.1X session established |
| Can we even see it? | Host telemetry is reaching the validation tool |

Validation is usually run from a portal or tool at the **deployment group** level (a batch of racks). The tool outputs a list of **aggregate errors** per link. The validation engineer's job is to clear those errors by finding and fixing the real cause — not by hiding the symptom.

**Validation is not troubleshooting.** Troubleshooting gets a link working again fast. Validation proves the link meets the standard and documents it. A link can be "up" and still fail validation (for example, high bit errors that FEC is silently correcting).

---

## 2. What is root cause analysis?

**Root cause analysis (RCA)** is a structured way of finding out *why* a problem really happened — not just what went wrong on the surface — so the fix stops it from coming back.

| Level | Meaning | Data center example |
|---|---|---|
| **Symptom** | What people notice | Rack 12 servers are unreachable |
| **Direct cause** | The last event before the symptom | TOR switch port was shut down |
| **Contributing cause** | What made it easier or worse | No alert on port state; on-call engineer was new |
| **Root cause** | The deepest cause we can still control | Config pushes have no peer review, so a typo shut the wrong port |

Two tests for a real root cause:

1. **If you remove it, does the problem stop recurring?** If only for a while, you haven't gone deep enough.
2. **Can we actually control it?** "Human error" and "bad luck" are never root causes — they're where you keep asking *why*.

### The 5 Whys (data center example)

| Why? | Answer |
|---|---|
| Why did rack 12 lose network? | The uplink port on the TOR was shut down. |
| Why was it shut down? | An automation script pushed `shutdown` to the wrong port. |
| Why the wrong port? | The port variable came from an old template with different numbering. |
| Why was an old template used? | There is no single approved template library. |
| Why no approved library? | Standardisation was never scheduled when automation was introduced. **← root cause** |

### RCA in six steps

1. **Define the problem** — what, where, when, how big.
2. **Collect evidence** — timeline from logs, portal history, tickets, interviews. Preserve readings before changing anything.
3. **Find possible causes** — brainstorm across People / Process / Equipment / Environment / Materials / Measurement.
4. **Identify the root cause** — 5 Whys on the strongest candidates; it must explain *all* the evidence.
5. **Fix and prevent** — corrective action (this link) + preventive action (this class of problem), each with an owner and a date.
6. **Verify and close** — confirm the fix works, watch for recurrence, publish the lesson.

A good RCA is **blameless**: it looks at systems and processes, not at who to punish. People only tell the truth when they feel safe.

---

## 3. Loopback — definition

A **loopback** is a test where you plug a small adapter into an optical port that connects the port's **transmit (TX)** fiber directly back into its own **receive (RX)** fiber. The port is now talking to itself — no cable, no patch panel, no far end.

```
   Transceiver                 Loopback plug
   ┌──────────┐                ┌───┐
   │       TX ├───────────────►│   │
   │          │                │   │  (TX looped straight back to RX)
   │       RX │◄───────────────┤   │
   └──────────┘                └───┘
```

**Why it matters:** it is the fastest way to split a fault in two.

| Loopback result | Meaning | Next step |
|---|---|---|
| Port comes up, RX power ≈ TX power | Port and transceiver are **good** | Fault is in the cabling or the far end |
| Port stays down or no RX power | Port or transceiver is **bad** | Swap the optic; if still down, the port is faulty |

**Types of loopback:**

- **LC duplex loopback** — one fiber pair. Used on SFP / SFP28 / LR-type QSFP optics (10G–100G).
- **MPO/MTP loopback** — multi-fiber. Used on SR4 / DR4 / DR8 QSFP optics (40G–400G). On a 12-fiber MPO the plug maps TX1→RX12, TX2→RX11, TX3→RX10, TX4→RX9. **All four lanes must show RX power** — one dead lane points at a single fiber or lane.
- **Host-side loopback** — same LC/MPO plug, but on the server NIC. Check with `ethtool <nic>` (link detected) and `ethtool -m <nic>` (RX power).

Always match the loopback to the optic: fiber mode (OM3/OM4 multimode vs OS2 single-mode), connector polish (UPC vs APC), and for MPO, fiber count and polarity type.

---

## 4. Terms to know for validation work

| Term | Plain-English meaning |
|---|---|
| **TOR** | Top-of-rack switch; the first switch each server connects to |
| **GPU host / HPC node** | The server carrying GPUs; the "host side" of the link |
| **Link** | One fiber path between a host NIC port and a switch port |
| **Transceiver / optic** | The pluggable module (SFP, QSFP, OSFP) that converts electrical signal to light |
| **DOM / DDM** | Digital Optical Monitoring — live readings from the optic: TX power, RX power, temperature, voltage, bias current |
| **TX / RX power** | Light sent / light received, measured in **dBm**. Low RX = weak signal |
| **dB vs dBm** | dBm is absolute power; dB is a difference (loss or gain) between two points |
| **Loss budget** | Maximum dB the link can lose (connectors + fiber + splices) and still work |
| **FEC** | Forward Error Correction — the receiver fixes bit errors on the fly; required on 25G+ links |
| **BER** | Bit Error Rate — fraction of bits received wrong. **Pre-FEC (raw) BER** is measured before correction; **post-FEC** after |
| **Corrected / uncorrected codewords** | FEC counters. Corrected = fixed; **uncorrected = real packet corruption, urgent** |
| **LLDP** | Link Layer Discovery Protocol — each end announces its name and port, so you can see who is really plugged in |
| **Polarity** | Which fiber is TX and which is RX. Reversed polarity = link down with good optics and good cable |
| **MPO polarity types** | Type A (straight), Type B (flipped), Type C (pair-flipped) — must match the design |
| **MACsec** | Link-layer encryption between switch and host; needs matching keys (CAK/CKN) on both ends |
| **802.1X** | Port authentication against a RADIUS server |
| **Err-disabled** | Switch shut the port automatically because a protection feature fired |
| **Flap** | Link going up and down repeatedly |
| **Loopback** | Test plug that returns a port's TX to its own RX (see section 3) |
| **Known-good** | A spare optic or cable you have already proven works, used for swap tests |
| **End-face** | The polished tip of a fiber connector; must be clean and scratch-free |
| **Quality gate** | A step you must pass before moving on (inspect & clean is the gate in this framework) |
| **As-built** | The record of what is actually installed, as opposed to the design |
| **MTTR** | Mean Time To Repair — the main metric a structured method improves |
| **Escalation** | Handing a fault to the next team with evidence (readings, serials, photos) |
| **Deployment group (DG)** | A batch of racks/hosts validated together |

---

## 5. Tools, cables and connectors to know

### Tools

| Tool | What it's for |
|---|---|
| **Fiber inspection scope** (video probe, 1.25 mm / 2.5 mm / MPO tips) | Inspect every end-face before connecting |
| **One-click cleaner** (1.25 mm LC, 2.5 mm SC, MPO/MTP) | Dry-clean connector faces; most common fix for low RX |
| **Lint-free wipes + IPA** | Wet-dry clean for stubborn contamination |
| **Optical power meter (OPM)** | Measure TX/RX in dBm; needs 850 nm and 1310 nm, LC + MPO adapters |
| **Visual fault locator (VFL)** | Red laser to trace a fiber and find breaks or bends (never look into the fiber) |
| **LC duplex loopback** (OM4 and OS2) | Loopback test on SFP/SFP28/LR-QSFP ports |
| **MPO/MTP loopback** (12F UPC multimode, 12F APC single-mode) | Loopback test on SR4/DR4 QSFP ports |
| **Known-good optics** (same part numbers as installed) | Swap test |
| **Known-good patch cords** (LC and MPO, 1–3 m, labelled TEST) | End-to-end test (steps 6–7) |
| **Dust caps** | Cap every open port, optic and loopback when not in use |
| **Labels / label printer** | Keep cabling records true |
| **Laptop with console / SSH** | Read port status, DOM, LLDP, FEC counters |

### Cables and fiber

| Item | Notes |
|---|---|
| **OM3 / OM4 / OM5** | Multimode fiber (aqua / aqua-violet / lime). 850 nm. Short reach (SR) optics. |
| **OS2** | Single-mode fiber (yellow). 1310 / 1550 nm. LR / DR / FR optics. |
| **Patch cord** | Short jumper from optic to panel, or optic to optic |
| **Trunk cable** | Multi-fiber backbone between panels (MPO on both ends) |
| **Cassette** | Breaks an MPO trunk out into LC ports in a patch panel |
| **DAC** | Direct Attach Copper — short in-rack copper cable with built-in connectors |
| **AOC** | Active Optical Cable — fiber with fixed optics on both ends |
| **Breakout cable** | 1×100G/400G to 4×25G/100G |

### Connectors

| Connector | Where you'll see it |
|---|---|
| **LC** (duplex) | SFP, SFP28, LR/FR QSFP; the most common DC connector |
| **MPO / MTP** (8, 12, 16, 24 fiber) | SR4, DR4, DR8 QSFP/OSFP; trunk cables |
| **SC** | Older patch panels, some DR1 optics |
| **UPC** (blue) vs **APC** (green) | Polish type; never mate UPC to APC |
| **CS / SN / MDC** | Very-small-form-factor duplex connectors on 400G/800G optics |

### Transceivers

| Form factor | Speed |
|---|---|
| **SFP / SFP+ / SFP28** | 1G / 10G / 25G |
| **QSFP+ / QSFP28** | 40G / 100G |
| **QSFP-DD / OSFP** | 400G / 800G (OSFP common on GPU fabrics) |
| **SR / LR / DR / FR** | Short-reach multimode / 10 km / 500 m / 2 km single-mode |

---

## 6. Error types you will face during validation

Errors come in pairs. **No prefix = switch side.** **`HOST_` prefix = server / NIC side.**

| # | Error | Meaning | First thing to check |
|---|---|---|---|
| 1 | **INTERFACE** | Switch port down, wrong speed/FEC, flapping, or high error count | Admin state, speed/FEC match, err-disable reason |
| 2 | **HOST_INT** | NIC down on the server | `ip link` / `ethtool <nic>`; NO-CARRIER = physical, DOWN = config |
| 3 | **OPTICS** | Switch optic DOM out of range | Low RX = fiber or far-end TX; low TX = replace the optic |
| 4 | **HOST_OPT** | Host optic DOM out of range | `ethtool -m <nic>`; "unsupported module" in `dmesg` = NIC firmware |
| 5 | **RAW_BER** | Pre-FEC bit errors high on switch side | Clean, reseat, then swap cable; uncorrected FEC = urgent |
| 6 | **HOST_RAW_BER** | Pre-FEC bit errors high on host side | Confirm FEC mode matches the switch, then treat like RAW_BER |
| 7 | **LLDP / HOST_LLDP** | Neighbor seen ≠ cabling record | `lldpctl` vs the plan; swapped cable or outdated record |
| 8 | **LLDP_MISSING / HOST_LLDP_MISSING** | No neighbor seen at all | Link up? LLDP enabled on the port? Host online? |
| 9 | **AUTH / HOST_AUTH** | MACsec / 802.1X session not up | Compare key name (CKN) and cipher on both ends, bounce the port |
| 10 | **HOST_METRICS_MISSING** | No telemetry from the host | Host off, booting, or agent down — **fix first; other HOST_* errors are false until then** |
| 11 | **NOT_READY_FOR_LVV** | Host not ready for validation | Not a link fault; wait for host bring-up |

**Two rules that save hours:**

- Same error on **both** sides → suspect the **cable**.
- Error on **one side only** → suspect the **optic at the far end**.

**Typical thresholds** (use your design values):

| Check | Pass |
|---|---|
| Optical RX power | Within the optic's DOM range; typical window −4 dB to +4.5 dB vs reference |
| Pre-FEC BER | ≤ 1e-7 sustained; zero uncorrected codewords |
| Link state | Up at designed speed, correct FEC, zero flaps over 15 min |
| End-face | IEC 61300-3-35 pass: no defects in the core zone, no contamination |
| LLDP | Switch name + port match the record on both ends |

---

## 7. The 7-step fiber link validation framework

Work the steps **in order**. Stop as soon as the link is up, confirm it in the validation tool, and close the ticket.

### Step 1 — Physical inspection
- Check patch cords, connectors and transceiver seating. Look for damage, tight bends, loose cages.
- Confirm labels match the cabling records on both ends.
- Confirm the port is enabled and configured (speed, FEC, no `shutdown`).
- **Link up? → close.**

### Step 2 — Loopback + optical power test
- Disconnect the circuit. Fit the right loopback (LC or MPO) on the port.
- Check port status. Measure TX/RX power with the OPM and compare to DOM.
- **Port up with expected power → port and optic are good, go to step 4. Port down → suspect port or optic; swap in a known-good optic.**

### Step 3 — Fiber inspection & cleaning *(quality gate)*
- Disconnect. Inspect **every** end-face with the scope — both ends, plus the loopback plug.
- Clean with the one-click cleaner. Re-inspect. Never reconnect a dirty connector.
- **Pass = clean, no contamination, ready to reconnect.**

### Step 4 — Original circuit verification
- Remove the loopback. Reconnect the original circuit. Check port status.
- **Link up? → close.**

### Step 5 — Polarity verification (A ↔ B)
- Swap TX/RX on the patch cord. Reconnect. Check link status.
- **Link up? → polarity issue found; service restored. Record it so the record gets fixed.**

### Step 6 — Temporary end-to-end test
- Run a known-good cable directly between the two endpoints, bypassing panels and trunks.
- **Link up? → the permanent cabling path is faulty. Investigate trunk / cassette / panel.**

### Step 7 — Test-cable verification
- Replace with a second known-good cable and repeat.
- **Link up? → the previous test cable was defective. Still down? → escalate.**

### Escalate if
- Link still down after step 7
- Bootstrap failure or no management connectivity on the host
- Power in spec but link down
- Multiple swaps fail, trunk damage, or documentation mismatch

Escalate **with data**: readings, serial numbers, photos, and what you already swapped.

### Root cause categories (what you'll usually find)

| Category | Found at |
|---|---|
| Contamination (dirty end-face) | Step 3 |
| Patch cord failure | Step 6/7 |
| Polarity issue | Step 5 |
| Cabling infrastructure (trunk, connector, cassette, splice) | Step 6 |
| Transceiver failure | Step 2 |
| Port failure | Step 2 |
| Configuration issue | Step 1 |

### Golden rules
1. Change **one thing at a time**, then re-test.
2. Inspect and clean **every** connector before you reconnect it — both ends.
3. Record every power reading and every swap. The ticket is the evidence.
4. Re-validate in the tool after each step, not only at the end.
5. Escalate with data, not with "still down".

---

## 8. Validation checklist

Copy this into your ticket and tick as you go.

```
Link ID: __________   Switch/port: __________   Host/NIC: __________   Date: ________

[ ] Physical inspection completed (cords, seating, labels, port config)
[ ] Loopback test completed
[ ] Power meter readings recorded — TX: ____ dBm   RX: ____ dBm
[ ] Fiber end-faces inspected and cleaned (both ends + loopback)
[ ] Port and transceiver validated
[ ] Original circuit retested
[ ] Polarity verified
[ ] Temporary end-to-end cable tested
[ ] Second known-good cable used
[ ] Root cause identified: ______________________________
[ ] Corrective action completed (what / serials): __________
[ ] Preventive action logged (if any): __________
[ ] Link re-validated in the tool — status: PASS / FAIL
[ ] Tracker and as-built records updated
[ ] Ticket closed / escalated to: __________
```

---

## 9. Useful commands

**Switch (NX-OS / EOS / SONiC style — adjust to your platform)**

```bash
show interface <port> status
show interface <port>                         # errors, CRC, flaps, last change
show interface <port> transceiver details     # DOM: TX/RX power, temp, voltage
show interface <port> fec                     # FEC mode, corrected/uncorrected
show interface <port> phy detail              # pre-FEC BER (EOS)
show lldp neighbors interface <port>
show macsec status interface <port>
show logging | include <port>
clear counters interface <port>
```

**Host (Linux)**

```bash
ip -br link show                              # interface state
ethtool <nic>                                 # link detected, speed, auto-neg
ethtool -i <nic>                              # driver + firmware version
ethtool -m <nic>                              # optic type + DOM readings
ethtool --show-fec <nic>                      # FEC mode
ethtool -S <nic> | grep -iE 'err|drop|crc|fec'
lldpctl <nic>                                 # neighbor switch + port
dmesg -T | grep -iE 'sfp|module|<nic>'
ip macsec show                                # MACsec SA status
systemctl status lldpd node_exporter          # agents the validation tool depends on
```

---

## Final Note

Content is shared for educational purposes. Data and Thresholds are typical values — better to use device and vendor specifications.
