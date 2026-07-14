# ArtLynk datasheets (unofficial)

Unofficial hardware reference for the Artosyn "ArtLynk" FPV platform - the Proxima-9311 application SoC and the AR8030 digital-RF link chip - plus the carrier board they sit on.

There are no public datasheets for this silicon. Everything here was reconstructed from the outside: on-device introspection of a running unit (`/proc/device-tree`, `/proc/iomem`, `/proc/interrupts`, `/proc/cpuinfo`, `dmesg`, `lsmod`), reverse engineering of the vendor firmware and its 4.9 kernel, and the bring-up of an open mainline 6.x kernel on the same hardware - which validates the peripheral, clock, and memory maps by actually driving them. These are original notes written from that evidence, not scanned or copied vendor documents.

They were compiled as part of [MissingLynk](https://github.com/Missing-Lynk), an effort to build an open firmware stack for ArtLynk-based FPV devices; the maps here are what the open kernel and userspace were written against.

## Contents

- `soc-proxima-9311.md` - the Proxima-9311 SoC: peripheral map, memory map, interrupt map.
- `ar8030-rf-link.md` - the AR8030 RF link chip: SDIO interface, configuration, RF parameters.
- `carrier-board.md` - the carrier board: flash, storage, and how the parts are wired together.

Accuracy is best-effort and device-specific (reference unit: BetaFPV VR04 HD, firmware `1.0.44.rel`). Corrections welcome.
