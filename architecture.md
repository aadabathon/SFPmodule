# SFP BERT — Firmware / RTL Architecture

Scope: digital-side module decomposition, clock domains, interfaces, and build order. Port lists and block responsibilities are defined here; internal logic is specified per-block below.

## System Overview

Two independent compute elements, two independent firmware/RTL codebases:

- **FPGA (dev board, standard side of the link)** — BERT engine. Transceiver configuration, PRBS generation/checking, error counting, results readout. Requires custom RTL.
- **Microcontroller (carrier board)** — low-speed SFP management only. I2C to the module, control pin drive/readback, power sequencing, host reporting. Embedded firmware only.

Line rate (1G vs 10G) is selected at the transceiver wizard. All other content in this document is rate-agnostic.

---

## FPGA / RTL Side

### Phasing

- **Phase 1 — no custom datapath.** Configure the vendor GT transceiver IP via the wizard, run the vendor IBERT core over a loopback fiber, establish a BER baseline. Work at this phase consists of IP instantiation and pin/clock constraints. This validates the electrical link in isolation before any custom logic is introduced.
- **Phase 2 — custom BERT engine** replaces IBERT. This is the primary RTL deliverable defined below.

### Clock Domains

Three distinct clock domains are present in the design:

1. **TX user clock** — drives the PRBS generator. Derived from the reference clock through the GT.
2. **RX recovered clock** — drives the PRBS checker. Recovered by the CDR from the received signal. Asynchronous to the TX clock even in loopback. Error counters reside in this domain.
3. **System clock** — free-running board oscillator (~100 MHz). Control registers, readout, UART.

Counters accumulate in the RX domain and are snapshot-transferred to the system domain. The CDC crossing between these two domains warrants its own module and dedicated testbench.

### Module Decomposition (FPGA)

```
bert_top.sv            — top level; wires blocks, instantiates GT wrapper
  gt_wrapper.sv        — wraps vendor GT IP; exposes clean parallel-data interface
  prbs_gen.sv          — TX domain, parallel PRBS source
  prbs_check.sv        — RX domain, self-synchronizing checker + error detect
  lock_monitor.sv      — RX domain, lock/loss-of-lock FSM
  error_counter.sv     — RX domain, wide bit/error counters, atomic snapshot
  cdc_status.sv        — RX → system clock crossing for the snapshot
  control_regs.sv      — system domain, config + status register file
  uart_iface.sv        — system domain, command/response over UART
  error_inject.sv      — TX domain, single-bit flip for self-test
```

**gt_wrapper** — instantiates the GT transceiver IP from the wizard, configured for the target rate in raw/PMA mode. Exposes `txusrclk`, `rxusrclk`, `tx_parallel_data`, `rx_parallel_data`, plus reset/ready signals. Uses the wizard's example reset controller; the GT reset FSM is not hand-rolled.

**prbs_gen** — parallel PRBS source matching the GT datapath width (e.g. 20 or 40 bits/clock). The LFSR is advanced *N* steps per clock cycle via a combinational XOR network derived from the N-step state-transition matrix. Supported polynomials (ITU standard):
`PRBS7 = x^7+x^6+1`, `PRBS15 = x^15+x^14+1`, `PRBS23 = x^23+x^18+1`, `PRBS31 = x^31+x^28+1`.

**prbs_check** — self-synchronizing checker. Received bits are fed into the LFSR; once the shift register fills (poly-order bits), the next word is predicted from the polynomial and compared against the received word. Mismatches are counted as errors. Lock is acquired without any external alignment, independent of round-trip delay. Note: a single channel bit error produces multiple detected errors (one per polynomial tap) due to the self-sync architecture; raw error count is a fixed multiple of true channel errors and must be normalized when computing BER.

**lock_monitor** — asserts locked after X consecutive clean words; asserts loss-of-lock after Y errors within a sliding window. Gates error counter accumulation so counts are only valid while locked.

**error_counter** — wide counters for total bits received and total errors detected, with `clear` and an atomic snapshot register pair enabling a consistent (bits, errors) read.

**cdc_status** — transfers the error counter snapshot from the RX recovered clock domain to the system clock domain using a Gray-code or request/acknowledge handshake.

**control_regs / uart_iface** — configuration registers (polynomial select, run/stop, clear, inject) and status registers (lock, bits, errors) exposed over a simple UART text protocol: `Pn` set polynomial, `S`/`X` start/stop, `C` clear, `R` read stats. Protocol is human-readable and operable from any serial terminal.

**error_inject** — flips a single TX bit on command, enabling verification that the checker detects errors and that a zero error count reflects a clean channel rather than a broken path.

### Testbenches

- `tb_prbs_gen` — output compared against a golden serial reference model.
- `tb_prbs_check` — feed known PRBS sequence, confirm lock and zero error count; inject a single-bit flip, confirm expected error count; insert arbitrary delays, confirm lock is re-acquired. Validates that the unknown-delay problem is resolved in simulation.
- `tb_bert_loopback` — generator → delay model → checker; sweep delay, confirm lock across all values.

### Hardware Bring-Up Sequence

1. GT wizard + IBERT, loopback fiber, BER baseline. (No custom RTL.)
2. `prbs_gen` + `prbs_check` in simulation with delay model.
3. Wire gen/check to `gt_wrapper`, use GT internal (near-end PMA) loopback — no fiber. Confirm lock on silicon.
4. External loopback fiber. Custom BERT engine now replicates IBERT behavior.
5. Insert experiment. Characterize.

---

## Microcontroller / Carrier Firmware Side

Low-speed SFP management on the carrier MCU, following the hot-plug model from the SFF MSA.

```
main.*          — state machine below
sfp_i2c.*       — I2C master to module: 0xA0 (serial ID / SFF-8472 base),
                  0xA2 (DDM diagnostics). DDM is OPTIONAL on the SFP-M2 — handle
                  "not present" gracefully. Reads vendor/part/rate; polls temp, Vcc,
                  TX power, RX power, bias if DDM present.
sfp_gpio.*      — drives TX_DISABLE; reads TX_FAULT, RX_LOS, Mod_ABS. Honors MSA
                  timing from the datasheet timing table (t_init reset window,
                  TX_DISABLE assert/negate times).
power_seq.*     — enables and sequences the filtered 3.3 V rails; optional current monitor.
host_cli.*      — USB/UART to PC: module inventory, DDM readout, TX_DISABLE control,
                  status reporting. Protocol mirrors the FPGA UART for unified scripting.
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
sfp-bert/
  docs/
    architecture.md
    interface-summary.md
  rtl/
  tb/
  constraints/<board>.xdc
  fw/
  vivado/
```

## Scope Boundaries

- IBERT before custom RTL. 1G before 10G. Internal GT loopback before fiber. Fiber before the experiment.
- The carrier board handles low-speed control and coax only. No FPGA, no clock gen on the carrier in the initial revision.
