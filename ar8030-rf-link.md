# Artosyn AR8030 RF link (unofficial datasheet)

Reference for the **AR8030** (AR803x family), the digital-RF baseband that carries the FPV video/control link in the BetaFPV VR04 HD system. It is a **separate chip from the Proxima-9311 SoC** ([`soc-proxima-9311.md`](soc-proxima-9311.md)); the SoC has no integrated radio. No official datasheet is public (NDA-only); this is reconstructed from the `ar_lowdelay` / `liblowdelay_mid` / `artosyn_sdio.ko` / `daemon` decompiles, the baseband config JSON in the rootfs, and on-device observation of firmware `1.0.44.rel`. Subject to revision.

The same chip is on both ends: the goggle (`P1_GND`, Rx/master) and the air unit (`P1_SKY`, Tx).

## Scope and provenance

> **Everything here is reconstructed from one device: the goggle (`P1_GND`) side of a BetaFPV VR04 HD, firmware `1.0.44.rel`.** Two kinds of fact are mixed below, and they generalize differently.

- **Chip / baseband-firmware properties** (expected to hold for any AR8030-based link): the 5.8 GHz band plan, SDIO attach + ROM/firmware enumeration (`4152:8030` -> `4152:8031`), the `bb_ioctl` primitive and its command codes, the link framing (`0xCC`/`0xDD`/`0xEE` trailers) and TX-credit flow control, the two-transport-plane model, the adaptive-MCS / ~20 Mbps envelope, and the `video_packet_header` wire format.
- **This-device specifics** (bound to this goggle build; expect different values on another product, another firmware version, or the air-unit side): all **filenames** (`bb_demo_gnd_d.img`, `bb_config_gnd.json`, `cfg_transmedium.json`, ...; `_gnd` = ground/goggle build), all **on-device paths** (`/usr/usrdata/ar813x/`, ...), the **firmware version** `1.0.44.rel`, the **MAC/candidate values** (`ap_mac aabbccdd`, air `22000301`, the candidate slots), and the **role assignment** (this unit is the AP because it is the goggle).

## Identity

| field | value |
|---|---|
| Vendor | Artosyn |
| Part | AR8030 (AR803x family); RF firmware on this goggle `bb_demo_gnd_d.img` (`_gnd` = ground build) |
| Function | proprietary digital-RF FPV link baseband (video + audio + RC + telemetry) |
| Relation to the SoC | external companion to the Proxima-9311; **not** on-die. Attaches over SDIO. |
| Band | 5.8 GHz (raceband R1..R8 = 5658/5695/5732/5769/5806/5843/5880/5917 MHz, plus low-band channels) |
| Public docs | none (NDA-only) |

## How it attaches to the SoC

- **Bus: SDIO.** On the goggle it sits on the SoC's `mmc0` controller (`0x01b0_0000`), driven by `artosyn_sdio.ko` - the vendor's closed module on stock slot A, or the open GPL reimplementation on the slot-B stack (`kernel/modules/artosyn_sdio.c`; see "Closed vs open"). Confirmed by slot-A dmesg `1b00000.mmc0 -> "mmc1: new SDIO card"`; the microSD card is on `mmc1` (`0x01c0_0000`). It surfaces to Linux as the net interface **`sdio0`** (goggle `10.0.0.1`, air `10.0.0.100`, `/24`, `NOARP`, MTU 4096).
  - **Enumeration**: `4152:8030` (ROM loader) -> host uploads firmware + config -> re-enumerates `4152:8031` (firmware loaded and running; stock slot A shows `0x8031`).
  - **Reset**: active-low, SoC gpio23 (GPIO bank 1, line 0). The SDIO bring-up path on the open 6.18 kernel (`dw_mci-artosyn` glue + `ar_dtbo_sdio` DT overlay + `artosyn_gpio`) and the firmware-upload step are in `kernel/modules/HW-BRINGUP.md` (Phase 6).
- **Firmware**: the baseband image (`bb_demo_gnd_d.img`) is loaded into the chip by `artosyn_sdio.ko` at `insmod` (see Configuration).
- **Two transport planes over the one SDIO link** (detailed in "Channel topology" below): a **bb-socket control plane** (`0xDD` frames on `/dev/artosyn_sdio`, chip/MAC/PHY control only) and an **IP-tunnel plane** (`10.0.0.x` UDP over `sdio0`, `0xCC` frames). All media AND the FSM/message/telemetry/OSD traffic ride the IP-tunnel plane as UDP (video `:10001`, message/telemetry `:10000`).
- **Pairing/bring-up**: the `auto_sync` daemon (`BB_SYNC_MSG_ASK/INF/ACK`) over the baseband; the goggle is the **`ap`/master**, the air unit a station. Topology observed **1V1** (one master, one air unit).

## Roles: the goggle is the AP, the air unit is the client

- The **goggle (VR04)** is the **AP / master**: `role="ap"` in `bb_config_gnd.json`. **It does not scan.** It broadcasts as a known AP and accepts its *bound* peers. (The vendor SDK names the goggle side "TX": `AR_AR8030_TX_*` ioctls, `AR_FSM_TX_8030_LINK_UP/DOWN` - link-FSM naming, unrelated to who transmits the video.)
- The **air unit** is the **DEV / client**. *It* scans for the AP and connects, one of the AP's bound candidate slots.
- **Association is firmware-automatic from the config loaded at `insmod`.** To current knowledge there is **no AP-side "scan" or "connect" command** in the host software: the scan ioctls `GET_ScanResult`, `SET_ChnMode`, and `SELECT_ChnIndex` appear only on the `AR_AR8030_RX_*`/DEV path (RE of `ar_lowdelay`). The AP just needs to be powered, configured, and broadcasting; the air unit finds it.
- **Link-up is event-driven**, not polled: the chip pushes a `BB_EVENT_LINK_STATE` callback (value `2` = linked) when the DEV associates. `ar_lowdelay` registers for it via `AR_AR8030_TX_SetCb` (a local-only ioctl, `0x02000000`).

## RF parameters

| parameter | value | notes |
|---|---|---|
| Topology | 1 master (goggle) + 1 air unit | `ap`/station; `1V1` |
| Channels | **Race = 16**, Normal/Freestyle/Long-Range = **3** | bitmaps `0x0007FFF8` (bits 3..18) vs `0x00000007` (bits 0..2), in `chan_valid_bmp*.json` |
| RF channel bandwidth | broadcast/control channel **2.5 MHz**; video payload **10 MHz** default (**5 MHz** in the Long-Range `_5m` profile) | a config field (`"bandwidth"` MHz), distinct from the clock. See Configuration below. |
| Adaptive bandwidth (`bw_auto`) | range **5..20 MHz**, currently **disabled** | so the chip/firmware supports up to **20 MHz**; the shipped configs use fixed bandwidths instead |
| MCS | adaptive (auto), observed live oscillating 8..10 | proprietary modulation; per-mode default from JSON |
| Throughput | up to ~20 Mbps | `GetThroughput` / `GetRealThroughput` |
| TX power | settable ~5..32 dBm (observed "23"); auto available | each end sets its OWN TX power via `bb_ioctl` `SetPower` (`0x02000008`) + `SetPowerAuto` (`0x02000009`); the goggle issues these at video bring-up. dBm calibration in the LD config (offset 0x68) / `ar8030_pwr.json` |
| Antennas | 2, TX beamforming on | |
| Baseband clock | 200 MHz | `"clock": "200MHz"` in the config; fixed, separate from the channel bandwidth above |
| Link latency | RTT ~100 ms on the IP channel | the low-delay video path is separate and faster |
| Addressing | air MAC e.g. `22000301`, `ap_mac aabbccdd` | |

The per-mode value table (Race/Freestyle/Long-Range -> {power, MCS, bitrate, bandwidth}) lives in the stock UI, not these binaries (see [`../rf-modes.md`](../rf-modes.md)). The **work mode** (Race/Freestyle/Long-Range) is owned by the **air unit** (via its low-delay config `SetLdCfg` + `SET_Tranmit_Param` cmd 0x24); the goggle only displays it.

## Control interface (`bb_ioctl`)

The vendor's `AR_AR8030_{RX,TX}_*` functions are thin wrappers over a single primitive, `bb_ioctl(handle, code, in, out)`, exposed by the baseband driver. Those wrappers are static inside `ar_lowdelay` (not linkable), so the open code (`userspace/ml-linkd/bb-cmd.h`) reimplements them over `bb_ioctl`. The command class selects GET (`0x010000xx`) vs SET (`0x020000xx`).

| code | call | in / out |
|---|---|---|
| `0x01000005` | RX GetDistance | in `{u8:1}`, out distance (u32, m) |
| `0x01000006` | RX GetThroughput | in `{u16:1}`, out u64 (throughput in high u32) |
| `0x01000011` | RX GetRealThroughput | in `{u16:0x200}`, out vReal@8 / tReal@12 |
| `0x0100006b` | RX GetQuality | out 8 B (3x u16 + 2x u8; RSSI/SNR/MCS + flags, mapping unconfirmed) |
| `0x02000006` | RX SelectChn | in `{u8:2, u8:index}` |
| `0x02000015` | RX SetRemote | paired with select-chn |
| `0x0200000c` | SetMcs / "Mode" | MCS mode in the high byte of a u16 |
| `0x02000008` | TX SetPower | in `{u8:0, u8:dbm}` |
| `0x02000009` | TX SetPowerAuto | |
| `0x02000000` | SET callback | handled locally, never sent to the chip |
| `0x01000001` | GET pair candidate bitmask | polled in pair mode |
| `0x0100000a` | GET scan result | DEV-only |
| `0x01000003` | GET candidates | |
| `0x02000002` | SET pair mode | |
| `0x02000003` | SET ap mac | |

Band (channel-set) select is one layer up, in `ar_lowdelay`/`liblowdelay_mid` (`SetFastMode` cmd 0x3c / `GetFastMode` 0x3d, also the `/usrdata/lowdelay/fastmode` file): Race vs Normal. Out-of-range channel/MCS values crash the firmware, so clamp before writing.

### Wire payload length is fixed per command (crash-relevant)

`bb_ioctl` does not send the caller's struct size on the wire. It looks the command up in a fixed **92-entry `{code, in_len, out_len}` table** (`get_bb_ioctl_cmdiptlen` @VA 0x4afa40 reading the array at VA 0x4eb480 in `ar_lowdelay`) and always stamps that table's `in_len` as the frame payload length, zero-padding the caller's buffer up to it. So the wire length is a property of the command, not of how many bytes the caller filled in: pair mode enters with just `{0x01, 0x01}` meaningful but goes out as a **14-byte** payload; the pair-lock's `{0x00,0x01,MAC}` goes out as **22 bytes**. Sending the short "meaningful" length instead is a malformed frame, and malformed RF setters have crashed the firmware, so this length is authoritative for any new setter we wire up.

The full table (all 92 `{class, selector, in_len, out_len}` rows, transcribed from the ELF) lives in `userspace/ml-linkd/bb-cmd.h` as `bb_cmd_lens[]`, where `bb_build_cmd()` uses it to pad every frame. Read it there rather than duplicating it here; `in` = host-to-chip payload bytes, `out` = chip reply payload bytes, both excluding the 19-byte frame envelope.

## Link transport (framing, credit)

- Link framing (4-byte trailer `{0x00,type,len_lo,len_hi}`): **`0xCC`** = IP tunnel, **`0xDD`** = command, **`0xEE`** = flow-control ACK.
- **TX credit**: the chip grants send credit via event reg `0x68` bit3 -> byte count in reg `0x60` (`<<9`). The host transmits only while it holds credit. (The host must also *drain the chip's RX FIFO* for the firmware to advance and grant that credit; this recv-drain step is implemented in `kernel/modules/artosyn_sdio.c`.)

## Software stack (three layers)

1. **`artosyn_sdio` (kernel)**: SDIO transport, firmware+config upload, the bb-link (`0xCC`/`0xDD`/`0xEE`), TX-credit flow control, and the `sdio0` IP tunnel.

2. **`daemon -i 1` (passive RPC <-> bb-socket bridge)**: listens on **TCP port 50000**; relays RPC frames to/from `/dev/artosyn_sdio`. It has **no link/scan/connect logic** (its main thread is a hotplug-poll loop). It opens bb-sockets only when an RPC client commands it, and **drops chip->host RX unless an RPC client has opened the bb-socket** for that slot/port (`"%d-%d refuse write opt while remote disconnect"`; "remote" = the TCP client, not the RF peer). Args: `-i` interface (0=usb/1=sdio/2=uart), `-p` rpc port (default 50000).

3. **`ar_lowdelay` (link FSM + data plane)**: the RPC **client** of the daemon (connects to `127.0.0.1:50000`). Owns the TX link FSM (`AR_FSM_TX_8030_LINK_UP/DOWN`). Per session it: calls `bb_dev_getlist` (`get_workid_list`) to read the chip's associated-device list; opens the per-`(slot,port)` **data sockets** via `ld_socket_open` (video AR8030 port = 1, plus msg/audio/rc); and registers the `BB_EVENT_LINK_STATE` callback. **This is the background service that drives the data plane, not the menu UI**, which is why a replacement/libre menu does not break the link, but running the daemon alone never moves data.

### RPC wire protocol (TCP 50000)

```
0xAA | plen(u32 LE) | channel(byte5) | opcode(byte6) | slot(byte7) | port(byte8)
     | wordB(u32 BE) | wordC(u32 BE) | cksum(byte17) | payload[plen] | 0xBB
```

- `channel` `0x04` = bb-socket class (stamped into byte5 on the chip-bound frame).
- `cksum` = `~XOR(byte0..byte16) & 0xFF`; a bad checksum makes the parser resync to the next `0xAA`.
- Opcodes (socket-node sender FSM): `0x00` = **open-socket** (`"socket try init slot %d port %d"` -> `"slot %d port %d socket has opened!"`), `0x01` = data, `0x03` = close.
- Embedded `bb_ioctl` codes (in the control payload): `0x010000xx` = GET, `0x020000xx` = SET (see the `bb_ioctl` table above).

## Channel topology (two transport planes)

One SDIO link multiplexes **two independent transport planes**; the "AR8030 video / message / audio / remote-control sockets" that `ar_lowdelay` logs are **UDP sockets on the IP plane, not separate physical channels** (from `ar_lowdelay` disasm + pcaps).

### Plane A - bb-socket / chip control (`/dev/artosyn_sdio`, `AA..BB` frames, `0xDD` trailer)
Chip/MAC/PHY control only; carries no media. Frame envelope: `AA | len(u16 LE) | 00 00 | cmd(32b BE) | wordB(BE) | wordC | cksum=~XOR(byte0..byte16) | payload | BB`.

| channel | carries |
|---|---|
| **ch01** | polls / GETs: link poll (port `0x73`), video-servicing poll (`0x0c`), `getlist` |
| **ch02** | SET / `bb_ioctl` (cmd `0x020000XX`): `SET_POWER 0x08`, `POWER_AUTO 0x09`, MCS `0x0c/0x0d`, BANDWIDTH `0x16` |
| **ch04** | socket-OPEN: **subscribes** the chip's per-`(slot,port)` data plane (video port 1, ...). A SUBSCRIPTION that tells the chip to forward the IP tunnel for that stream - NOT the byte pipe the media rides. |
| **ch05/port01** | chip ASCII debug log (`cur type:X req type:Y`, `IDLE->LOCK->CONNECT`, mcs, pwr) |

### Plane B - IP tunnel / UDP (`sdio0` netdev, `0xCC` trailer)
Carries **all media AND all FSM control messages** as ordinary UDP datagrams between `10.0.0.1` (goggle) and `10.0.0.100` (air). Each vendor "AR8030 socket" is a UDP port = `Ar8030PortNum + 10000` (`ld_socket_open` @`0041d170` does `socket(AF_INET,DGRAM)`, `bind`/`sendto` port `n+10000`).

| stream | UDP port | direction | payload | on VR04 |
|---|---|---|---|---|
| **VIDEO** | `:10001` | air->goggle | H.265 (36B `video_packet_header_t`, 2 tiles) | enabled |
| **MESSAGE** (= "telemetry") | `:10000` | bidirectional | FSM common msgs (`PARAMS_REQUEST`, `NO_PARAMS`, `IDR_REQUEST`, `LINK_UP/DOWN`, `HDMI_*`, `MCS_CHANGE`) **and** status/OSD (types `0x09/0x10/0x11`) | enabled |
| **IDENTITY** | `:20001` | bidirectional | the 3-way identity hello (see "Identity handshake") | enabled |
| **AUDIO** | `n+10000` | air->goggle | AAC | disabled (`AudioEnable:0`) |
| **REMOTE-CONTROL** | `n+10000` | goggle->air | RC | disabled (`RcEnable:0`) |

Two non-obvious points:
1. **"Telemetry `:10000`" and the "message channel" are the SAME UDP socket.** The battery/OSD status frames (types `0x09/0x10/0x11`) and the video-control FSM messages (`PARAMS_REQUEST` etc.) all ride `:10000`, sharing one 20-byte common-message header (`msg_type @0`, `timestamp_us @8`, `payload_len @16`, `payload @20`). The "telemetry" on `:10000` is the air->goggle half of the message channel.
2. **Video (`:10001`) and messages (`:10000`) are UDP**, carried as ordinary IP over the `0xCC` tunnel (a successful `ping 10.0.0.100` is a simple proof of association). The ch04 bb-socket opens on Plane A only arm the chip's forwarding; the bytes are UDP on Plane B.

### The link as a general IP device (and where that ends)
Plane B is a normal point-to-point network device: both ends run Linux with an IP interface on it, so any IP traffic works over the tunnel (ping, SSH, arbitrary UDP/TCP). The vendor "video / message / audio / RC sockets" are UDP ports on this device (`Ar8030PortNum + 10000`) - applications on top of the tunnel, not a special transport.

The ownership boundary sits at the IP layer:
- **At/above IP:** ordinary networking, fully open. Any peer program on either end can send anything; the vendor stream is one application among possible others.
- **Below IP:** the closed AR8030 baseband firmware - association, modulation/MCS adaptation, the ~20 Mbps ceiling, the TX-credit flow control, and the `type:3`/`type:8` bandwidth mode. Usable but not modifiable; any traffic inherits these characteristics.

A custom media path (a source on the air unit sending UDP straight to the goggle) skips the vendor video-start handshake entirely, because that handshake exists only to drive the vendor firmware's media pipeline, not the link itself. Such a path still rides the closed baseband, so it still needs the high-bandwidth link mode (`type:8`) for video-rate data and its own capture/encode on the air unit.

### Identity handshake (`:20001`, three-way)
Before the media-params exchange, a 3-way hello on `:20001`: the goggle sends 520 B type-0 probes (~3 Hz); the air answers type-1 (its identity, once); the goggle must echo the air's type-1 back as a type-2 ACK (`byte[0]=0x02, byte[5]=0x00`). Without the ACK the air retransmits type-1 forever and never advances. The goggle stops the type-0 probe once this completes.

### Video-start handshake (`:10000`, three-way)
The air unit begins streaming `:10001` video only after a three-message exchange on the `:10000` message socket, each message using the 20-byte common-message header (`msg_type @0`):

| step | direction | `msg_type` | name | size | notes |
|---|---|---|---|---|---|
| 1 | goggle->air | 1 | `PARAMS_REQUEST` | 24 B | retransmitted ~every 2 s while the air is in `NO_PARAMS`; stops on the reply |
| 2 | air->goggle | 2 | `MEDIA_PARAMS` | 104 B (20 B header + 72 B params + 12 B trailer) | codec/resolution/fps; sent once per request, not retransmitted |
| 3 | goggle->air | 3 | ACK/START | 24 B | ~70 ms after step 2; sent once, not retransmitted. The air starts `VideoSend` on receipt |

UDP has no transport ACK, so reliability is application-level and asymmetric: only the request (step 1) is retransmitted (poll until answered), which also covers a lost reply; the reply and the ACK are fire-once and rely on the low-loss local link. A mid-session re-negotiation repeats the same exchange. Recovery of a lost keyframe is a separate `:10000` `IDR_REQUEST` message.

### Video stream format (`:10001`, air->goggle)
Each `:10001` UDP datagram is one vendor video packet: a **36-byte little-endian `video_packet_header` + StreamLen bytes of HEVC + a 4-byte tail**. Field layout (mirrors the `struct video_packet_header` in `kernel/modules/artosyn_sdio.c`):

| off | field | notes |
|---|---|---|
| +0  | `MagicCode` | `0x12345678` |
| +4  | `StreamLen` | HEVC payload byte count |
| +8  | `ChnIndex` | encode channel; 0 and 1 interleaved (see tiling) |
| +12 | `isIdrStream` | IDR flag |
| +16 | `FrameId` | |
| +20 | `TimeStap` | (sic) timestamp |
| +24 | `Resolution` | low16 = height, high16 = width (`0x07800438` = 1920x1080) |
| +28 | `TailMagicCode` | `0x87654321` |
| +32 | `CrcCode` | CRC-32 (poly `0xedb88320`) over the first 32 header bytes |
| tail | `0x87654321` | 4 bytes after the payload |

- **Codec:** H.265/HEVC (venc8). Bit-exact decodable by mainline ffmpeg AND the open `wave5` V4L2 decoder (`kernel/STATUS.md` "Open codec").
- **Two encode channels = two vertical TILES of one 1080p frame** (NOT stereo/PIP): `ChnIndex 0` = top band (per-channel SPS 1920x560), `ChnIndex 1` = bottom band (1920x552). Each tile is an independent HEVC elementary stream with its own VPS/SPS/PPS. Demux one channel to decode it (`ml-rf-udp.py --chn N`). **The tiles overlap by 32 rows** (560 + 552 = 1112 = 1080 + 32): the bottom tile's top 32 rows carry the same image content as the top tile's bottom 32 rows. The full 1080-row frame is top tile rows 0..559 + bottom tile rows 32..551 (520). Stacking both tiles in full instead duplicates a 32-row band at the centre.
- **IDR policy:** exactly one IDR per channel at `FrameId 0` (session start), then all P-frames - no periodic IDR. A mid-stream join therefore sees P-frames only; capture from association to include the keyframe. A recovery IDR is request-based (the `:10000` `IDR_REQUEST` message).
- **In-band SEI, one PREFIX_SEI (NAL 39) per frame,** ASCII: `ChnId N FrameId N PTS <hex> ... BR 3926 QP N` - a frame-aligned metadata sidechannel.

Authoritative extractor + the byte-exact spec: `libre/tools/ml-rf-udp/` (reads an `ml-sniff` pcap, verifies the magics/CRC, writes the HEVC elementary stream, prints a `DECODABLE:` verdict).

## Configuration (the `bb_config` JSON, and how it reaches the chip)

The AR8030 is configured by a JSON file loaded **into the `artosyn_sdio.ko` driver at `insmod`**, alongside the firmware image, not (only) by runtime ioctls. The files live in `/usr/usrdata/ar813x/` (plus a `/usr/usrdata/product/ar813x/` copy). Everything below is read directly from those files on this unit.

`bb_config_gnd.json` (the main config) is a `baseband.basic.ap` tree:
- `"clock": "200MHz"`, `"ant_num": 2`, `"role": "ap"`, `init_band "5G"`, `band_mode "band_single"`.
- `user_mode`: `"mode": "1V1"` (one air unit), fixed slot, `compact_br`.
- `mcs`: init `"auto"`, plus a `"bw_auto"` block (adaptive bandwidth) with `range { low "5", high "20" }` MHz, but `"enable": false`, so adaptive BW is off and the fixed `para` bandwidths apply.
- `para.br` (the broadcast/control channel): `"bandwidth": "2.5"`, 1TX / 1T1R.
- `para.slot` (the video payload channel): `"bandwidth": "10"`, with interleaving, 1TX / **1T2R** (two-antenna receive diversity).
- `bb_config_gnd_5m.json` is identical except the two payload `"bandwidth"` fields are `"5"` instead of `"10"` (the Long-Range profile). That single field is the only difference between the two files.

How a session is brought up (`start_ar813x.sh`), the merge that configures the AP:

```
bb_config_gnd.json        (base; role=ap)
  + ar8030_pwr.json       (TX power calibration, from /factory)
  + chan_valid_bmp.json   (legal RF channel bitmap)
  + usr_cfg.json          (the REAL bound candidate slots + ap_mac, from
                           /usrdata/lowdelay/lowdelay_cfg)
  = bb_config_gnd.json.usr_cfg.json
```

1. `auto_merge` builds the merged working copy above.
2. `insmod /mod/artosyn_sdio.ko fw_name=bb_demo_gnd_d.img cfg_name=<merged>.json`; the driver loads the **firmware image** (`bb_demo_gnd_d.img`) and the merged config into the AR8030 over SDIO at module init.
3. Runtime adjustments after bring-up (channel, MCS, TX power) go through `bb_ioctl` (table above), not the JSON.

Real merged values (captured from a working slot A): `role=ap`, `ap_mac=aabbccdd`, `candidate.slot=[..., e515815c, e59522b7]`, `chan_valid_bmp=0x0007FFF8`, power cal present. The raw on-disk `bb_config_gnd.json` alone holds **placeholders** (`candidate=["aabbccdd"]`, `ap_mac="66000000"`) and no channel/power data. Uploading it instead of the merged file leaves the chip broadcasting on nothing / accepting no one.

**Upload mechanism** (driver, RE-verified byte-exact vs the vendor): the config blob is streamed through the SD-framed ROM loader to **`troot_load_addr`** (read from the firmware image header; `0x00200000` in this image), *after* the 3 firmware segments (SPL/troot/sig) and *before* the `addr=0/len=0` finalize. There is no config-specific header field; the troot load-address slot is reused, and the real config overwrites the 16 KB troot placeholder at that address. The firmware reads the config from there on boot.

So bandwidth/bandwidth-profile/clock/antennas are set **at driver load** from the JSON; channel and MCS and power are tweaked **live** via `bb_ioctl`. To change the channel bandwidth you would edit the JSON (or swap to the `_5m` file) and reload the driver, not call an ioctl (there is no runtime `SET_BANDWIDTH`). The shipped fixed bandwidths are 2.5, 5, and 10 MHz; the (disabled) `bw_auto` range of 5..20 MHz shows the firmware supports up to 20 MHz, but values above 10 MHz or below 2.5 MHz are not characterized, and out-of-range RF values can crash the firmware (see [`../rf-modes.md`](../rf-modes.md)).

Other config files in the same directory: `chan_valid_bmp.json` (normal-mode channel bitmap `0x00000007` = 3 channels), `chan_fast_valid_bmp.json` (race bitmap `0x0007FFF8` = 16), `ar8030_pwr.json` (a `pwr_offset` calibration array, `[0,0,0,0]` on this unit, merged into the bb_config; this is a per-path power trim, not the operating TX power level, which comes from the LD config / `SetPower`), `/usrdata/lowdelay/fastmode` (the Race/Normal band flag).

### Transport selection and link adaptation (`cfg_transmedium.json`)
Maps each media stream to a transport and sets the AR8030 link-adaptation parameters. The captured goggle (slot A) and air-unit copies are byte-identical:
- **Transport**: `Ar803xEnable: 1`, `EthEnable: 0` - the media rides the AR8030 baseband link, and the SoC's separate **wired** Ethernet MAC is off. This does NOT mean IP is off: the media still travels as **UDP over the `sdio0` IP tunnel**, which is itself carried over the AR8030 baseband (`EthEnable` = the wired GbE, not the IP-over-radio tunnel; see "Channel topology"). `ApNum: 1`, `DevNum: 1` (one air unit).
- **Per-stream enable + port**: `VideoEnale: 1` (AR8030 port 1), `AudioEnable: 0`, `RcEnable: 0`, `MsgEnable: 1`; each stream carries both an `*Ar8030PortNum` and an `*EthPortNum`.
- **Rate cap**: `ArMaxBitRate: 20000` (and `EthBitRate: 20000`), i.e. 20 Mbps.
- **MCS / link adaptation**: `Ar803xMcsMode: "auto"`, `Ar803xMcsLevel: 5`, `Ar803xMinMcs: -1`; throughput targets `Ar803xThroutputRate: 0.7` and `f32Ar803xThroutputRateLowMcs: 0.8`; link thresholds `Ar803xRxDataSNRThreshold: 10`, `Ar803xRxDataLDPCErrThreshold: 10`, `Ar803xRxLinkSNRThreshold: 10`.
- **Beamforming**: `Ar803xTxBFEMEnable: 1`.
- **Role**: `Ar803xRole: "Tx"` on **both** ends (goggle and air unit carry identical files), so this field does not encode the media direction; the per-end role comes from `bb_config`'s `role` (ap vs dev).

## Association sequence (stock slot A)

1. `insmod` uploads firmware + merged config -> chip boots `0x8031` as the AP and broadcasts (ap_mac, candidate list, channel bitmap, power cal).
2. The air unit (DEV, a bound candidate) scans, finds the AP, and connects.
3. The chip adds it to its device list and fires `BB_EVENT_LINK_STATE` (=2). The IP tunnel is now up (ping works).
4. `ar_lowdelay`: `bb_dev_getlist` sees the device -> opens the data sockets -> video flows to the app (and onward to RTSP / the goggle display).

## Pairing: binding a new air unit (distinct from association)

Normal association reuses an already-bound peer. *Binding* a new air unit is the `ar_lowdelay -bb_pair` path (`AR_AR8030_TX_BbPair`): SET ap_mac = broadcast (`0x02000003`) -> SET pair mode (`0x02000002`) -> poll `bb_ioctl 0x01000001` every 20 ms (for `timeout x 50` iters) for a candidate bitmask -> on a hit, read the peer MAC, lock ap_mac to it (`0x02000003`), and persist. This is the only TX-side poll loop, and it applies only while in pair mode.

## Replicating the data plane on the open stack (slot B)

The open stack talks **directly** to the two planes - `/dev/artosyn_sdio` (Plane A, bb-socket control) and `sdio0` (Plane B, UDP) - with no vendor `daemon` and no `:50000` RPC: media is UDP-pushed on the IP plane, not RPC-pulled, and the vendor daemon's real control traffic is `ch01`/`ch02`, not `ch04` socket-opens. The working sequence (proven by `libre/tools/ml-rf-video`, productionized in `ml-linkd/`) is:

1. Bring up the link: `insmod artosyn_sdio` (firmware + merged config) -> `sdio0` up -> air unit associates. Association proof = `ping 10.0.0.100`.
2. Plane A poll cadence: `port 0c` ~24 Hz, `port 73` ~6 Hz (every 4th tick), `ff/02` ~3.4 Hz (every 7th tick) (match the vendor; faster wedges the air).
3. Plane A TX power: `SetPower` + `SetPowerAuto` -> the chip log reaches `cur type:8` (the high-bandwidth video link type).
4. Plane B `:20001` identity handshake (type-0 probe -> air type-1 -> type-2 ACK).
5. Plane B `:10000` handshake: send `PARAMS_REQUEST` (`msg_type=1`, retried ~2 s); on the air's `MEDIA_PARAMS` (`msg_type=2`) reply, send the `msg_type=3` ACK. The air then streams.
6. Read the H.265 video as UDP on `:10001`; decode/deframe with `libre/tools/ml-rf-udp`.

## Closed vs open

- **Closed (irreducible):** the AR8030 silicon and the signed baseband firmware blob it runs (`bb_demo_gnd_d.img` on this goggle) - the PHY, the MAC, modulation/MCS adaptation, association. This is the only closed piece. It is loaded and used unmodified; nothing below `bb_ioctl` is reimplementable without RF/silicon docs.
- **Open (reimplemented on the slot-B stack):**
  - the **`artosyn_sdio.ko` driver itself** - a GPL-2.0 reimplementation from scratch (`kernel/modules/artosyn_sdio.c`) that reproduces the vendor module's userspace ABI and on-wire contract (SDIO fw upload, `/dev/artosyn_sdio` ioctls, the `sdio0` IP tunnel, the `0xCC`/`0xDD`/`0xEE` framing + TX-credit), so the firmware blob runs unchanged on top;
  - the **`bb_ioctl` command set** and the link-control layer built on it (`libre/sdk/include/link.h`);
  - the **data plane**, driven directly against the two transport planes (`/dev/artosyn_sdio` + `sdio0`) by `libre/tools/ml-rf-video` (bring-up cadence + TX power + the `:10000` handshake) and read/decoded by `libre/tools/ml-rf-udp`. No vendor `daemon`, no `:50000` RPC (see "Replicating the data plane").

So the open stack reuses exactly one closed artifact: the baseband firmware image. Everything from the SDIO driver up is open.

## Related / Sources

- [`soc-proxima-9311.md`](soc-proxima-9311.md), the application SoC the AR8030 attaches to (and which has the unrelated, unused wired Ethernet MAC).
- [`carrier-board.md`](carrier-board.md), physical observations on the sub-board that co-locates this chip with the Proxima-9311 SoC and the SPI-NAND flash.
- [`../rf-modes.md`](../rf-modes.md), the Race/Freestyle/Long-Range modes in detail.
- [`../air-unit-firmware.md`](../air-unit-firmware.md), the air-unit (Tx) side.
- `libre/sdk/include/link.h`, the open control-layer header.
- `kernel/modules/artosyn_sdio.c`, the open GPL reimplementation of the SDIO driver (framing, TX-credit, recv-drain, the `sdio0` IP tunnel, the `video_packet_header`).
- `kernel/modules/HW-BRINGUP.md`, open slot-B bring-up.
