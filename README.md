# OpenWrt for ipTIME BE3600plus

**English** | [한국어](README.ko.md)

> [!TIP]
> **Choosing a router for OpenWrt?** Pick one with official OpenWrt support instead, for example an ipTIME model on MediaTek Filogic (such as the AX3000 series). This port is for people who already own a BE3600plus.

An unofficial OpenWrt port for the ipTIME BE3600plus (Airoha AN7563, 256 MiB RAM, SPI-NAND, MT7990 Wi-Fi). It keeps the stock U-Boot and never touches the bootloader. OpenWrt is installed into the second stock firmware slot, so the stock firmware stays in the first one.

The port shares its source with the [ipTIME BE5100M port](https://github.com/junkim0215/iptime-be5100m-openwrt).

> [!WARNING]
> This is unofficial firmware. Installing it may void your warranty, and if a flash goes wrong you may need a UART connection to recover. Use it at your own risk.

## Status

| Feature | Status |
|---|---|
| 3 LAN ports, WAN (DHCP) | Working |
| Wi-Fi 2.4 / 5 GHz | Working (KR regulatory domain), through the CPU (see below) |
| Hardware routing offload (PPE) | Working, on by default |
| LEDs (status, 2.4G, 5G, ports), reset and WPS buttons | Working |
| Stock MAC addresses | Working (from r147) |
| LuCI, sysupgrade (keeping settings) | Working |
| NPU Wi-Fi acceleration | **Not supported** (MT7990) |
| Long-term stability | **Untested** |

Tested on a single unit.

### Throughput (r147, hardware offload on, TCP)

| Test | Result |
|---|---|
| Wired upload, UDP 20 Mbit/s and TCP 200 Mbit/s, 60 s | no stall, no loss |
| Wi-Fi 5 GHz 160 MHz upload | about 930 Mbit/s |
| Wi-Fi 5 GHz 160 MHz download | about 870–890 Mbit/s |

Wi-Fi is limited by the CPU. The NPU Wi-Fi offload of the AN7563 supports only the MT7992 (BE5100M, Xiaomi BE5000), not the MT7990 in this router. Wired routing still goes through the hardware offload.

5 GHz Wi-Fi starts at 80 MHz. 160 MHz can be set in LuCI (Network → Wireless).

## Downloads

Get the files from [Releases](../../releases).

| File | Use |
|---|---|
| `…-squashfs-factory_16_006.bin` | install from the stock web UI |
| `…-squashfs-sysupgrade.bin` | upgrade from OpenWrt (keep settings) |
| `…-squashfs-slot.bin` | U-Boot TFTP install (UART) |

## Installing (no UART, via the stock web UI)

1. Open the stock admin page (`192.168.0.1`) → 관리도구 (Management) → 시스템 관리 (System) → 펌웨어 업그레이드 (Firmware upgrade).
2. Upload `openwrt-airoha-an7563-iptime_be3600p-squashfs-factory_16_006.bin` as it is. The file name must contain `_16_006`.
3. Do not power off; the router reboots by itself. Check whether LuCI comes up at `http://192.168.1.1` (user `root`, no password).
   - If LuCI comes up, set a root password.
   - **If OpenWrt (LuCI) does not come up within 5 minutes and the stock firmware returns** (the router reboots once by itself in between), the stock firmware was running from the second slot. Upload the official stock firmware (from the [ipTIME download page](https://iptime.com/support/downloads?dfsid=21&dftid=1011)) through the web UI once more, which brings it back to the main slot, then repeat steps 1–3.

## Installing or recovering with UART (advanced)

Use this if the web install is not possible or OpenWrt no longer boots. You need a 3.3 V USB-UART adapter (connect GND, TX and RX only; never VCC), a TFTP server on a PC at `192.168.0.100`, and the `*-slot.bin` image.

1. Connect the UART (115200 8N1), power on and press any key at `Hit any key to stop autoboot` to get the U-Boot `ECNT>` prompt (the countdown is short).
2. Set the PC to a static IP `192.168.0.100/24`, connect it to a LAN port (e.g. LAN1) and start the TFTP server (on Windows, allow TFTP in the firewall). Then run (replace `<len>` with the file size rounded up to a multiple of `0x20000`):

```
tftpboot 0x88000000 openwrt-airoha-an7563-iptime_be3600p-squashfs-slot.bin
crc32 0x88000000 <len>
flash erase 0x2D00000 <len>
flash write 0x2D00000 <len> 0x88000000
flash read 0x2D00000 <len> 0x89000000
crc32 0x89000000 <len>
flash erase <0x2D00000+len> <0x2800000-len>
bootflag read
reset
```

3. Both `crc32` values must match the CRC of the file. If `bootflag read` shows `0`, run `bootflag swap` before `reset` so the second slot (OpenWrt) boots.

> [!CAUTION]
> Write only to `0x2D00000`–`0x54FFFFF` (the second slot). Do not run `saveenv` and do not write to any other address. To boot the stock firmware again from U-Boot, run `bootflag swap` so that the flag is `0`.

## Upgrading

From inside OpenWrt, use LuCI → System → Backup / Flash Firmware and upload a `sysupgrade.bin` (keeping settings is fine). Do not use `-F` (force).

## Returning to stock (no UART)

Run the following in OpenWrt; U-Boot then switches to the stock firmware on the next boot. The stock firmware must still be in the main slot (the normal installed state). Back up your settings first.

```
fw_setenv boot_try 0 && reboot
```

## Known limitations

- No NPU Wi-Fi acceleration (see above). The NPU page in LuCI shows only gaming mode, PPPoE MSS and MLO.
- The stock reset-button TFTP recovery mode has not been tested. ipTIME Firmware Wizard does not support BE models.
- After a sysupgrade, the reboot sometimes hangs (about 1 in 3, same as on pabragin's BE5000). The image and settings are already written: unplug the power and plug it back in.
- Only the official `base`, `luci`, `packages`, `routing`, `telephony` and `video` package feeds are kept. Kernel modules that are not in the image cannot be installed with `apk add kmod-…`.

## Source and building

The source is the `be3600p` branch of junkim0215/openwrt. Each release is tagged
at the commit it was built from (`be3600p-rNNN`); the tag and commit are listed
in the release notes.

```
git clone https://github.com/junkim0215/openwrt.git && cd openwrt
git checkout be3600p-rNNN    # e.g. be3600p-r147
./scripts/feeds update -a && ./scripts/feeds install -a
cp <diffconfig> .config && make defconfig && make -j$(nproc)
```

## Other ipTIME devices

- [ipTIME BE5100M](https://github.com/junkim0215/iptime-be5100m-openwrt) (Airoha AN7563)
- [ipTIME BE5100plus](https://github.com/junkim0215/iptime-be5100plus-openwrt) (MediaTek MT7987, preview until official OpenWrt support)

## Credits

This port builds on the work of:

- **Ray Liu (xinyooo)**: AN7563 base support (pin controller, NAND bad-block
  tables, U-Boot) and the Xiaomi BE5000, openwrt/openwrt#23960
- **Iscle**: AN7563 Ethernet, switch, PCIe and PPE forwarding
- **Pavel Bragin (pabragin)**: the Xiaomi BE5000 build this port is based on
- **Paulo José Rolim Braga Filho (paulojjj)**: the QDMA reset fix
  (openwrt/openwrt#26755)
- **kyoto44**: the MT7990 TXD fix (openwrt/mt76#1043, #1065)
- The OpenWrt project

This port was written with the help of Claude (Anthropic), and the changes were tested on real hardware.

Licensed under GPL-2.0.

The image also contains binary firmware for Wi-Fi and the NPU. See the license information of the corresponding OpenWrt packages.
