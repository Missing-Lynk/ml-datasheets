# Artosyn Proxima carrier board (unofficial datasheet)

> **Reconstructed from physical inspection** of an opened BetaFPV VR04 HD goggle. Established observations and clearly labelled inferences; subject to revision as more units are inspected.

## What is on the board

The Artosyn Proxima-9311 SoC, the SPI-NAND flash, and the AR8030 RF link chip sit together on a single sub-board, soldered to the main PCB by edge pins (castellated). The main PCB carries only device-specific support: power supply, button inputs, and the LCD driver. The firmware's board model string, `model = "Artosyn, Proxima Development Board"`, is reported by the stock kernel regardless of end-product enclosure and corroborates the sub-board's standalone identity.

**Components confirmed on the sub-board (physical observation, BetaFPV VR04 HD):**

- Artosyn Proxima-9311 SoC. Register map and peripheral list: [`soc-proxima-9311.md`](soc-proxima-9311.md).
- SPI-NAND flash, Xtx XT26G0xA family (128 MiB, 2048-byte pages, 128-byte OOB). Kernel boot banner: "XT SPI NAND". Identified via the SPI-NAND core's Xtx manufacturer table (`CONFIG_SPI_XTX` on the vendor 4.9 kernel; built into `CONFIG_MTD_SPI_NAND` on mainline 6.x). Partition map arrives on the kernel cmdline (`mtdparts=spi32766.1:...` on stock; 21 MTD partitions); full table in [`../open-firmware-bsp.md`](../open-firmware-bsp.md).
- AR8030 RF link chip (AR803x family). Interface and configuration: [`ar8030-rf-link.md`](ar8030-rf-link.md).

Physical details not yet confirmed (silkscreen/part number, edge-pin mapping): see Unknowns.

## How the blocks attach to the SoC

### AR8030 over SDIO (mmc0)

The AR8030 sits on the SoC's `mmc0` SDIO controller (`0x01b0_0000`); the microSD slot is on the separate `mmc1` (`0x01c0_0000`; the card is an unpartitioned exFAT volume, `/dev/mmcblk2` on stock, `/dev/mmcblk0` on the open kernel), so `mmc0` is dedicated to the AR8030. Kernel config for this block: `CONFIG_MMC=y`, `CONFIG_MMC_DW=y`. Driver, firmware upload, and the `sdio0` net interface: [`ar8030-rf-link.md`](ar8030-rf-link.md) "How it attaches to the SoC".

The AR8030 reset line is SoC gpio23 (GPIO bank 1, line 0), active-low: released low->high and held high at bring-up, before SDIO enumeration (`kernel/modules/HW-BRINGUP.md` Phase 6).

### SPI-NAND over QSPI (ar9301-qspi)

The flash connects to the SoC's QSPI controller (`0x01e0_0000`, `artosyn,ar9301-qspi`, IRQ GIC 130), driven by the custom open driver `spi-ar9301.c` (`kernel/overlay/drivers/spi/`). Above it: the SPI-NAND layer (`CONFIG_MTD_SPI_NAND`, `CONFIG_MTD_NAND_CORE`), whose core carries the Xtx manufacturer table. The partition table is passed on the kernel cmdline (`CONFIG_MTD_CMDLINE_PARTS`) because the vendor does not encode it in the device tree. UBI (`CONFIG_MTD_UBI`, `CONFIG_MTD_UBI_BLOCK`) and squashfs (`CONFIG_SQUASHFS`, `CONFIG_SQUASHFS_XZ`) complete the storage stack.

## Shared-sub-board hypothesis

**Inference, not established fact.** The SoC, NAND, and AR8030 form a self-contained assembly with its own "Proxima Development Board" identity string, so the sub-board is plausibly a common part reused across Artosyn-based products (other BetaFPV devices, other ArtLynk vendors). The observed split (all compute and RF on the sub-board, only power/buttons/LCD on the main PCB) is consistent with, but does not confirm, cross-device reuse.

If true:

- The AR8030 reset GPIO (gpio23) is fixed by the sub-board wiring, identical across devices using it.
- The `mmc0`-to-AR8030 SDIO wiring is identical across those devices.
- The QSPI wiring and flash chip carry over, making the SPI-NAND kernel config directly portable.
- A device-agnostic kernel config covering SDIO/mmc0, QSPI/SPI-NAND, and the RF reset GPIO works on any device with the same sub-board.

This is the rationale for treating those three blocks as shared/portable in the open kernel config (`kernel/configs/artosyn.config`) rather than deriving them per-device.

## Unknowns / to confirm

- **Board silkscreen / part number.** Would confirm a named, released sub-board and enable cross-referencing with other teardowns.
- **Edge-pin count and signal mapping.** Which signals the main PCB routes to the SoC, AR8030, and NAND.
- **Photos.** Would allow cross-referencing with FCC or community teardowns of other ArtLynk devices.
- **Other devices using this sub-board.** Not characterized; do not assert a device list without evidence.

## Related

- [`soc-proxima-9311.md`](soc-proxima-9311.md), the Proxima-9311 SoC datasheet (peripheral map, memory map, interrupt map).
- [`ar8030-rf-link.md`](ar8030-rf-link.md), the AR8030 RF link chip (SDIO interface, configuration, RF parameters).
- [`../open-firmware-bsp.md`](../open-firmware-bsp.md), the full MTD partition table and flash layout.
- [`../hardware-overview.md`](../hardware-overview.md), the top-level hardware and firmware overview for the BetaFPV VR04 HD.
- `kernel/configs/artosyn.config`, the open kernel config covering SDIO, QSPI, and SPI-NAND for this platform.
