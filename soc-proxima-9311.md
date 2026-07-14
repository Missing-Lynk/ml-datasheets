# Artosyn Proxima-9311 (unofficial datasheet)

> **Reference, stable.** The canonical SoC datasheet; complete as of fw `1.0.44.rel`. Amend only when new silicon facts are confirmed, not for active work-in-progress.

Reference for the Artosyn Proxima-9311, the application SoC in the BetaFPV VR04 HD goggle and air unit. No official datasheet is public; this is compiled from on-device introspection (device tree, `/proc`, `lsmod`, `dmesg`) of firmware `1.0.44.rel` / kernel `4.9.38`.

## Identity

| field | value |
|---|---|
| Vendor | Artosyn |
| SoC | Proxima-9311 (`compatible = artosyn,proxima-9311`) |
| Board model string | "Artosyn, Proxima Development Board" |
| Family | Proxima / Sirius (IP blocks tagged `ar9311-*` and `ar9301-*`) |
| Application | Artosyn HD digital-FPV link endpoints (goggle and air unit) |
| Companion IC | AR8030 digital-RF baseband/link, external, SDIO-attached (loads `bb_demo_gnd_d.img`). The radio is a **separate chip**, not on-die; see [`ar8030-rf-link.md`](ar8030-rf-link.md). |
| Process / package | not characterized |

## Overview

- 2x ARM Cortex-A53 up to 800 MHz, aarch64, NEON and ARMv8 crypto extensions.
- NPU (CNN), DSP, and an MPP media block (ISP, H.264/H.265 and JPEG codecs, MIPI CSI/DSI, scaler, GE2D, GDC, EIS/LDC).
- DDR based at `0x2000_0000`.
- I/O: Gigabit Ethernet MAC, USB 2.0 OTG, 2x SD/SDIO and 1x SDHC, QSPI (SPI-NAND), 5x I2C, master and slave SPI, 6x UART, 7 GPIO ports, ADC, RTC (plus an external DS1307 on I2C0), temperature sensor, 2x PWM.
- Hardware security: TrustZone TZC-400, SPAcc crypto engine, RSA/PKA, TRNG, eFuse.

## CPU complex

- 2x ARM Cortex-A53 (`arm,cortex-a53`, implementer `0x41`, part `0xd03`, r0p4), 64-bit ARMv8-A. Features: `fp asimd aes pmull sha1 sha2 crc32`.
- Per-core L1 I-cache and D-cache, shared L2 (`arm,arch-cache`).
- GIC-400 interrupt controller (`arm,gic-400`) at `0x0020_0000`.
- ARMv8 architected timer (`arm,armv8-timer`), A53 PMU (`arm,cortex-a53-pmu`).
- DVFS via `operating-points-v2`. OPPs (shared across both cores): 24, 100, 200, 333.33, 400, 600, 666.67, 800 MHz.

## Accelerators

| block | node | notes |
|---|---|---|
| NPU | `soc/npu@0A000000` (`artosyn,npu`) | CNN inference engine; userspace API `AR_MPI_NPU_*` |
| DSP | `soc/dsp@010c0000` (`artosyn,dsp`) | on-chip DSP |

## MPP, multimedia subsystem (`/ar_mpp`)

Capture, process, encode/decode, and display path. All nodes are `artosyn,*` IP, served by `ar_mpp_drv` and the `ar_vb`/`ar_scaler`/`ar_sys`/`ar_osal` modules through the `/proc/umap/*` interface.

| function | node | IRQ |
|---|---|---|
| Video codec H.264/H.265 encode and decode | `h26x` (`artosyn,h26x`); the IP is a Chips&Media **Wave521C** (open driver: mainline `wave5` V4L2) | `irq-h26x` (GIC 100) |
| JPEG codec | `jpeg` (`artosyn,jpeg`); the IP is a Chips&Media **CODA9** | `irq-jpeg` (GIC 101) |
| ISP, camera image pipeline | `isp` (`artosyn,isp`) | |
| Camera input MIPI CSI / DVP | `mipi`, `dvp_rx`, `vif` | |
| Display output MIPI DSI | `dsi` (`artosyn,dsi`) | |
| Video output / compositor | `vo` (`artosyn,vo`) | `dc` (GIC 102) |
| 2D graphics engine | `ge2d` (`artosyn,ge2d`) | `ge2d` (GIC 111) |
| Scaler | `soc/scaler@08840000` (`artosyn,scaler`) | `arscaler` (GIC 107) |
| GDC, geometric-distortion correction | `soc/gdc@08848000` (`artosyn,gdc`) | |
| EIS / LDC, image stabilization and lens-distortion | `eis_ldc` (`artosyn,eis_ldc`) | |
| NUC, non-uniformity correction | `nuc` (`artosyn,nuc`) | |
| IFC | `soc/ifc@08844000` (`artosyn,ifc`) | |
| AHB/AXI DMA | `ahb_dma` (`0x01e1_0000` + syscon `0x01a0_0030`), `axi_dma` (`0x0880_0000`) | AHB 8 channels (GIC 82-89), `dw_dmac_axi0..2` (GIC 95-97) |
| eFuse | `efuse` (`artosyn,efuse`) | reg `0x0024_0000`, len `0x1000` |

The display video plane is 1080p planar YUV420 (I420/YV12, not NV12); NV12 appears only as a codec-side format.

## Display subsystem (DC / VO compositor)

The display output path is a hardware compositor feeding a MIPI-DSI transmitter and an external LCD panel. It is fixed-function: software programs registers and buffer base addresses, and the silicon fetches, composites, and scans out every frame with no per-pixel CPU or GPU involvement.

**Display controller (DC / VO).** A VeriSilicon/Vivante DC8000-class controller at base `0x0881_0000` (DT node `vo`, `artosyn,vo`, IRQ `dc` GIC 102). It is a multi-layer compositor: each layer independently DMA-fetches a surface from DDR; the DC then alpha-blends the layers, performs YUV-to-RGB color conversion (CSC) and scaling, generates the raster timing, and emits pixels over the internal DPI bus. Two layers are used:

| layer | base addr reg | config reg | emit bit | vendor use | source format |
|---|---|---|---|---|---|
| primary | `0x1400` (+ `0x1530`/`0x1538` for U/V) | `0x1518` | `0x1518` bit4 | decoded video background | YV12 / I420 |
| overlay | `0x15c0` | `0x1540` (format bits[21:16]) | `0x1540` bit24 | OSD / menu, exposed as `/dev/fb0` | ARGB4444 |

`0x1580` is the overlay alpha-blend register.

The overlay is alpha-composited over the primary. The primary is the full-screen video background (default image `/usr/usrdata/nosignal.yuv`, 1920x1080 I420, replaced at runtime by the last decoded camera frame); the overlay carries the on-screen display with per-pixel alpha, so the video shows through where the OSD is transparent. Config word `0x1518` holds the pixel format in bits[31:26] (Vivante surface enum, e.g. X8R8G8B8 = 5, YV12 = 15), an output enable (bit0) and display-start/emit (bit4), a config-lock / shadow-hold (bit3), and a clear-value (solid fill) enable (bit8). The framebuffer address registers are double-buffered (shadow); a write inside the bit3 lock window latches on the next frame-start.

**Fetch-master enable (CRG `0x0a10_6018`).** The DC AXI fetch master is clocked and released at CRG register `0x0a10_6018`, on the clock/reset page at `0x0a10_6000` (distinct from the CGU leaf-gate bank at `0x0a10_4000`). Bits 0/1 clock the master, bit2 (active) releases its reset, bit12 enables it, and bits 9/10 are a separate reset pulse. Until this is programmed the DC still runs the timing generator and the clear-value path (so solid colors present on the panel), but the layer fetch DMA returns zeros for every address, so no framebuffer content displays.

**Buffer memory.** Video / primary surfaces are allocated from the reserved MMZ media pool at `0x2940_0000` (size `0x06c0_0000`), which sits above the Linux System RAM window. The DC fetches surfaces by raw physical address; there is no IOMMU and the parent `dma-ranges` is identity, so the DC and CPU views of a physical address match. The TZC-400 is left disabled, so it does not gate DC master access.

**MIPI-DSI transmitter.** A Synopsys DesignWare MIPI-DSI host at base `0x0885_0000` (DT node `dsi`, `artosyn,dsi`) with an Artosyn custom D-PHY. It serializes the DPI pixel stream onto the DSI lanes. The lane bitrate is set by a MIPI-TX PLL at DSI offset `0x2c0`; the escape/LP clock derives from `CLKMGR_CFG` at offset `0x08`.

**Panel.** External MIPI-DSI LCD (QY45043A0 in the VR04 goggle), 1920x1080 at 60 Hz, 4 data lanes, driven in video mode (a continuous pixel stream; the panel has no local frame store, so it must be fed every frame).

**Pipeline.** DDR surfaces -> DC (fetch, composite, CSC, scale, timing) -> DPI (internal parallel pixel bus) -> MIPI-DSI host (`0x0885_0000`) -> LCD panel.

Register-level reverse engineering is in `../../../kernel/dev-log/artosyn-vo-video-path.md` (primary-surface cold bring-up, the fetch-master enable, and the video-buffer to layer handoff), `../../../kernel/dev-log/artosyn-vo-flip.md` (the layer config and shadow-latch), `../../../kernel/dev-log/artosyn-dc-bus-access.md` (bus/DDR access), and `../../../kernel/dev-log/artosyn-clk-mipi.md` (the display and MIPI clock tree).

## Audio

- Audio codec `ar_mpp/audio_codec` (`artosyn,audio_codec`), reg `0x0190_0000`, IRQ `acodec` (GIC 79).
- 4x I2S: 2 master + 2 slave at `0x0170_0000` / `0x0170_2000` / `0x0170_4000` / `0x0170_6000` (DT unit-addresses `i2s@08700000`..`@08780000` do not match `reg`), IRQs GIC 74-77.

## Connectivity and peripherals

| block | node / base | notes |
|---|---|---|
| Gigabit Ethernet MAC | `soc/ethernet@08040000` (`artosyn,sirius-gmac`) + `mdio@08040000` | external Realtek PHY (id `001c.c816`). **Wired Ethernet, NOT the RF link** (the radio is the external AR8030, [`ar8030-rf-link.md`](ar8030-rf-link.md)); unused on the goggle. |
| USB 2.0 OTG | `soc/usb@08000000` (`artosyn,ar9301-usb`), `0x0800_0000` | DWC2 (`dwc2_hsotg`), IRQ GIC 106 |
| SD/SDIO x2 | `mmc0@01b00000`, `mmc1@01c00000` (`artosyn,proxima-dw-mshc`) | DesignWare, IRQ `dw-mci` GIC 80/81 |
| SDHC x1 | `sdhci@08050000` (`artosyn,proxima-sdhc`) | |
| QSPI flash | `qspi@0A000000` (`artosyn,ar9301-qspi`), controller `0x01e0_0000` | SPI-NAND boot, IRQ GIC 130 |
| I2C x5 | `i2c0@01200000` to `i2c4@01208000` (`snps,designware-i2c`) | IRQ GIC 64 (i2c0), 68 (i2c4) |
| SPI | 2 masters at `0x0110_0000` / `0x0110_2000`, 2 slaves at `0x0180_0000` / `0x0180_2000` (`snps,dw-apb-ssi`; DT unit-addresses `spi@08100000`/`@08140000`, `spi_s@08800000`/`@08880000` do not match `reg`) | GIC 70-73; only master 1 (`0x0110_2000`, `spidev`) is enabled |
| UART x6 | `serial@01500000`, `@01502000/04/06/08` (5 at `0x0150_0000`+); a 6th, `serial@01052000`, actually sits at `0x0842_2000` and is disabled (`snps,dw-apb-uart`) | GIC 43-47 (console `0x0150_0000` = GIC 43) |
| PWM x2 | `pwm@08000000`, `pwm@08040000` (`artosyn,ar9301-pwm`), `0x0100_0000`/`0x0100_2000` | |
| GPIO | `gpio@0A10A000` (`artosyn,gpio`), 7 ports | IRQ GIC 121-128 |
| ADC | `adc@0a108000` (`artosyn,adc`) + `adc-keys` | |
| RTC | internal `rtc@0a10c000` (`artosyn,ar9301-rtc`); external `ds1307` on I2C0 | |
| Temperature sensor | `temperature@0a108000` (`artosyn,protemp`) | on-die |
| Watchdog x2 | `wdt@01600000`; a 2nd, `wdt@01056000`, actually sits at `0x0842_6000` and is disabled (`snps,dw-wdt`) | IRQ GIC 69 |
| Timer | `timer@08424000` (`snps,dw-apb-timer`), disabled; Linux uses the ARMv8 architected timer | |
| Clocks | CGU `clock-controller@0a104000` (`artosyn,ar9311-cgu`), reg `0x0a10_4000` (the QSPI driver claims its clock leaf at `+0x34`); root osc `osc24M` (24 MHz) | |

## Security

- TrustZone address-space controller TZC-400 (`tzc@0x08430000`, `artosyn,tzc-400`; two register windows, `0x0843_0000` + `0x0844_0000`). Left disabled by the stock firmware.
- SPAcc crypto engine (`spacc@280000`, `picochip,spacc-ipsec`); the DT node is disabled, and the engine the vendor `ar_cipher` stack actually drives sits at `0x0118_0000`.
- Public-key accelerator `dw_pka@8410000` (`snps,dw-pka`), RSA/ECC. Backs the BootROM/SPL secure-boot verify (keyed from eFuse); the OTA image RSA-2048 check is done in software by `artosyn_upgrade` (signature at image offset `0x120`, plaintext SHA-256 digest at `0x100`).
- TRNG `dw_rng@8418000` (`snps,dw-rng`).
- eFuse at `0x0024_0000`.

## Memory

- DDR controller DMC (`ar9301-dmc`); **256 MiB** DDR mapped `0x2000_0000` to `0x2fff_ffff` (on this board: accesses at or above `0x3000_0000` fault). The stock kernel reports `MemTotal` 138432 kB because the Linux System RAM window is only `0x2000_0000` to `0x293f_ffff`; the rest is reserved for the MPP media pools. Full split: [`../memory-map.md`](../memory-map.md).
- Kernel text loads at `0x200a_0000` (same on the stock 4.9 and open 6.18 kernels).

## Memory map

| base | block |
|---|---|
| `0x001f_0000` | SRAM (128 KiB; second `reg` window of the video codec) |
| `0x0020_1000` / `0x0020_2000` | GIC-400 distributor / CPU interface (+ `0x0020_4000` / `0x0020_6000` GICH/GICV) |
| `0x0024_0000` | eFuse |
| `0x0028_0000` | SPAcc crypto (DT node; disabled) |
| `0x0100_0000` / `0x0100_2000` | PWM0 / PWM1 |
| `0x0109_0000` | EIS / LDC |
| `0x010c_0000` | DSP |
| `0x0110_0000` / `0x0110_2000` | SPI masters 0 / 1 |
| `0x0118_0000` | SPAcc crypto (the active engine, per RE of `ar_cipher`) |
| `0x0120_0000` to `0x0120_8000` | I2C0 to I2C4 |
| `0x0150_0000` to `0x0150_8000` | UART0-4 (console at `0x0150_0000`) |
| `0x0160_0000` | Watchdog |
| `0x0170_0000` to `0x0170_6000` | I2S x4 |
| `0x0180_0000` / `0x0180_2000` | SPI slaves 0 / 1 |
| `0x0190_0000` | audio codec |
| `0x01a0_0000` | syscon |
| `0x01b0_0000` / `0x01c0_0000` | SD/SDIO mmc0 / mmc1 |
| `0x01e0_0000` | QSPI controller; `0x01e1_0000` DMA (shared with `ahb_dma`) |
| `0x0800_0000` | USB 2.0 OTG (DWC2) |
| `0x0804_0000` | Gigabit Ethernet MAC and MDIO |
| `0x0805_0000` | SDHC |
| `0x0841_0000` / `0x0841_8000` | PKA / RNG |
| `0x0842_2000` / `0x0842_4000` / `0x0842_6000` | UART5 / timer / watchdog 1 (all disabled) |
| `0x0843_0000` / `0x0844_0000` | TZC-400 (two windows) |
| `0x0870_0000` | VIF (`0x0887_4000` DVP RX, `0x0887_8000` NUC) |
| `0x0880_0000` | AXI DMA |
| `0x0881_0000` | DC / VO display controller (compositor); GE2D at `0x0881_f000` |
| `0x0883_0000` | JPEG codec (CODA9) |
| `0x0884_0000` / `0x0884_4000` / `0x0884_8000` | scaler / IFC / GDC |
| `0x0885_0000` | MIPI-DSI host (DesignWare) |
| `0x0888_0000` | MIPI CSI (camera input) |
| `0x08c0_0000` | ISP |
| `0x0a00_0000` | NPU |
| `0x0a08_0000` | H.264/H.265 video codec (Wave521C) |
| `0x0a10_4000` | CGU clocks |
| `0x0a10_6000` | CRG clock/reset (DC fetch-master enable at `+0x18`) |
| `0x0a10_8000` | ADC and temperature sensor |
| `0x0a10_a000` | GPIO (7 ports) |
| `0x0a10_c000` | RTC |
| `0x2000_0000` to `0x2fff_ffff` | DDR (256 MiB) |

## Interrupt map (GIC SPI)

`arch_timer` 27 (PPI), `tzc` 32, `spacc` 35, `pka` 36, `uart0-4` 43-47, `i2c0-4` 64-68, `wdt` 69, `spi` 70-73, `i2s` 74-77, `acodec` 79, `mmc (dw-mci)` 80/81, `ahb dma` 82-89, `gmac` 90, `sdhc` 91, `mipi` 92/93, `vif` 94, `dmac_axi0..2` 95-97, `h26x` 100, `jpeg` 101, `dc` (display) 102, `isp` 103, `usb` 106, `scaler` 107, `gdc`/`eis_ldc` 108, `ifc` 109, `npu` 110, `ge2d` 111 (+116), `rtc` 118, `gpio` 121-128, `qspi` 130, `pmu` 141/142.

## Not characterized

- DDR type (the size is established: 256 MiB on this board).
- Process node and package.
- NPU throughput and supported model formats; codec maximum frame rate.

## Notes

Source: the decompiled stock device tree, on-device introspection (`/proc/device-tree`, `/proc/cpuinfo`, `/proc/iomem`, `/proc/interrupts`, `/proc/meminfo`, `lsmod`, `dmesg`) of firmware `1.0.44.rel` and of the open 6.18 slot-B kernel. Some device-tree node unit-addresses differ from the actual `reg` base (for example `qspi@0A000000` vs controller base `0x01e0_0000`); the memory map uses `/proc/iomem` where available. Platform context: <https://fpvwiki.co.uk/artosyn-kap-artlink-digital-fpv-systems-betafpv-artlink-hglrc-draco-skyzone-steadydigital-more>.

See also [`carrier-board.md`](carrier-board.md) for physical observations on the sub-board that co-locates this SoC with the AR8030 and the SPI-NAND flash inside the BetaFPV VR04 HD goggle.
