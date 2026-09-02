# W65C816 Modern Computer

A two-board, retro-inspired computer built around a physical WDC W65C816S CPU. The CPU is the computer: it runs the boot firmware, operating system, applications, game logic, and interrupt handlers. An RP2354B daughterboard acts as a memory-mapped multimedia and I/O chipset.

Revision A deliberately combines socketed, through-hole 65xx-style hardware with modern coprocessor-based video, audio, storage, USB, and networking.

## Revision-A architecture

| Area | Decision |
| --- | --- |
| Main processor | W65C816S6PG-14, DIP-40, 5 V |
| Main clock | 8 MHz supported target; 1 MHz and 4 MHz bring-up |
| Main RAM | Four fitted AS6C4008-55PIN DIP-32 SRAMs (2 MiB) |
| RAM expansion | Four optional AS6C4008 footprints, for 4 MiB maximum |
| Boot Flash | SST39SF040-55-4I-NHE in a PLCC-32 socket |
| Mainboard glue | 5 V 74AHC, preferably socketed DIP parts |
| Daughterboard | RP2354B at 3.3 V with QMI PSRAM |
| Networking | ESP32-C3, sole controller of the W5500 Ethernet device |
| Mainboard/daughterboard connector | 98-pin ISA-form-factor, project-specific signalling |

The ISA-style connector is a mechanical choice only. It does **not** implement the ISA electrical or protocol standard.

## System model

```text
5 V system input
├── W65C816 mainboard: CPU, SRAM, Flash, AHC glue, wait-state logic
└── 3.3 V regulator
    └── RP2354B daughterboard: PSRAM, video, audio, USB, SD, ESP32-C3

W65C816 ── memory-mapped registers/FIFOs ── RP2354B ── UART ── ESP32-C3 ── W5500
```

The RP2354B is a chipset, not a transparent MMU or replacement CPU. Its PSRAM is used for media buffers, assets, DMA, blitter operations, and staging. Revision A does not execute W65C816 code from RP PSRAM.

## CPU/RP interface

The RP is selected over `$F00000-$F1FFFF`; its 1 KiB interface is deliberately mirrored because only A0-A9 are interpreted.

| Range | Function |
| --- | --- |
| `$F00000-$F000FF` | Control and status registers |
| `$F00100-$F001FF` | Command FIFO write window |
| `$F00200-$F002FF` | Response FIFO read window |
| `$F00300-$F003FF` | Input/event FIFO read window |

The first wait state for an RP access is generated in deterministic mainboard hardware. RP PIO handles the bus-cycle handshake; RP firmware handles commands and asynchronous work. Slow operations such as SD, network, DMA, and blits must complete asynchronously and report completion through the RP interrupt controller.

## Safety and timing principles

- SRAM is intended to be zero-wait at 8 MHz, with at least 10 ns calculated worst-case margin.
- Flash starts with one hardware wait state. Zero-wait Flash is optional only after calculation and logic-analyser verification.
- U2 is a 74AHC573 transparent bank latch, controlled by a single 74AHC04 inversion of PHI2.
- U3 remains a 74AHC245 memory-side data transceiver unless timing and loading analysis proves a better arrangement.
- U16 is an SN74LXC8T245 dual-supply transceiver: 3.3 V on the RP side and 5 V on the system side.
- RDY and IRQ are 5 V pulled-up, open-drain signals; the daughterboard never drives either high.
- The mainboard must boot without the daughterboard, with the RP held in reset, and with no RP firmware.
- Every IC requires local 100 nF decoupling.

The RP boundary is intentionally conservative. 5 V signals should be translated to 3.3 V by design. Direct 5 V use is permitted only on individually verified RP2354B FT GPIO, only while IOVDD is at 3.3 V, and never on analog-capable or other non-FT pins.

## Bring-up progression

1. Mainboard only at 1 MHz, then 4 MHz and 8 MHz.
2. Verify Flash wait-state, SRAM, bank-latch, decode, and write-strobe timing.
3. Fit a passive/reset-held daughterboard and verify no contention or phantom powering.
4. Bring up RP registers, FIFO, interrupts, PSRAM/DMA, ESP networking, then SD/USB/video/audio.
5. Consider 10 MHz, 12 MHz, and 14 MHz only after full timing captures and stress testing at 8 MHz.

## Repository layout

```text
Mainboard/                 KiCad 10 mainboard project
RP2350-Daughterboard/      KiCad 10 daughterboard project (name retained for compatibility)
Shared-Libraries/          Shared symbols and footprints
Documentation/             Project documentation
Firmware/                  CPU, RP, and ESP firmware
board_design.txt           Detailed Revision-A electrical/design contract
AGENTS.md                  Mandatory design and KiCad workflow rules
```

See [board_design.txt](board_design.txt) for the complete architecture, timing, interface, implementation, and validation contract.
