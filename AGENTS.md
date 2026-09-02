# W65C816 Computer KiCad Rules

## Architecture contract

- This is a W65C816 computer. Application and operating-system code execute on the physical W65C816S CPU.
- The RP2354B is a video, audio, storage, USB, DMA, and I/O chipset; it is not the primary CPU or a Revision-A transparent MMU.
- The ESP32-C3 is the networking coprocessor and sole controller of the W5500 Ethernet device.
- The 98-pin ISA-form-factor connector is mechanical only. Its signalling is project-specific and is **not ISA-standard**.
- The computer must boot and run safely without the daughterboard, with the RP held in reset, or before RP firmware has started.

## Revision-A electrical contract

- Mainboard CPU, SRAM, Flash, and timing-critical glue run at **5 V**.
- Use the W65C816S6PG-14 in DIP-40, preferably in a turned-pin socket.
- Use 5 V 74AHC glue logic in DIP packages where practical. Do not substitute a selected part without explaining why.
- Fit four AS6C4008-55PIN DIP-32 SRAMs for 2 MiB. Provide four additional optional AS6C4008 footprints for 4 MiB maximum.
- Place all SRAM close to the CPU and each other. Provide optional 74AHC244 address-buffer footprints and zero-ohm bypass options; fit buffers only after a loading/timing calculation.
- Use 5 V SST39SF040-55-4I-NHE Flash in a PLCC-32 socket. Hardware Flash write-enable must default disabled.
- Use U2 as a 74AHC573 transparent bank latch and generate PHI2_N with one 74AHC04 gate. The bank-latch timing proof is mandatory.
- Keep U3 as a 74AHC245 memory-side transceiver until timing and loading analysis proves otherwise. Qualify it by PHI2 and memory selection.
- Qualify SRAM and Flash /OE and /WE by selected target, RWB, and PHI2. No write pulse may occur while address or data changes.
- Include an independent hardware wait-state generator: SRAM target 0 waits, Flash 1 wait by default, RP/peripheral waits as required. All RDY sources are open-drain/open-collector; none may drive RDY high.
- Every IC requires local 100 nF decoupling.
- Add accessible test points for PHI2, PHI2_N, reset, IRQ, NMI, RDY, RWB, VDA, VPA, chip selects, memory strobes, U3/U16 enables, selected data/bank lines, 5 V, and 3.3 V.

## 3.3 V RP2354B boundary

- The daughterboard derives 3.3 V from the same 5 V input. It must not backfeed either rail through GPIO, translators, USB, SD, or protection structures.
- Use SN74LXC8T245 for U16: VCCA/RP side at 3.3 V, VCCB/system side at 5 V. /OE must have an external pull-up and default disabled during reset, absent/unpowered daughterboard, PHI2 low, invalid cycles, and non-RP accesses.
- U16 direction and enable controls must be valid in the 3.3 V domain. The RP drives its local data pins only during a selected CPU read.
- 5 V should be translated to 3.3 V by design. Direct 5 V inputs are permitted only on individually verified RP2354B FT GPIO, only as inputs, and only with the correct power state.
- **IOVDD is operating at 3.3 V !!!!!** Direct 5 V must never reach an unpowered/disabled 3.3 V domain.
- **Non-FT pins must not receive 5 V. FT pins should receive 3.3 V by design; 5 V tolerance is only a fault-resilience constraint. !!!**
- Never use analog-capable or otherwise non-FT pins for direct 5 V signals. Document each pin assignment against the RP2354B pin table.
- RDY and RP_IRQ_N use daughterboard open-drain/open-collector stages with 5 V mainboard pull-ups. Both default released during reset, brownout, absence, or unstarted firmware.

## Bus and firmware rules

- Revision A presents the RP as registers and FIFOs at `$F00000-$F003FF`, mirrored within the intentionally partial `$F00000-$F1FFFF` decode.
- The RP interface uses A0-A9, RP_CS_N, PHI2, RWB, VDA, VPA, RESET_N, RDY, and D0-D7. Upper address lines may remain reserved on the connector but are not direct Revision-A RP inputs.
- PIO, not ordinary firmware interrupt handling, performs the bus-facing cycle handshake and must produce exactly one side effect per stretched cycle.
- Long storage, networking, DMA, blitter, and media operations are asynchronous. Never hold RDY indefinitely for a slow operation. Empty FIFO reads must have defined non-blocking or bounded-timeout behaviour.
- Ordinary daughterboard events combine into one active-low RP_IRQ_N with pending, enable, acknowledge, and vector registers. NMI is reserved for physical/debug/emergency use, never ordinary device events.
- RP PSRAM is not executable CPU memory in Revision A; use indirect registers, FIFOs, DMA, and block commands.

## KiCad workflow

- Explain proposed electrical changes before applying them; work on one subsystem at a time.
- Preserve reference designators.
- Never directly edit KiCad source files. Use Konnect MCP tools for all `.kicad_sch`, `.kicad_pcb`, library-table, symbol, and footprint changes.
- Run ERC after every schematic subsystem change and DRC after related PCB changes.
- Do not autoroute USB or video differential pairs. Route them manually.
- Do not generate manufacturing files unless explicitly requested.
- Keep all files compatible with KiCad 10.
