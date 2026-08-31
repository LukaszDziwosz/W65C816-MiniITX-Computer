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