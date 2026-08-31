# W65C816/RP2350 Interface Plan

## Goal

Keep the W65C816 as the computer that runs applications and game logic. Treat
the RP2350 as a memory-mapped video, audio, storage, input, networking, and DMA
chipset. The design should feel closer to an Apple IIgs or the intended
Commodore 65 architecture than to an ARM computer with a 65C816 attached.

The ESP module remains a network offload device behind the RP2350. External
RP2350 PSRAM is primarily media, buffer, and accelerator memory.

## Agreed Revision-A Architecture

### RP2350 bus role

- The RP2350 is a memory-mapped I/O device.
- The RP2350 bus interface consumes `A0-A9`, `RP_CS_N`, `PHI2`, `RWB`,
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

The existing partial decode appears to select the complete
`$F00000-$F1FFFF` region. For revision A, the 1 KiB interface may mirror
through that region if this is confirmed and documented. Alternatively, the
mainboard decode can be tightened in a later, separate change.

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

## Data Bus Contract

Add the documented socketed RP2350 data transceiver, `U16`, because it is not
present in the current mainboard schematic.

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

Use the selected 74HC245 for the initial 8 MHz system with wait states. Before
attempting zero-wait operation at 14 MHz, review the complete timing budget and
consider a pin-compatible 74AHC245 only if the faster part is justified.

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
- ESP and Ethernet interrupt inputs terminate at the RP2350. They set network
  or I/O pending bits rather than driving the W65C816 directly.

### NMI policy

- Keep `NMI_N` for the physical NMI/RESTORE-style button and debugger break.
- Do not route frame, raster, audio, storage, input, or network events to NMI.
- Any future programmable NMI source must use a deliberate one-shot pulse and
  must default to disabled, because W65C816 NMI is falling-edge-sensitive.

## Current Schematic Gaps

- `board_design.txt` says `A0-A7`, but the four proposed 256-byte pages require
  `A0-A9`.
- J6 currently exposes the complete address and system data buses.
- The planned `U16` RP2350 data transceiver is missing.
- `FRAME_IRQ`, `AUDIO_IRQ`, and `IO_IRQ` currently end at J7 and are not
  combined with `IRQ_N`.
- The component table calls U12 a 74HC32, while the schematic uses U12 as a
  74HC139 decoder. Do not assume unused OR gates exist.
- The RP2350 daughterboard schematic is currently empty.
- The RP select and read/write strobes must be checked for `VDA/VPA`
  qualification before peripheral side effects are allowed.

## Implementation Sequence

Make and verify one subsystem at a time.

### Immediate actions:

C13, C14, C15 and R7 have no footprints. These include the new glue-logic decoupling and the fail-safe U16_OE_N pull-up, so PCB synchronization should wait until they are assigned.

Think about ESP32 C3 role
                    ┌──────────── Wi-Fi
                    │
W65C816 ⇄ RP2350B ⇄ ESP32-C3
                    │
                    └── SPI ⇄ W5500 ⇄ Ethernet

I would consider connecting the ESP32-C3 directly to the W65C816 bus interface, rather than routing all network traffic through the RP2350B:

W65C816 bus
    ├── RP2350B: video, audio, USB, SD, system I/O
    └── ESP32-C3: Wi-Fi, Ethernet and network services
                     │
                     └── W5500


### Stage 1: Document and decode the interface

- [x] Update `board_design.txt` from `A0-A7` to `A0-A9`.
- [x] Replace the shared-memory-window proposal with indirect PSRAM access.
- [ ] Document whether the 1 KiB RP interface intentionally mirrors throughout
      `$F00000-$F1FFFF`.
- [ ] Verify the exact `RP_CS_N` decode truth table.
- [ ] Add `BUS_VALID = VDA OR VPA` qualification to side-effecting accesses.
- [x] Run ERC.

### Stage 2: Add the RP data path

- [ ] Add socketed U16 and local 100 nF decoupling.
- [x] Rename the connector-side data nets to `RP_D0-RP_D7` if U16 is placed on
      the mainboard.
- [ ] Implement `DIR`, fail-safe `/OE`, and valid-cycle qualification.
- [ ] Add test points for `RP_CS_N`, `U16_OE_N`, `RWB`, `PHI2`, `RDY`, and at
      least one bit on each side of U16.
- [ ] Run ERC.
- [ ] Confirm that U16 is high-impedance during reset and non-RP cycles.

### Stage 3: Finish interrupt/control wiring

- [ ] Replace the three external interrupt-source nets with one `RP_IRQ_N`.
- [ ] Connect `RP_IRQ_N` to system `IRQ_N` through an explicitly open-drain
      interface.
- [ ] Verify the existing IRQ pull-up value and reset behavior.
- [ ] Keep `NMI_N` isolated from normal RP2350 events.
- [ ] Reassign or reserve the three freed connector pins.
- [ ] Run ERC.

### Stage 4: Build the daughterboard bus engine

- [x] Place the RP2350B/RP2354B core using the official reference design.
- [x] Assign contiguous GPIO ranges suitable for PIO to `A0-A9`, `RP_D0-RP_D7`,
      and bus controls.
- [ ] Implement PIO bus sampling and data direction.
- [ ] Implement conservative `RDY` wait-state handling.
- [ ] Implement the register page and three FIFO pages.
- [ ] Implement interrupt pending, enable, acknowledge, and vector registers.
- [ ] Test reset behavior before enabling any RP bus output.

### Stage 5: Bring-up tests

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
