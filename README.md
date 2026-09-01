# The W65C816 Modern Computer Project

Short Introduction

I recently realised that I literally grew up with 6502-based computers and consoles, including the C64, C128, my beloved NES, and later the SNES. I always wanted an Apple IIGS, but of course I could never afford one. I still consider it the pinnacle of that architecture, especially with its fantastic GS/OS operating system.

Recently, while purchasing components for another project, I came across the W65C816S6PG-14. To my surprise, it is still being manufactured and sold in 2026. That got me thinking: could we build a new 6502-family computer using modern, readily available components?

The idea planted itself in my head and would not go away.

Initially, I was thinking about building the machine entirely in an old-school way. However, I quickly realised that although the CPU and RAM are still easy to source, other components, such as dedicated video, audio, storage, and peripheral controllers, are either no longer manufactured or would simply be too expensive.

Luckily, inexpensive microcontrollers such as the RP2350 can handle these responsibilities and even output an HDMI/DVI signal at retro-style resolutions. That possibility got me genuinely excited about the project.

I eventually understood that the computer needed to be divided into two parts.

The mainboard will contain the W65C816 CPU, RAM, ROM, bus logic, and other easily solderable through-hole components. The daughterboard will contain the more complex surface-mounted components and will need to be professionally manufactured. However, it will use inexpensive, modern, and widely available parts.

To avoid designing a completely custom enclosure, I chose the Mini-ITX form factor for the mainboard. In the position where a PCI Express graphics card connector would normally be located, the board will instead use a 16-bit ISA-style slot. This is only the physical connector; the computer will not use the actual ISA bus standard.

A 90-degree ISA adapter will connect the daughterboard to this slot. The RP2354B based daughterboard will sit horizontally above the mainboard and will be secured using standoffs in the same area where a CPU cooler would normally be mounted on a Mini-ITX motherboard.

Solid-62 case is initially chosen for the project as it has 2 x USB A in the front for gamepads, dc power cutout and on/off locking switch cu out.

The daughterboard is planned to include:

an HDMI output,

an Ethernet port,

an external Wi-Fi antenna connector,

two rear USB-A ports,

an SD card slot,

and two internal USB motherboard headers for connecting two additional front USB ports, primarily intended for gamepads.

With this arrangement, the only custom enclosure component required should be the rear I/O plate.

# Technical details

# W65C816 / RP2354B Computer

A modern 65C816-based computer built around a real WDC W65C816 processor, with an RP2354B acting as a memory-mapped multimedia and I/O chipset.

The W65C816 remains responsible for application code, operating-system services, and game logic. The RP2354B provides video, audio, storage, USB input, networking coordination, DMA, and hardware-accelerated media operations.

The architecture is intentionally closer to systems such as the Apple IIgs and the planned Commodore 65 than to an ARM computer using a 65C816 only as a peripheral.

## Revision-A Architecture

### Main processor

- WDC W65C816S
- 24-bit address space
- 8-bit external data bus
- Initial target clock: 8 MHz
- Higher clock rates may be considered after complete timing verification
- Native execution from mainboard SRAM and ROM
- Direct ownership of the main system bus

### Multimedia and I/O chipset

- Raspberry Pi RP2354B
- Integrated flash
- External QMI PSRAM
- Memory-mapped interface to the W65C816
- PIO-based bus interface
- Video generation through HSTX
- Audio generation and sample playback
- SD-card storage
- USB host support
- Input-device processing
- DMA, blitter, copy, and fill operations
- Communication bridge to the ESP32-C3 networking processor

The RP2354B is not a transparent system MMU in Revision A. It exposes registers, FIFOs, DMA commands, and indirect PSRAM access.

### Networking processor

- ESP32-C3
- Dedicated networking coprocessor
- Sole SPI master for the W5500 Ethernet controller
- Wi-Fi support
- Ethernet support through W5500
- Packetised UART connection to the RP2354B
- Hardware RTS/CTS flow control
- USB Serial/JTAG retained for development and diagnostics

The W65C816 does not communicate directly with the ESP32-C3 or W5500. Networking requests and events pass through the RP2354B chipset interface.

## System Philosophy

The design follows these principles:

- Application code executes on the W65C816.
- Main SRAM remains directly accessible to the W65C816.
- RP2354B PSRAM is accelerator and media memory rather than primary CPU memory.
- The RP2354B behaves like a custom chipset, not the main computer.
- Time-sensitive bus control is handled by deterministic PIO and external logic.
- The system must boot and operate safely without the daughterboard installed.
- Revision A prioritises electrical safety and predictable timing over maximum performance.

## Mainboard and Daughterboard Connection

The mainboard and RP2354B daughterboard use a 98-pin ISA-style edge connector with a matching right-angle adapter.

The connector is used only for its mechanical properties:

- robust insertion
- keyed orientation
- wide pin availability
- low-cost availability
- mechanically secure daughterboard mounting

The electrical interface is project-specific and is not ISA-compatible.

The connector pinout remains provisional until PCB layout is complete. Every pinout change must be reviewed individually because electrical changes may require signal reassignment, logic changes, or updated timing analysis.

## RP2354B Bus Interface

The RP2354B appears as a memory-mapped peripheral on the W65C816 bus.

### Address and control signals

The Revision-A RP interface uses:

- `A0-A9`
- `RP_CS_N`
- `PHI2`
- `RWB`
- `VDA`
- `VPA`
- `RESET_N`
- `RDY`
- `D0-D7`

Address lines `A10-A23` are not connected directly to RP2354B GPIO in Revision A.

The mainboard connector may retain these upper address signals as reserved raw-bus signals for:

- future expansion
- debugging
- a CPLD-based bus controller
- a later transparent memory-window implementation

Any future transparent MMU or executable-PSRAM design requires deterministic external logic that observes the complete address bus and participates directly in RAM and ROM chip-select generation.

## Revision-A Memory Map

| Address range | Function |
|---|---|
| `$F00000-$F000FF` | RP2354B control and status registers |
| `$F00100-$F001FF` | Command FIFO write window |
| `$F00200-$F002FF` | Response FIFO read window |
| `$F00300-$F003FF` | Input and event FIFO read window |

Address bits `A8-A9` select one of the four 256-byte interface pages.

Within FIFO pages, every address may access the same FIFO data port. This allows the W65C816 to use efficient sequential transfer instructions without repeatedly accessing a single fixed address.

### Partial address decoding

The current hardware decode selects:

```text
$F00000-$F1FFFF
```

Only the lower 10 address bits are interpreted by the RP2354B interface. Because `A10-A16` are not decoded, the 1 KiB interface is mirrored 128 times throughout the selected 128 KiB region.

Examples of equivalent mirrored register addresses include:

```text
$F00000
$F00400
$F00800
...
$F1FC00
```

Tighter address decoding is deferred to a later revision.

A proposed direct shared-memory window at `$F10000-$F1FFFF` is not implemented in Revision A. Such a window would require at least `A0-A15` and a more complex deterministic bus bridge.

## RP2354B Register Interface

The register page occupies `$F00000-$F000FF`.

The final register layout remains to be frozen, but the interface is expected to include the following groups:

- chipset identification
- hardware revision
- firmware version
- feature discovery
- interrupt control
- FIFO status
- PSRAM indirect access
- DMA control
- video control
- audio control
- storage status
- USB and input status
- networking status
- diagnostic counters

All reserved registers must return a defined value and ignore writes unless otherwise documented.

Firmware-visible side effects must occur only during a selected and valid W65C816 bus cycle.

## FIFO Interface

Three memory-mapped FIFO pages provide high-throughput communication.

### Command FIFO

Address range:

```text
$F00100-$F001FF
```

Used by the W65C816 to submit commands and bulk data to the RP2354B.

Typical commands include:

- DMA transfer
- PSRAM upload
- PSRAM download
- memory copy
- memory fill
- blitter operation
- file operation
- video configuration
- audio configuration
- network request
- device-control request

### Response FIFO

Address range:

```text
$F00200-$F002FF
```

Used by the RP2354B to return:

- command results
- requested data
- storage responses
- network responses
- error information
- device descriptors
- diagnostic data

### Input and Event FIFO

Address range:

```text
$F00300-$F003FF
```

Used for asynchronous events such as:

- keyboard input
- mouse movement
- mouse buttons
- game-controller input
- USB device changes
- storage completion
- network events
- system notifications

Each FIFO page may treat all 256 addresses as aliases of the same data port.

FIFO status registers must expose at least:

- empty
- full
- current fill level
- overflow
- underflow
- error state

## PSRAM

External QMI PSRAM is attached directly to the RP2354B.

It is intended for:

- framebuffers
- tile maps
- tile graphics
- sprite data
- fonts
- audio samples
- audio mixing buffers
- off-screen rendering buffers
- file buffers
- decompression buffers
- networking buffers
- DMA staging
- blitter source and destination memory

The W65C816 does not execute directly from PSRAM in Revision A.

### Indirect PSRAM access

The CPU accesses PSRAM through chipset registers such as:

- `MEM_ADDR0`
- `MEM_ADDR1`
- `MEM_ADDR2`
- `MEM_ADDR3`
- `MEM_DATA`
- `MEM_CONTROL`
- `MEM_STATUS`

The interface should support optional address auto-increment for sequential reads and writes.

Large transfers should use command-FIFO DMA operations rather than repeated single-byte register accesses.

Expected memory commands include:

- CPU RAM to PSRAM
- PSRAM to CPU RAM
- PSRAM to PSRAM
- fill region
- copy region
- upload graphics
- upload audio samples
- readback
- blitter operation

## Mainboard Bus Safety

The W65C816 multiplexes the bank address onto `D0-D7` while `PHI2` is low. System data transceivers and memory write strobes must therefore remain inactive outside the valid `PHI2`-high data phase.

Required equations:

```text
U3_OE_N  = PHI2_N
MEM_OE_N = NOT(RWB AND PHI2)
MEM_WE_N = NOT((NOT RWB) AND PHI2)
```

These rules ensure that:

- SRAM does not drive the bus during the bank-address phase.
- The system data transceiver does not contend with the CPU.
- SRAM write strobes occur only during the valid data phase.
- Address and data transitions cannot accidentally produce memory writes.

Unused logic gates may be used to implement these equations only after a pin-level logic review confirms correct polarity, propagation delay, and power-up behaviour.

Every unused CMOS input must be tied to a defined logic level.

## RP Data Transceiver

The RP daughterboard interface uses a socketed `74HC245`-class bidirectional data transceiver, designated `U16`.

Recommended orientation:

```text
RP_D0-RP_D7  <--- U16 --->  SYS_D0-SYS_D7
    A side                    B side
```

### Direction

```text
U16 DIR = RWB
```

Behaviour:

```text
RWB = 1: RP_D drives SYS_D during a CPU read
RWB = 0: SYS_D drives RP_D during a CPU write
```

### Output enable

Define:

```text
BUS_VALID = VDA OR VPA
U16_OE_N  = RP_CS_N OR PHI2_N OR NOT(BUS_VALID)
```

The transceiver may only be enabled when:

- the RP interface is selected
- `PHI2` is high
- either `VDA` or `VPA` indicates a valid CPU cycle

A pull-up on `U16_OE_N` keeps the transceiver disabled when:

- the daughterboard is absent
- the daughterboard is held in reset
- RP firmware has not started
- control logic is unconfigured
- the interface is not selected

The RP2354B must drive its local data pins only during a selected CPU read. During writes, idle cycles, reset, and non-RP transactions, RP data GPIO must remain inputs or otherwise high impedance.

## Bus-Cycle Qualification

Hardware transceiver enable is not sufficient by itself.

The RP2354B PIO bus engine must qualify every firmware-visible access using:

- `RP_CS_N`
- `PHI2`
- `RWB`
- `VDA`
- `VPA`

A register write, FIFO push, FIFO pop, acknowledgement, or other side effect must not occur during:

- invalid bus cycles
- unselected cycles
- bank-address phases
- reset
- repeated wait-state sampling
- mirrored bus activity not meeting the complete access rule

The exact PIO state-machine behaviour must be documented and tested.

## Wait-State Policy

The RP2354B may extend selected W65C816 cycles by pulling `RDY` low.

`RDY` must be driven using open-drain or open-collector behaviour. The RP2354B must never actively drive `RDY` high.

Initial Revision-A policy:

- apply one conservative fixed wait state to every RP access
- detect `RP_CS_N` during `PHI2` low
- assert `RDY` before the CPU completes the selected cycle
- prepare read data or capture write data
- release `RDY` only when the transfer can complete safely

Wait states may be reduced or removed only after logic-analyser measurements confirm correct setup, hold, and propagation timing at the target CPU frequency.

## Interrupt Controller

All ordinary daughterboard interrupt sources are combined inside the RP2354B.

The daughterboard exports one active-low open-drain signal:

```text
RP_IRQ_N
```

This connects to the pulled-up system `IRQ_N` net.

Conceptually:

```text
FRAME pending ----+
AUDIO pending ----+
INPUT pending ----+
STORAGE pending --+--> pending AND enabled --> RP_IRQ_N
NETWORK pending --+
DMA pending ------+
```

### Required interrupt registers

- `IRQ_PENDING`
- `IRQ_ENABLE`
- `IRQ_ACK`
- `IRQ_VECTOR`

`IRQ_ACK` uses write-one-to-clear semantics.

`IRQ_VECTOR` should return the highest-priority enabled pending source.

### Initial interrupt sources

The initial controller should support:

- vertical blank
- raster compare
- audio FIFO low
- audio channel completion
- audio sample completion
- keyboard event
- mouse event
- controller event
- storage completion
- storage error
- ESP/network event
- DMA completion
- DMA error

`RP_IRQ_N` remains asserted while any enabled interrupt source remains pending.

Level-sensitive conditions, such as audio FIFO low, may remain asserted or reassert after acknowledgement until the underlying condition has been corrected.

Subsystem status registers must provide sufficient detail to determine the cause of each interrupt.

## NMI

`NMI_N` is reserved for:

- the physical NMI or RESTORE-style button
- debugger break
- explicitly designed future emergency functions

Normal daughterboard subsystems must not use NMI.

The following must use IRQ instead:

- vertical blank
- raster interrupts
- audio events
- storage events
- keyboard and mouse events
- controller events
- networking events
- DMA completion

Any future programmable NMI source must:

- be disabled by default
- generate a controlled one-shot pulse
- avoid remaining permanently low
- account for the W65C816 falling-edge-sensitive NMI input

## RP2354B to ESP32-C3 Link

The RP2354B and ESP32-C3 communicate using a full-duplex UART with hardware flow control.

### Pin assignment

| Function | RP2354B | ESP32-C3 |
|---|---:|---:|
| RP TX / ESP RX | GPIO36 | GPIO20 |
| RP RX / ESP TX | GPIO37 | GPIO21 |
| ESP RTS_N / RP CTS_N | GPIO38 | GPIO0 |
| RP RTS_N / ESP CTS_N | GPIO39 | GPIO1 |
| ESP enable | GPIO10 | `EN` |
| ESP boot control | GPIO11 | GPIO9 |

ESP32-C3 GPIO9 requires an explicit safe pull-up.

ESP32-C3 GPIO18 and GPIO19 remain reserved for USB Serial/JTAG.

Runtime handshaking must avoid ESP32-C3 strapping pins GPIO2, GPIO8, and GPIO9, apart from the deliberately controlled boot function on GPIO9.

RP2354B GPIO44-GPIO46 remain reserved for future expansion or debugging.

## Networking Protocol

Communication between the RP2354B and ESP32-C3 uses framed packets.

Every packet must include:

- message type
- payload length
- sequence number
- payload
- CRC

The protocol should provide:

- bounded transmit queues
- bounded receive queues
- hardware flow control
- request timeouts
- retry handling
- CRC-error counters
- framing-error counters
- queue-overflow counters
- protocol-version discovery
- capability discovery
- recovery after either processor resets
- recovery from incomplete packets
- recovery from queue overflow

A separate ESP event GPIO is optional. UART receive interrupts and framed asynchronous event messages are sufficient for Revision A.

## W5500 Ethernet

The ESP32-C3 is the only W5500 SPI master.

The W5500 interface must include:

- SPI clock
- MOSI
- MISO
- chip select
- reset
- interrupt
- 25 MHz clock network
- verified power-up biasing
- documented LED handling

The W5500 interrupt terminates at the ESP32-C3. It does not connect directly to the W65C816.

Network events are translated into RP protocol messages, which set the appropriate RP interrupt-controller pending bits.

## Video

The RP2354B generates digital video using HSTX.

Revision-A video support is expected to include:

- indexed framebuffers
- tile-based graphics
- hardware sprites
- text modes
- programmable palettes
- raster position tracking
- vertical-blank interrupt
- raster-compare interrupt
- blitter operations
- off-screen buffers in PSRAM

The initial output standard must be explicitly frozen as either:

- DVI-compatible output only

or:

- HDMI-compatible connector support including HPD, DDC, and CEC handling

No claim of full HDMI compliance should be made if only DVI-compatible video signalling is implemented.

## Audio

The RP2354B provides the audio subsystem.

Expected capabilities include:

- multiple software-controlled channels
- sample playback from PSRAM
- audio FIFO
- programmable playback rates
- volume control
- stereo mixing
- channel-completion interrupts
- sample-completion interrupts
- FIFO-low interrupt
- DMA-assisted sample transfer

Audio playback must continue independently of normal W65C816 instruction timing once configured.

## SD Card

The SD-card interface is intended to use 4-bit SDIO rather than SPI.

Required signals include:

- `CLK`
- `CMD`
- `DAT0`
- `DAT1`
- `DAT2`
- `DAT3`
- card detect

The final design must include:

- required pull-ups
- clock-series damping as appropriate
- card-detect biasing
- defined power-up states
- short and controlled routing
- a verified connector footprint

The current SPI-only wiring is incomplete and must not be treated as the final Revision-A interface.

## USB

The RP2354B provides USB host connectivity through a USB hub.

The completed design must include:

- USB hub upstream connection
- downstream USB ports
- ganged power enable
- combined overcurrent feedback
- downstream data-line ESD protection
- correct differential-pair routing
- controlled impedance where required
- verified VBUS power switching
- defined behaviour during overcurrent
- defined behaviour while the RP2354B is reset

USB and HDMI/DVI differential pairs must be manually routed and must not be autorouted.

## Fail-Safe Behaviour

The mainboard must remain operational when the daughterboard is:

- not fitted
- unpowered
- held in reset
- running no firmware
- running invalid firmware

Required default states include:

- `U16` disabled
- `RDY` released
- `IRQ_N` released
- `NMI_N` unaffected
- RP data bus high impedance
- ESP32-C3 held in a safe boot state
- W5500 chip select inactive
- W5500 reset defined
- PSRAM chip select inactive
- USB power switching in a defined state

Open-drain driver inputs for `RDY` and `IRQ_N` must be biased so that both outputs default to released while the RP2354B is reset.

## Timing Requirements

Before accepting 8 MHz operation, a complete worst-case timing budget must cover:

- W65C816 address-valid timing
- bank-address latch propagation
- address-decode propagation
- SRAM access time
- ROM access time
- U3 transceiver propagation
- U16 transceiver propagation
- RP2354B input sampling
- RP2354B read-data preparation
- `RDY` assertion timing
- CPU data setup time
- CPU data hold time
- control-signal setup and hold requirements

The calculation must use worst-case datasheet values rather than typical values.

After the schematic timing calculation passes, the result must be verified with a logic analyser.

Signals to capture include:

- `PHI2`
- `RP_CS_N`
- `RWB`
- `VDA`
- `VPA`
- `RDY`
- `U16_OE_N`
- system data bus
- RP local data bus
- memory read strobe
- memory write strobe

Faster oscillator frequencies remain experimental until the same calculation and measurement process has been repeated successfully.

## Bring-Up Sequence

### Stage 1: Mainboard-only boot

Verify that the W65C816:

- exits reset
- reads reset vectors correctly
- accesses ROM safely
- reads and writes SRAM
- operates without the daughterboard
- shows no data-bus contention

### Stage 2: Passive daughterboard

Install the daughterboard while holding the RP2354B in reset.

Verify:

- normal mainboard boot
- `U16` remains disabled
- `RDY` remains released
- `IRQ_N` remains released
- no RP data pin drives the bus

### Stage 3: Register access

Implement a minimal PIO bus engine and verify:

- register reads
- register writes
- walking-bit patterns
- reset behaviour
- mirrored address behaviour
- invalid-cycle rejection

### Stage 4: FIFO access

Verify:

- at least 256 sequential writes to the command FIFO
- at least 256 sequential reads from the response FIFO
- input FIFO operation
- overflow reporting
- underflow reporting
- FIFO-level reporting

### Stage 5: Interrupt controller

Verify:

- each source independently
- multiple simultaneous pending sources
- masking
- write-one-to-clear acknowledgement
- priority-vector reporting
- IRQ remaining asserted while another source is still pending
- independence of NMI button operation

### Stage 6: PSRAM and DMA

Verify:

- indirect PSRAM reads and writes
- auto-increment
- CPU RAM to PSRAM transfer
- PSRAM to CPU RAM transfer
- PSRAM copy
- fill operation
- DMA completion interrupt
- DMA error reporting

### Stage 7: Networking

Verify:

- RP-to-ESP packet framing
- CRC rejection
- sequence-number handling
- timeout recovery
- reset recovery
- queue-overflow recovery
- Wi-Fi operation
- Ethernet operation
- simultaneous Wi-Fi and Ethernet traffic
- network event propagation to the W65C816 IRQ controller

### Stage 8: Multimedia and I/O

Verify:

- video output
- vertical-blank interrupt
- raster interrupt
- audio playback
- audio FIFO interrupt
- SDIO storage
- USB keyboard
- USB mouse
- USB game controller
- hot-plug and error behaviour

## Revision-A Acceptance Criteria

Revision A is accepted when all of the following are true:

- The W65C816 boots and runs without the daughterboard installed.
- The main system bus remains free from contention.
- U16 defaults to disabled.
- U16 cannot drive the system bus outside a selected valid RP cycle.
- Memory read and write strobes are qualified by `PHI2`.
- Every RP access requires a valid `VDA` or `VPA` cycle.
- Register and FIFO transfers operate reliably at 8 MHz.
- The conservative wait-state policy is stable and repeatable.
- `RDY` and `IRQ_N` default to released while the RP2354B is reset.
- No ordinary RP subsystem uses `NMI_N`.
- Multiple interrupt sources remain independently latched and discoverable.
- ESP and W5500 events reach the CPU through the RP interrupt controller.
- PSRAM is accessible through indirect registers and DMA commands.
- The RP2354B does not require direct access to `A10-A23`.
- The system recovers correctly after RP2354B or ESP32-C3 reset.
- All unused CMOS inputs are tied to defined levels.
- Every deliberately unused connector pin is documented.
- ERC contains no unexplained violations.
- DRC contains no unexplained violations.
- USB and video differential pairs are manually routed and reviewed.
- Logic-analyser captures confirm the calculated 8 MHz timing margins.

## Known Revision-A Limitations

- The RP interface is mirrored throughout `$F00000-$F1FFFF`.
- PSRAM is not directly executable by the W65C816.
- There is no transparent RP-controlled MMU.
- The RP2354B does not observe the complete 24-bit CPU address bus.
- Every RP transaction may initially require a wait state.
- Direct shared-memory windows are deferred.
- Higher W65C816 clock rates are not guaranteed.
- Full HDMI support is not guaranteed unless HPD, DDC, and CEC are implemented.
- The connector is mechanically ISA-style but electrically incompatible with ISA hardware.

## Future Expansion

Potential later revisions may add:

- tighter RP address decoding
- CPLD-based bus control
- full `A0-A23` address observation
- transparent shared-memory windows
- executable accelerator memory
- programmable memory mapping
- cache or prefetch support
- reduced or eliminated RP wait states
- faster data-bus transceivers
- higher W65C816 clock rates
- additional expansion GPIO
- dedicated debug interface
- advanced DMA chaining
- hardware decompression
- expanded video modes
- additional audio channels

These features are outside the Revision-A electrical contract and must not compromise the requirement that the W65C816 remains the primary computer.