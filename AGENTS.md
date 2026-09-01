# W65C816 Computer KiCad Rules

## General

- This is a 3.3 V W65C816 computer.
- Application code runs on a physical W65C816S CPU.
- The RP2354B is a video, audio and I/O coprocessor.
- Prefer through-hole and socketable components wherever practical on the mainboard.
- The RP2354B daughterboard may use SMD components as required for the RP2350B, HDMI/DVI, USB, high-speed memory and other dense or high-speed circuitry.
- Do not replace selected parts without explaining why.
- Do not make large changes in a single operation.

## Component preferences

- W65C816S6PG-14 in DIP-40.
- AS6C4008 SRAM in DIP-32.
- Mainboard glue logic in DIP packages.
- Use turned-pin sockets for CPU, memory and important logic.
- HDMI and high-speed video circuitry remain on the RP2350 daughterboard and are not expected to be through-hole.

## Electrical rules

- Main logic voltage is 3.3 V.
- Never introduce a 5 V-only IC without explicit review.
- Every IC requires local decoupling.
- Add test points for clock, reset, IRQ, NMI, RDY, R/W, VDA, VPA and chip selects.
- Verify the W65C816 bank-address latch timing against the datasheet.

## KiCad workflow

- Explain proposed electrical changes before applying them.
- Make one subsystem at a time.
- Preserve reference designators.
- Run ERC after schematic changes.
- Run DRC after PCB changes.
- Never generate manufacturing files until specifically requested.
- Keep all project files compatible with KiCad 10.
