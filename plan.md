# W65C816/RP2354B Computer Plan

## Goal

Keep the W65C816 as the computer that runs applications and game logic. Treat
the RP2350 as a memory-mapped video, audio, storage, input, networking, and DMA
chipset. The design should feel closer to an Apple IIgs or the intended
Commodore 65 architecture than to an ARM computer with a 65C816 attached.

The ESP32-C3 remains a networking coprocessor behind the RP2354B and owns the
W5500 Ethernet controller. External RP2354B PSRAM is primarily media, buffer,
and accelerator memory.

## Agreed Revision-A Architecture

### Mechanical and connector contract

- Keep the 98-pin ISA-style edge connector and matching right-angle adapter.
- This is a mechanically convenient, robust connector only. The electrical
  interface is project-specific and is not ISA-compatible.
- The existing connector pinout is provisional and may be revised during PCB layout. However,
  every change must be reviewed individually, as any electrical modification may require pin reassignment.
- Update `AGENTS.md` and `board_design.txt` later to remove the conflicting
  2x20 and 80-contact connector descriptions.

### RP2354B bus role

- The RP2354B is a memory-mapped I/O device.
- The RP2354B bus interface consumes `A0-A9`, `RP_CS_N`, `PHI2`, `RWB`,
  `VDA`, `VPA`, `RESET_N`, `RDY`, and an 8-bit data bus.
- Do not connect `A10-A23` directly to RP2350 GPIO for revision A.
- The mainboard connector may retain `A10-A23` as reserved raw-bus signals for
  future expansion or a hardware bus-controller daughterboard.
- The RP2350 is not a transparent system MMU in revision A. It provides
  MMU-like PSRAM address/window registers, FIFO transfers, and DMA commands.
- A future transparent MMU or executable PSRAM design requires deterministic
  hardware such as a CPLD that sees `A0-A23` and participates in RAM/ROM chip
  select generation.

### Revision-A memory map

| Address range | Function |
|---|---|
| `$F00000-$F000FF` | RP2350 registers |
| `$F00100-$F001FF` | Command FIFO write window |
| `$F00200-$F002FF` | Response FIFO read window |
| `$F00300-$F003FF` | Input/event FIFO read window |

`A8-A9` select the four pages. Within each FIFO page, every address may act as
the same FIFO data port. This permits efficient sequential 65C816 transfers.

The existing decode selects the complete `$F00000-$F1FFFF` region. `A10-A16`
are not decoded, so the 1 KiB interface intentionally mirrors 128 times through
that 128 KiB region in revision A. Tightening the decode is deferred.

Do not implement the proposed `$F10000-$F1FFFF` direct shared-memory window in
revision A. It conflicts with the narrow address interface and would require
`A0-A15` plus a substantially more complex bus bridge.

### PSRAM access

Expose PSRAM through indirect registers and chipset commands:

- `MEM_ADDR0-MEM_ADDR3`
- `MEM_DATA`, with optional auto-increment
- `MEM_CONTROL`
- `MEM_STATUS`
- command-FIFO DMA, copy, fill, upload, and blitter operations

Use PSRAM for framebuffers, tiles, sprites, fonts, audio samples, off-screen
buffers, file buffers, and networking buffers. The 65C816 must not depend on
PSRAM as zero-wait executable main memory in revision A.

## Mainboard Bus-Safety Contract

The CPU/system data transceiver and SRAM strobes must be active only during the
PHI2-high data phase. Implement and verify:

```text
U3_OE_N  = PHI2_N
MEM_OE_N = NOT(RWB AND PHI2)
MEM_WE_N = NOT((NOT RWB) AND PHI2)
```

This prevents SRAM and the always-enabled U3 path from contending with the
W65C816 bank address on D0-D7 during PHI2 low, and prevents write strobes while
addresses or data are changing. Prefer the existing unused NAND gates if the
pin-level review confirms they implement these equations cleanly.

Before accepting 8 MHz, document the worst-case path through the bank-address
latch, address decode, memory, U3, and CPU setup/hold requirements. Verify the
result on a logic analyser after the schematic calculation passes.

## Data Bus Contract

The socketed RP data transceiver `U16` is present, with local decoupling,
fail-safe `/OE` pull-up, control logic, and test points.

Recommended orientation:

```text
RP_D0-RP_D7  <--- U16 --->  SYS_D0-SYS_D7
    A side                    B side
```

Control rules:

- `U16 DIR = RWB`
- `RWB=1`: `RP_D` drives `SYS_D` for a CPU read.
- `RWB=0`: `SYS_D` drives `RP_D` for a CPU write.
- U16 must never drive the system bus outside a selected, valid RP2350 cycle.
- Add a pull-up to `U16_OE_N` so the transceiver defaults to disabled during
  reset and when the daughterboard is missing or unconfigured.

Define:

```text
BUS_VALID = VDA OR VPA
U16_OE_N = RP_CS_N OR PHI2_N OR NOT(BUS_VALID)
```

The RP2350 PIO bus engine must change its local data-pin direction in step with
the bus cycle. It must only drive `RP_D0-RP_D7` during a selected CPU read.

`BUS_VALID` currently protects U16 `/OE`, but firmware-visible side effects
must also require a selected cycle with valid `VDA` or `VPA`. Define and test
the exact PIO/state-machine rule; do not rely only on address decode.

Use the selected 74HC245 initially. A complete worst-case timing budget is
required before accepting operation at 8 MHz; faster clocks remain a later
goal and may justify a pin-compatible faster logic family after review.

### Wait-state policy

- Permit the RP2350 to pull `RDY` low for selected accesses.
- Drive `RDY` in an open-drain/open-collector manner; never drive it high
  against another source.
- The PIO engine should detect `RP_CS_N` during PHI2 low, request a wait state
  when necessary, prepare or capture the byte, then release `RDY` safely.
- Start with a conservative fixed wait state for every RP2350 transaction.
- Remove wait states only after logic-analyser timing measurements pass at the
  target CPU clock.

## Interrupt Architecture

Combine all normal daughterboard interrupt sources inside the RP2350. Export
one active-low, open-drain interrupt signal to the mainboard:

```text
FRAME pending ---+
AUDIO pending ---+--> IRQ_PENDING & IRQ_ENABLE --> RP_IRQ_N --> IRQ_N
I/O pending -----+
```

Required interrupt-controller registers:

- `IRQ_PENDING`: latched source bits
- `IRQ_ENABLE`: source masks
- `IRQ_ACK`: write-one-to-clear acknowledgements
- `IRQ_VECTOR`: highest-priority pending source, optional but recommended
- subsystem status registers that explain the video, audio, storage, input,
  network, and DMA condition

Initial pending-source assignments should include:

- vertical blank
- raster compare
- audio FIFO low
- audio channel/sample completion
- keyboard, mouse, or controller event
- storage completion/error
- ESP/network event
- DMA completion/error

`RP_IRQ_N` remains asserted while any enabled pending condition exists. A
level-triggered condition such as audio FIFO low must reassert or remain
pending until its underlying condition is resolved.

### Physical interrupt signals

- Use one RP2350 GPIO as open-drain `RP_IRQ_N` connected to the existing
  pulled-up `IRQ_N` system net.
- Do not independently connect `FRAME_IRQ`, `AUDIO_IRQ`, and `IO_IRQ` to the
  CPU. These names become internal RP2350 interrupt groups.
- Repurpose their existing connector pins as reserved/debug GPIOs, or mark them
  not connected in revision A.
- W5500 interrupt terminates at the ESP32-C3. Network events reach the RP2354B
  through the RP/ESP protocol and set network pending bits rather than driving
  the W65C816 directly.

### NMI policy

- Keep `NMI_N` for the physical NMI/RESTORE-style button and debugger break.
- Do not route frame, raster, audio, storage, input, or network events to NMI.
- Any future programmable NMI source must use a deliberate one-shot pulse and
  must default to disabled, because W65C816 NMI is falling-edge-sensitive.

## Networking Architecture

- The ESP32-C3 is the networking coprocessor and is the sole W5500 SPI master.
- The RP2354B communicates with the ESP32-C3 over a packetised, full-duplex
  UART link with hardware flow control.
- Move the link to the contiguous RP UART1 pin group:

| Signal | RP2354B | ESP32-C3 |
|---|---:|---:|
| RP TX / ESP RX | GPIO36 | GPIO20 |
| RP RX / ESP TX | GPIO37 | GPIO21 |
| ESP RTS_N / RP CTS_N | GPIO38 | GPIO0 |
| RP RTS_N / ESP CTS_N | GPIO39 | GPIO1 |

- Move `ESP_EN` from RP GPIO36 to GPIO10.
- Move `ESP_BOOT` from RP GPIO37 to GPIO11 and add an explicit safe pull-up at
  ESP GPIO9.
- Preserve ESP GPIO18/19 for USB Serial/JTAG and avoid GPIO2/8/9 for runtime
  handshaking because they are strapping pins.
- Reserve the newly freed RP GPIO44-GPIO46 for future expansion or debug.
- Frame every message with type, length, sequence number and CRC. Use bounded
  queues, timeouts, retry/error counters and protocol-version discovery.
- A separate ESP event GPIO is optional. UART receive interrupts plus framed
  event messages are sufficient for revision A.
- Add the W5500 25 MHz clock network and verify reset, interrupt and chip-select
  default levels. Connect or deliberately mark unused LED outputs.

## Review Findings Still To Resolve

- Mainboard U3 and `MEM_OE_N`/`MEM_WE_N` are not safely qualified by PHI2.
- U18 has unplaced unused inverter units; place them and tie every unused CMOS
  input to a defined level.
- Prove U16 is disabled during reset and non-RP cycles, and specify how
  `VDA/VPA` suppress side effects inside the RP bus engine.
- Daughterboard R15/R16 currently make `RDY` and `IRQ` assert while the RP is
  reset. Bias the U5 inputs high so both open-drain outputs default released.
- Complete the RP2354B/QMI-PSRAM power, pull-up, footprint and ERC details.
- The SD connector is wired for SPI, not the required 4-bit SDIO interface;
  DAT1/DAT2, pull-ups, damping and card detect remain unfinished.
- The USB hub's ganged power-enable and overcurrent paths are disconnected;
  downstream USB data pairs also need ESD protection.
- The W5500 lacks its 25 MHz clock and needs reset/interrupt/CS bias review.
- Decide and document DVI-only versus HDMI HPD/DDC/CEC support for revision A.
- Assign all remaining daughterboard footprints and define the RJ45 shield to
  logic-ground strategy.
- Reconcile stale component identities and connector descriptions in
  `board_design.txt`, `README.md`, and `AGENTS.md`.

## Implementation Sequence

Make and verify one subsystem at a time.

### Stage 0: Freeze and reconcile the architecture

- [x] Keep the 98-pin ISA-form-factor connector.
- [x] Document that its signalling is project-specific and not ISA-compatible.
- [x] Freeze ESP32-C3 as networking coprocessor and sole W5500 owner.
- [x] Freeze RP2354B with integrated flash and QMI PSRAM as the daughterboard
      core.
- [ ] Correct the conflicting connector, component and architecture text in
      `AGENTS.md`, `board_design.txt`, and `README.md`.

### Stage 1: Make the mainboard bus electrically safe

- [ ] Qualify U3 `/OE` with PHI2.
- [ ] Generate PHI2-qualified `MEM_OE_N` and `MEM_WE_N`.
- [ ] Verify the bank-address latch and complete the worst-case 8 MHz timing
      budget.
- [ ] Place and terminate unused U18 gates.
- [ ] Run ERC and reduce warnings to documented intentional exceptions.

### Stage 2: Finish and prove the RP bus interface

- [x] Place U16, local 100 nF decoupling, fail-safe `/OE` pull-up and test
      points.
- [x] Implement `BUS_VALID = VDA OR VPA` in U16 enable logic.
- [x] Verify the exact decode: `$F00000-$F1FFFF`, with the 1 KiB interface
      mirrored 128 times.
- [ ] Define PIO side-effect qualification using `RP_CS_N`, PHI2, RWB and
      `VDA/VPA`.
- [ ] Confirm U16 high-impedance behavior during reset and non-RP cycles.
- [ ] Verify reset and selected-cycle timing on a logic analyser.

### Stage 3: Make daughterboard control outputs fail-safe

- [ ] Bias U5 `RP_RDY_DRV` and `RP_IRQ_DRV` inputs high during reset.
- [ ] Verify that the mainboard boots normally with the daughterboard absent,
      held in reset, or running no firmware.
- [ ] Keep `NMI_N` isolated from ordinary subsystem events.
- [ ] Add power-entry flags and power-rail test points, then run ERC.

### Stage 4: Complete the RP2354B core and bus firmware

- [x] Place the RP2354B core and QMI PSRAM.
- [x] Assign contiguous PIO-friendly ranges to `A0-A9`, `RP_D0-RP_D7`, and bus
      controls.
- [ ] Verify QMI CS1 pull-up, power network, footprints and layout constraints.
- [ ] Implement PIO bus sampling, data direction and conservative RDY handling.
- [ ] Implement the register page, FIFO pages and interrupt controller.
- [ ] Test reset behavior before enabling any RP bus output.

### Stage 5: Upgrade the RP/ESP networking link

- [ ] Move RP UART to GPIO36-GPIO39 as TX/RX/CTS/RTS.
- [ ] Move `ESP_EN` and `ESP_BOOT` control to RP GPIO10/GPIO11.
- [ ] Connect ESP GPIO0/GPIO1 as RTS/CTS and preserve GPIO18/GPIO19 USB/JTAG.
- [ ] Add the ESP GPIO9 boot pull-up and verify all strap states.
- [ ] Implement framed, CRC-protected, flow-controlled messages and recovery
      from reset or queue overflow on either processor.
- [ ] Add the W5500 25 MHz clock network and complete reset, interrupt, CS and
      LED handling.
- [ ] Exercise simultaneous Wi-Fi and Ethernet traffic without losing RP
      messages or blocking the W65C816 bus service loop.

### Stage 6: Complete storage, USB, video and remaining I/O

- [ ] Convert SD wiring to 4-bit SDIO and add pull-ups, damping and card detect.
- [ ] Connect USB hub ganged power enable and combined overcurrent feedback.
- [ ] Add downstream USB ESD protection.
- [ ] Freeze DVI-only or complete HDMI HPD/DDC/CEC support.
- [ ] Define RJ45 shield grounding and mark every deliberately unused pin.

### Stage 7: Footprints and electrical closure

- [ ] Assign every remaining daughterboard footprint.
- [ ] Check each footprint against the exact selected manufacturer part.
- [ ] Run ERC after each subsystem and finish with no unexplained violations.
- [ ] Only then synchronize the PCB and begin placement/routing.
- [ ] Run DRC after PCB changes; do not autoroute USB or HDMI.

### Stage 8: Bring-up tests

- [ ] Write and read every RP register with walking-bit patterns.
- [ ] Transfer at least 256 bytes through each FIFO window.
- [ ] Verify that accesses outside the selected range never enable U16.
- [ ] Verify mirrored addresses if partial decode is retained.
- [ ] Test simultaneous frame, audio, and I/O pending bits.
- [ ] Confirm that clearing one source leaves `IRQ_N` asserted when another is
      still pending.
- [ ] Confirm that NMI button operation is independent of RP IRQ activity.
- [ ] Capture PHI2, `RP_CS_N`, `RWB`, `RDY`, U16 `/OE`, and data with a logic
      analyser at 8 MHz.
- [ ] Repeat timing tests at each proposed faster oscillator frequency.
- [ ] Run ERC after schematic changes and DRC after PCB changes.

## Revision-A Acceptance Criteria

- The W65C816 boots and runs without the daughterboard fitted.
- U16 is disabled by default and cannot contend with RAM, ROM, or the CPU.
- Every RP access is qualified by a valid address cycle.
- Register and FIFO transfers are reliable at 8 MHz with the selected wait-state
  policy.
- No normal RP subsystem uses `NMI_N`.
- Multiple simultaneous interrupt sources remain latched and discoverable.
- ESP/network activity is reported through the RP interrupt controller.
- PSRAM transfers work through indirect registers or DMA commands without
  exposing the full CPU address bus to RP GPIO.
- ERC passes after each schematic stage, and DRC passes after related PCB work.

## Reference Documents

- WDC W65C816S datasheet: IRQ, NMI, RDY, VDA/VPA, and bus timing
- Raspberry Pi RP2350 datasheet and hardware design guide: PIO, GPIO, QMI/PSRAM,
  HSTX, power, reset, and layout requirements
- `board_design.txt`: overall architecture and selected component preferences
- `AGENTS.md`: project voltage, packaging, test-point, and KiCad workflow rules
