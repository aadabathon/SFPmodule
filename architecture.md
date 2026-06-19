# SFP BERT — Firmware / RTL Architecture

Scope of this doc: the *layout* of the digital side — module decomposition, clock
domains, interfaces, build order. It deliberately stops at port lists and
responsibilities. The logic inside each block is yours to write; the parts marked
**(derive yourself)** are exactly where the understanding lives, so don't farm them out.

## System recap (who computes what)

Two separate compute elements, two separate firmware/RTL codebases:

- **FPGA (dev board, standard side of the link)** — the BERT engine. Transceiver
  config, PRBS generate/check, error counting, results readout. This is the part
  that needs real RTL.
- **Microcontroller (carrier board)** — low-speed SFP management only. I2C to the
  module, drive/read the control pins, power sequencing, host reporting. Bounded
  embedded work, squarely in your wheelhouse.

The line rate (1G vs 10G) plugs in at the transceiver wizard. Everything else in
this doc is rate-agnostic, so none of it is blocked on Rob.

---

## FPGA / RTL side

### Phasing (matches the IBERT-first plan — don't skip phase 1)

- **Phase 1 — no custom datapath.** Configure the vendor GT transceiver IP via the
  wizard, run the vendor IBERT core over a loopback fiber, get a BER baseline. The
  only "RTL" here is IP instantiation + pin/clock constraints. This proves the
  electrical link before any of your logic can be blamed for a bad result.
- **Phase 2 — your BERT engine** replaces IBERT. This is the real deliverable below.

### The crux: three clock domains

This is the thing to hold in your head, because it's where the non-obvious bugs hide.

1. **TX user clock** — drives the PRBS generator. Derived from the reference clock
   through the GT.
2. **RX recovered clock** — drives the PRBS checker. The CDR recovers this from the
   *returned* data. It is asynchronous to your TX clock even in pure loopback, because
   it's recovered from the received signal, not phase-locked to your transmitter. The
   error counters live here.
3. **System clock** — free-running board oscillator (~100 MHz). Control registers,
   readout, UART. The microcontroller also lives at this conceptual level.

Counters live in the RX domain; you snapshot them across to the system domain. That
CDC crossing is the single place most likely to give you a subtle wrong-number bug,
so it gets its own module and its own testbench.

### Module decomposition (FPGA)

```
bert_top.sv            — top level, wires the blocks, instantiates the GT wrapper
  gt_wrapper.sv        — wraps vendor GT IP; clean parallel-data interface
  prbs_gen.sv          — TX domain, parallel PRBS source            (derive yourself)
  prbs_check.sv        — RX domain, self-sync checker + error detect (derive yourself)
  lock_monitor.sv      — RX domain, lock/loss-of-lock FSM
  error_counter.sv     — RX domain, wide bit/error counters, atomic snapshot
  cdc_status.sv        — RX -> sys clock crossing for the snapshot   (be careful here)
  control_regs.sv      — sys domain, config + status register file
  uart_iface.sv        — sys domain, command/response over UART
  error_inject.sv      — TX domain, optional single-bit flip for self-test
```

**gt_wrapper** — instantiate the GT transceiver IP from the wizard, configured for
the target rate in raw/PMA mode. Expose `txusrclk`, `rxusrclk`, `tx_parallel_data`,
`rx_parallel_data`, plus reset/ready. **Do not hand-roll the GT reset sequence** —
wrap the wizard's example reset controller. That FSM is a known time sink.

**prbs_gen** *(derive yourself)* — parallel PRBS matching the GT datapath width
(e.g. 20 or 40 bits/clk). The catch: you must advance the LFSR *N* steps per clock,
not one. That's a parallel LFSR — each output and next-state bit becomes a combinational
XOR of current-state bits (the N-step state-transition matrix). Deriving that XOR
network by hand for one polynomial is the single best RTL exercise in this whole
project. Polynomials (ITU standard):
`PRBS7 = x^7+x^6+1`, `PRBS15 = x^15+x^14+1`, `PRBS23 = x^23+x^18+1`, `PRBS31 = x^31+x^28+1`.

**prbs_check** *(derive yourself)* — self-synchronizing checker. Feed received bits
into the LFSR; once the register fills (poly-order bits), predict the next word from
the polynomial and compare to what actually arrived. Mismatches = errors. It locks on
its own regardless of round-trip delay — that's the entire reason you don't need to
align anything. One property to know and document: in a self-sync checker a single
channel bit error shows up as multiple detected errors (one per tap), so raw error
count is a fixed multiple of true channel errors. Note the factor when you report BER.

**lock_monitor** — declares locked after X consecutive clean words, loss-of-lock after
Y errors in a window. Gates whether the counters are valid (don't accumulate errors
while unlocked).

**error_counter** — wide counters for total bits and total errors, with a `clear` and
an atomic snapshot register pair so a consistent (bits, errors) can be read together.

**cdc_status** *(be careful here)* — moves the snapshot from RX recovered clock to
system clock. Gray-code or request/acknowledge handshake. This is where a lazy
crossing produces numbers that are *almost* right.

**control_regs / uart_iface** — config (polynomial, run/stop, clear, inject) and status
(lock, bits, errors) over a simple UART text protocol: e.g. `Pn` set poly, `S`/`X`
start/stop, `C` clear, `R` read stats. Debuggable from any serial terminal. AXI-Lite +
a soft core is a later option, not a first call.

**error_inject** (optional, build it anyway) — flip one bit on command so you can prove
the checker actually catches errors. Cheap, and it buys you real confidence during
bring-up instead of trusting a zero you can't distinguish from a dead path.

### Testbenches (do these before hardware)

- `tb_prbs_gen` — compare against a golden serial reference model.
- `tb_prbs_check` — feed known PRBS, confirm lock + zero errors; inject one flip,
  confirm exactly the expected error count; insert arbitrary delay, confirm it still
  locks. This last one is your proof that the unknown-delay problem is solved *in sim*.
- `tb_bert_loopback` — gen → delay model → check, sweep the delay, confirm always locks.

### Hardware bring-up order

1. GT wizard + IBERT, loopback fiber, BER baseline. (No custom RTL.)
2. prbs_gen + prbs_check in sim with the delay model.
3. Wire gen/check to gt_wrapper, use the GT's **internal (near-end PMA) loopback** —
   still no fiber. Confirm lock on real silicon.
4. External loopback fiber. You've now reproduced IBERT with your own engine.
5. Insert the experiment. Characterize.

---

## Microcontroller / carrier firmware side

Low-speed SFP management on the carrier's MCU. Hot-plug model straight out of the SFF MSA.

```
main.*          — the state machine below
sfp_i2c.*       — I2C master to module: 0xA0 (serial ID / SFF-8472 base),
                  0xA2 (DDM diagnostics). DDM is OPTIONAL on the SFP-M2 — handle
                  "not present" gracefully. Read vendor/part/rate; poll temp, Vcc,
                  TX power, RX power, bias if DDM exists.
sfp_gpio.*      — drive TX_DISABLE; read TX_FAULT, RX_LOS, Mod_ABS. Honor the MSA
                  timing from the datasheet's timing table (t_init reset window,
                  TX_DISABLE assert/negate times).
power_seq.*     — enable/sequence the filtered 3.3 V rails, optional current monitor.
host_cli.*      — USB/UART to the PC: module inventory, DDM readout, set TX_DISABLE,
                  report status. Mirror the FPGA's UART so the whole rig is scriptable.
```

Tying state machine:

```
ABSENT --(Mod_ABS low: module inserted)--> INIT
INIT   --(wait t_init, clear TX_DISABLE)--> ENABLED
ENABLED--(TX_FAULT or RX_LOS asserted)----> FAULT
FAULT  --(operator reset)-----------------> INIT
```

---

## Suggested repo layout

```
sfp-bert/
  docs/
    architecture.md          (this file)
    interface-summary.md     (your by-hand datasheet table — you write this)
  rtl/                       (FPGA, the modules above)
  tb/                        (the three testbenches above)
  constraints/<board>.xdc
  fw/                        (carrier MCU firmware above)
  vivado/                    (project / IP build scripts)
```

## Anti-scope reminders

- IBERT before custom RTL. 1G before 10G. Internal GT loopback before fiber. Fiber
  before the experiment.
- The carrier board is low-speed control + coax only. No FPGA on it, no clock gen, first pass.
- The parts marked *(derive yourself)* are the point — they're where you stop renting
  understanding and start owning it.
