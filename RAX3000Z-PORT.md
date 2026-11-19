# CMCC RAX3000-Z port (ZX279128R) — on top of the E2631/RAX1800-Z overlay

This adds the **CMCC RAX3000-Z (普通版)** as a third device in this sanechips
overlay, next to the ZTE E2631 and the CMCC RAX1800-Z. The RAX3000-Z is the
**same ZX279128R SoC** as the RAX1800-Z and shares its external switch/PHY
stack (`sw@92300000` + `gephy128s`) and its 12-partition flash layout, so the
ethernet/switch/PPE driver and the SoC support are **reused unchanged** — only
board facts differ.

The board facts below were read off *this* unit over a UART root shell, and the
datapath was cross-checked against a full Ghidra reverse-engineer of the stock
`/kmodule/*.ko` (see the `re/` tree and `re/spec/idm-datapath.md` in the
RAX3000Z working repo). The mainline PPE driver's architecture matches that RE
(reserved DMA pool, TM-interrupt NAPI, ingress-port-from-descriptor), which is
why reuse is safe.

## What this port changes

| file | change |
|---|---|
| `overlay/target/linux/sanechips/dts/zx279128r-cmcc-rax3000z.dts` | **new** board DTS (forked from RAX1800-Z: 256 MiB DRAM, MT7916) |
| `overlay/target/linux/sanechips/image/Makefile.e2631` | **+** `Device/cmcc_rax3000-z` (MT7916 pkgs, nft-offload) |
| `overlay/target/linux/sanechips/base-files/lib/upgrade/platform.sh` | board-aware verhdr select (rax3000z.verhdr) |
| `overlay/target/linux/sanechips/base-files/lib/upgrade/rax3000z.verhdr` | **new** 480-byte ZTE version header, read from this unit's mtd3 |
| `overlay/target/linux/sanechips/base-files/etc/board.d/02_network` | **+** `cmcc,rax3000-z` → 3 LAN + 1 WAN |
| `seed/rax3000z.config` | **new** lean first-bring-up seed selecting `cmcc_rax3000-z` |

## Board facts (measured on this unit, 2026-11-20)

- SoC ZX279128R, dual Cortex-A9, **256 MiB** DRAM (`/proc/meminfo` 236864 kB)
- 128 MiB parallel SLC NAND, Denali controller (stock `denali,denali-nand-dt`)
- 12 MTD partitions, **byte-for-byte the RAX1800-Z offsets** (from
  `/sys/class/mtd/mtd*/offset`): u-boot@0, kernel0@0x200000, kernel1@0x700000,
  rootfs0@0xc00000, rootfs1@0x2c00000, parameter-tags@0x4c00000, …
- stock U-Boot "CMCRV5.1.0", `bootm 0x440001e0` (480-byte header, magic
  0x99999999, product "12871" ahead of the uImage)
- WiFi MT7916AN (PCIe 14c3:7906 + 790a = one chip, two HIFs)
- wlan partition (mtd9) starts all-0xFF → **no stored caldata**; mt76 falls back
  to on-chip OTP (WiFi still comes up; TX-power calibration may be non-optimal)

## Still a hypothesis — verify on first RAM boot

Same open items the RAX1800-Z port has (the vendor fills these at runtime, so
they can't be read statically):

- the four **MDIO PHY addresses** (assumed 10–13) and the **port→label** map
  (assumed gmac_port0..3 = lan3,lan2,lan1,wan)

Verify once booted:
```
ls /sys/bus/mdio_bus/devices
dmesg | grep -iE 'gmac|phy|zx279128'
ip -br link
```
If the LAN/WAN jacks don't match, fix `&gmac_portN { label }` /
`phy-handle` in the DTS and `02_network`. A board whose wired net is wrong is
still reachable over serial + WiFi, so this never bricks the flash path.

## Building (host is macOS → use Linux)

OpenWrt does not build on macOS. Two options:

1. **GitHub Actions** (easiest): push this repo to your fork and run the
   `firmware.yml` workflow, or add a job that copies `seed/rax3000z.config` to
   `.config`. CI already has the Linux toolchain + the segmented downloader.
2. **Docker on the Mac**: run an `openwrt/sdk`-style Ubuntu container, then:
   ```
   git clone --branch pr-zte-zxhn-h3600 \
     https://github.com/Zlion-Y/openwrt-sanechips-mirror openwrt
   git -C openwrt checkout db8b8d4130457dbe46558a7cea872dd2dae539e6
   ./apply.sh openwrt                       # lays the overlay, hooks Makefile.e2631
   cp seed/rax3000z.config openwrt/.config  # RAX3000-Z seed (NOT bootstrap.sh,
                                            #   which hardcodes the RAX1800-Z seed)
   ( cd openwrt && make defconfig && make -j"$(nproc)" V=s )
   ```
   Outputs land in `openwrt/bin/targets/sanechips/generic/`:
   - `*-cmcc_rax3000-z-initramfs-kernel.bin` ← RAM-boot test image
   - `*-cmcc_rax3000-z-squashfs-sysupgrade.bin` ← the flashable image

## Flashing — RAM-boot FIRST, it cannot brick

You have U-Boot + serial, so prove the build in RAM before writing flash:

1. Interrupt U-Boot (power-cycle, spam Enter), get the `=>` prompt.
2. Load the **initramfs** image to 0x44000000 (tftp if you set up a server, or
   xmodem over the console), then `bootm`. Nothing is written to NAND; a power
   cycle restores stock.
3. At the OpenWrt shell do the PHY/port verification above and confirm WiFi.

Only once RAM boot is clean, flash **slot A** (keeps slot B stock as rollback):

- From the booted OpenWrt (RAM or installed), `sysupgrade` the
  `*-squashfs-sysupgrade.bin`. `platform.sh` prepends this unit's
  `rax3000z.verhdr`, writes rootfs then kernel to the `rootfs`/`kernel` MTD
  labels, and `mtd verify`s both before allowing reboot.
- First NAND install also needs BootPara `BootImageNum=0` (point U-Boot at slot
  A) — a one-time step, same as the RAX1800-Z port.

## Rollback

Slot B (`kernel1-stock` / `rootfs1-stock`, mtd4/mtd5) is marked read-only and is
never a write target — the stock system there is the permanent rollback. U-Boot
code (mtd0 0x0–0x180000) is never written, so the serial rescue console always
survives.
