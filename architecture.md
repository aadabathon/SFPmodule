# SFP Carrier Board — System Architecture

Scope: system-level architecture for the SFP carrier board that fronts a long-wavelength
free-space optical communications experiment. Covers the data path, the carrier board (the
active deliverable), the board revision plan, and two deferred subsystems (FPGA BERT engine,
on-board MCU firmware) that are documented for future reference but are **not** in the current
scope.

Status: mk1 routed and DRC-substantially-clean. mk2 (10 Gb/s) in progress.

---

## System Overview

The system is a reflective optical link. A host sends a known data pattern out through the
SFP module, the pattern traverses the experimental optical path and returns after some delay,
and the host compares returned bits against sent bits to measure bit error rate.

Data path:

```
PC (host)
  └─ fiber ─▶ SFP module (on carrier board)
                └─ SMA coax ─▶ experimental laser ─▶ free space ─▶ optical device / mirror
                                                                        │ (reflection)
        SMA coax ◀─ detector ◀───────────────────────────────────────┘
          └─▶ SFP module ─ fiber ─▶ PC (compares returned vs. sent → BER)
```

Key architectural facts (current):

- **The PC is the host and the bit-error tester.** The PC's NIC/SFP drives the data and
  performs the comparison in software. No FPGA or MCU is required on the standard side for the
  present setup.
- **The carrier board is a media/interface board**, not a compute board. It holds the SFP
  module, powers it, and breaks its two interfaces out: high-speed data to coax, low-speed
  control to a header.
- **The laser and detector are off-board**, each with their own power supplies, reached over
  SMA coax. On-board integration of these is a possible future revision (see Deferred, below).
- **Line rate target: 10 Gb/s** (SFP+, 100 Ω differential).

---

## Carrier Board (Active Deliverable)

### Function

Provide a standard, characterized SFP interface to the experiment: hold the module, supply
filtered power, route the high-speed pairs out to coax for the laser/detector, and expose the
low-speed control/management pins on a header so any host or dev board can manage the module.

### Blocks

- **SFP cage / connector** — Samtec SFPK-SL (solder, edge-mount; body overhangs the board edge).
- **Power** — single +3.3 V rail in through the header, split through a pi-filter (L1/L2 series
  inductors, C1 bulk, C2/C3 bypass) into the module's VccT and VccR to isolate TX-side supply
  noise from the RX-side supply. No on-board regulator; power is provided by whatever drives
  the header.
- **High-speed data** — the two differential pairs (TD±, RD±) route from the cage to four SMA
  coax connectors, which carry the signals out to the experimental laser and detector.
- **Low-speed control** — TX_DISABLE, TX_FAULT, RX_LOS, MOD_ABS, and the I2C bus (SCL/SDA, with
  pull-ups) break out to an 8-pin header (P1: +3V3, GND, SDA, SCL, TX_DIS, TX_FLT, RX_LOS,
  MOD_ABS). Any host/dev board plugs in here to manage the module. No MCU on the board.
- **Ground** — all VeeT/VeeR/SHIELD pins tie to the ground plane/pours; not individually routed.

### Stackup (4-layer)

4 layers, required at 10 Gb/s for controlled 100 Ω differential impedance and a continuous
ground reference under the high-speed pairs. Final arrangement to be confirmed against the fab's
impedance calculator; two candidates:

- **Signal / GND / PWR / Signal** — textbook high-speed stackup (Rob's/Grok's suggestion).
- **Signal / GND / GND / Signal** — symmetric double-ground variant; likely preferable here
  since board power is trivial (a few 3.3 V pins) and does not justify a dedicated power plane,
  while double ground improves the high-speed reference and crosstalk.

High-speed pairs route on an outer layer (Top) referenced to the adjacent inner GND plane
through the prepreg.

### Revision Plan

- **mk1 — complete.** Carrier routed, 4-layer, DRC substantially clean (remaining items:
  inherent SFP solder-mask slivers, cosmetic silkscreen, diff-pair length match). Validates the
  SFP-to-coax path and the overall layout.
- **mk2 — in progress, 10 Gb/s. Deliverable target 2026-07-17.** Refined carrier; task list
  below.
- **mk3 — deferred.** Potential on-board integration of the laser driver and detector/TIA,
  gated on obtaining the part numbers and any existing driver-IC / TIA reference designs. CDR
  (e.g. SY87701L) footprint provision also belongs to this stage.

### mk2 Task List

1. **Rebuild the SFP footprint** to match Samtec's recommended PCB layout drawing. The
   library/auto-generated footprint carried an over-restrictive multi-layer keepout (blocking
   tracks on all layers rather than only under the true cage body); Note 3 of the drawing
   permits traces in the pad-adjacent zone. A corrected footprint removes most of the escape
   difficulty.
2. **Set the differential pairs to true 100 Ω** using the impedance calculator with the final
   10G stackup (trace width/spacing over the prepreg-to-GND height).
3. **Tighten intra-pair length matching.** mk1 skew: RD ≈ 8.8 mm, TD ≈ 4.0 mm — both well over
   tolerance. Serpentine-tune the shorter trace of each pair to < ~5 mil.
4. **Widen SMA spacing** so the coax connectors physically fit and the pairs launch cleanly.
5. **Convert high-speed runs to GCPW** (grounded coplanar waveguide) after the fan-out —
   ground copper alongside the pairs on the signal layer, referenced to the plane below.
6. **Add periodic ground stitching vias** alongside the pairs (spacing well under λ/10 at 10G,
   ~1–2 mm) to keep the return path and side-ground continuous.
7. **Confirm stackup and CDR footprint provision** (unpopulated for now).

---

## Deferred: On-Board Laser / Detector Integration (mk3)

Rob's stated ideal is to eventually host the laser and detector on the same board so a single
control point (one microcontroller) manages the module, laser, and detector, including TEC
control. This is a significant scope increase — it moves the board from a digital/connector
design into a high-speed **analog optical front-end** (laser driver, photodiode + TIA,
bias/TEC), which is the hardest and most layout-sensitive part of the experiment.

Gating requirement before this can be scoped: **the laser and detector part numbers, and
whether a driver IC / TIA / reference design exists.** With a reference design, this is an
ambitious-but-feasible board addition; without one, it is a from-scratch analog optical
front-end and should be treated as its own effort.

---

## Deferred / Optional: FPGA BERT Engine

**Status: not in current scope.** The PC performs the bit-error test in software, so a dedicated
FPGA-based tester is not required. This section is retained as a reference design for the case
where a standalone FPGA BERT is later preferred over the PC host (e.g. for delay-tolerant
characterization independent of a PC NIC). If pursued, it lives on a separate FPGA dev board on
the standard side of the link — never on the carrier.

### Phasing

- **Phase 1 — no custom datapath.** Configure the vendor GT transceiver IP via the wizard, run
  the vendor IBERT core over a loopback fiber, establish a BER baseline. IP instantiation and
  pin/clock constraints only. Validates the electrical link before any custom logic.
- **Phase 2 — custom BERT engine** replaces IBERT.

### Clock Domains

1. **TX user clock** — drives the PRBS generator; derived from the reference clock through the GT.
2. **RX recovered clock** — drives the PRBS checker; recovered by the CDR from the received
   signal, asynchronous to TX even in loopback. Error counters reside here.
3. **System clock** — free-running board oscillator (~100 MHz); control registers, readout, UART.

Counters accumulate in the RX domain and are snapshot-transferred to the system domain; the CDC
crossing warrants its own module and dedicated testbench.

### Module Decomposition

```
bert_top.sv            — top level; wires blocks, instantiates GT wrapper
  gt_wrapper.sv        — wraps vendor GT IP; clean parallel-data interface
  prbs_gen.sv          — TX domain, parallel PRBS source
  prbs_check.sv        — RX domain, self-synchronizing checker + error detect
  lock_monitor.sv      — RX domain, lock/loss-of-lock FSM
  error_counter.sv     — RX domain, wide bit/error counters, atomic snapshot
  cdc_status.sv        — RX → system clock crossing for the snapshot
  control_regs.sv      — system domain, config + status register file
  uart_iface.sv        — system domain, command/response over UART
  error_inject.sv      — TX domain, single-bit flip for self-test
```

- **gt_wrapper** — instantiates the GT transceiver IP for the target rate in raw/PMA mode;
  exposes `txusrclk`, `rxusrclk`, `tx_parallel_data`, `rx_parallel_data`, reset/ready. Uses the
  wizard's example reset controller (GT reset FSM is not hand-rolled).
- **prbs_gen** — parallel PRBS matching the GT datapath width (e.g. 20/40 bits/clock); LFSR
  advanced *N* steps per clock via a combinational XOR network from the N-step transition matrix.
  Polynomials: `PRBS7 = x^7+x^6+1`, `PRBS15 = x^15+x^14+1`, `PRBS23 = x^23+x^18+1`,
  `PRBS31 = x^31+x^28+1`.
- **prbs_check** — self-synchronizing checker; received bits feed the LFSR, the next word is
  predicted and compared, mismatches counted. Locks without external alignment, independent of
  round-trip delay. A single channel bit error yields multiple detected errors (one per tap);
  raw count is a fixed multiple of true errors and must be normalized for BER.
- **lock_monitor** — asserts locked after X clean words, loss-of-lock after Y errors in a
  window; gates counter accumulation.
- **error_counter** — wide total-bit / total-error counters, `clear`, atomic snapshot pair.
- **cdc_status** — RX→system snapshot transfer via Gray-code or req/ack handshake.
- **control_regs / uart_iface** — config (poly select, run/stop, clear, inject) and status
  (lock, bits, errors) over a UART text protocol: `Pn` set poly, `S`/`X` start/stop, `C` clear,
  `R` read stats.
- **error_inject** — flips one TX bit on command to verify the checker detects errors.

### Testbenches

- `tb_prbs_gen` — output vs. golden serial reference.
- `tb_prbs_check` — feed known PRBS, confirm lock + zero errors; inject a flip, confirm expected
  count; insert arbitrary delays, confirm re-lock. Proves delay-tolerance in simulation.
- `tb_bert_loopback` — generator → delay model → checker; sweep delay, confirm lock everywhere.

### Bring-Up Sequence

1. GT wizard + IBERT, loopback fiber, BER baseline (no custom RTL).
2. `prbs_gen` + `prbs_check` in simulation with delay model.
3. Wire gen/check to `gt_wrapper`, GT internal (near-end PMA) loopback — no fiber. Confirm lock.
4. External loopback fiber — custom engine replicates IBERT.
5. Insert experiment; characterize.

---

## Deferred / Optional: On-Board MCU Firmware

**Status: not in current scope.** mk1/mk2 break the low-speed control out to a header rather than
placing an MCU on the board; management is handled by the host or a plugged-in dev board. This
section applies only if a future revision populates an on-board MCU (or if a dev board on the
header runs this firmware). Follows the SFF MSA hot-plug model.

```
main.*          — state machine below
sfp_i2c.*       — I2C master: 0xA0 (serial ID / SFF-8472 base), 0xA2 (DDM diagnostics).
                  DDM is OPTIONAL on the SFP-M2 — handle "not present" gracefully. Reads
                  vendor/part/rate; polls temp, Vcc, TX power, RX power, bias if DDM present.
sfp_gpio.*      — drives TX_DISABLE; reads TX_FAULT, RX_LOS, Mod_ABS. Honors MSA timing
                  (t_init reset window, TX_DISABLE assert/negate).
power_seq.*     — enables/sequences the filtered 3.3 V rails; optional current monitor.
host_cli.*      — USB/UART to PC: inventory, DDM readout, TX_DISABLE control, status.
```

State machine:

```
ABSENT --(Mod_ABS low: module inserted)--> INIT
INIT   --(wait t_init, clear TX_DISABLE)--> ENABLED
ENABLED--(TX_FAULT or RX_LOS asserted)----> FAULT
FAULT  --(operator reset)-----------------> INIT
```

---

## Repository Layout

```
sfp-carrier/
  docs/
    architecture.md          (this file)
    interface-summary.md     (datasheet pinout / supply / control summary)
    mk1-review.pdf           (layer plots, 3D, schematics — sent to Rob)
  hardware/
    <altium project>         (schematics, PCB, footprint library)
  rtl/                       (deferred — FPGA BERT, if pursued)
  fw/                        (deferred — on-board MCU firmware, if pursued)
```

## Scope Boundaries

- **Active:** the carrier board (mk2, 10 Gb/s). PC is the host; the carrier is interface/media only.
- The carrier handles power, high-speed coax breakout, and low-speed control breakout — no
  compute (no FPGA, no MCU, no clock gen) on the board.
- **Deferred:** on-board laser/detector (mk3, gated on part data + reference design); CDR
  footprint provision; FPGA BERT engine (only if PC-host is replaced); on-board MCU firmware
  (only if an MCU is populated).
- Sequencing for any BERT work, if pursued: IBERT before custom RTL; internal GT loopback before
  fiber; fiber before the experiment.