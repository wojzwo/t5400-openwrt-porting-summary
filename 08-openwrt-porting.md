# ZTE T5400 — OpenWrt Porting Overview

## Porting principles

1. **Preserve hardware truth, not vendor software shape.** Document what the
   silicon actually does, not the QSDK abstraction over it.
2. **Prefer standard upstream subsystems.** qca8k/DSA over vendor SSDK,
   standard ath11k instead of the vendor CNSS/QSDK WLAN stack, and standard
   `nand_do_upgrade` over a custom flashing tool.
3. **Keep the board description minimal.** Only enable what's physically
   populated; don't carry over every alternate-BOM node from the stock DTS.
4. **Keep identity separate from calibration.** MAC (NVMEM) / board data
   (BDF) / RF calibration (ART) are three distinct concerns — never merge
   them (see `06-wifi.md`).
5. **Preserve recovery before any destructive installation.** No first
   install without a verified off-router stock backup (see
   `07-install-recovery.md`).

## The short version: how the port actually gets onto the device

```text
1. RAM-boot the OpenWrt NAND installer over stock U-Boot (TFTP, RAM-only)
2. Guarded write of the OpenWrt factory UBI image to mtd17 — ONLY mtd17
3. Reattach + verify the UBI (kernel / rootfs / rootfs_data volumes present)
4. Power-cycle / restart
5. Unmodified stock bootcmd=bootipq boots the new OpenWrt UBI, no changes
   to U-Boot, MIBIB, BOOTCONFIG or any other partition
```

Everything else in the flash map (`03-flash-layout.md`) is left exactly as
stock shipped it. The whole port fits inside the pre-existing 60 MiB
`mtd17` slot; nothing about the NAND partition table itself is redesigned.

## NAND image / UBI contract

```text
mtd17 rootfs (60 MiB)
└── UBI
    ├── volume 0  kernel       FIT kernel + FDT
    ├── volume 1  rootfs       SquashFS
    └── volume 2  rootfs_data  UBIFS, autoresize on first boot
```

vs. the stock 5-volume layout it replaces:

```text
stock mtd17 UBI:    kernel / wifi_fw / bt_fw / ubi_rootfs / web
OpenWrt mtd17 UBI:  kernel / rootfs / rootfs_data
```

Exact LEB counts vary by build (e.g. 36/176/224 or 36/180/220 observed) —
the stable contract is the partition + volume IDs/names, not a fixed split.
`mtd18` (`rootfs_1`, the second stock slot) is deliberately left outside the
normal OpenWrt write path — no A/B slot logic is reproduced.

## Persistent boot chain (unchanged stock bootloader policy)

```text
stock U-Boot bootipq
  -> MIBIB rootfs / mtd17
  -> OpenWrt UBI volume 0: kernel (FIT config@mp03.1)
  -> Linux
  -> OpenWrt UBI volume 1: rootfs      -> /dev/ubiblock0_1 -> /rom (SquashFS ro)
  -> OpenWrt UBI volume 2: rootfs_data -> /overlay (UBIFS rw)
  -> overlayfs /
```

Validated across: first install → cold boot → `sysupgrade -n` → cold boot →
config-preserving `sysupgrade` → true post-upgrade power-cycle boot. No
persistent U-Boot environment, MIBIB, or BOOTCONFIG changes are required.
`/dev/ubiblock0_1` is UBI device 0, volume 1 inside `mtd17`; it must not be
confused with the separate `rootfs_1` partition (`mtd18`).

## Sysupgrade (after the first install)

```sh
zte,t5400)
    CI_UBIPART="rootfs"
    nand_do_upgrade "$1"
    ;;
```

Standard OpenWrt NAND helper, no custom T5400 flashing code. Recreates the
3 UBI volumes and handles normal config backup/restore.

## Protected storage boundary (same for install and every sysupgrade)

Never rewritten by the normal path: `SBL1, MIBIB, BOOTCONFIG(1), QSEE(_1),
DEVCFG(_1), CDT(_1), APPSBLENV, APPSBL(_1), ART, TRAINING, fota-flag, mac,
rootfs_1 (mtd18), openwrt_data, cfg-param, log, oops, fota, reserved`.

## Subsystem summary (cross-reference)

| Subsystem | Upstream model used | Details |
|---|---|---|
| Ethernet | qca8k / DSA, standard `eth0` WAN | `05-ethernet.md` |
| Wi-Fi | ath11k (AHB + PCIe), NVMEM MAC, per-radio ART/BDF | `06-wifi.md` |
| Storage | standard `nand_do_upgrade`, single `mtd17` UBI target | `03-flash-layout.md` |
| Boot | unmodified stock `bootipq`, standard FIT | `04-uart-uboot.md` |
| Recovery | RAM-boot installer, guarded `mtd17`-only writes | `07-install-recovery.md` |

## Validation state

Functionally validated end to end on real hardware: first install, persistent
cold boot, config-preserving and clean sysupgrade, Ethernet (all 4 ports), both
WLAN radios, LEDs/buttons, and a full return-to-stock recovery followed by a
clean OpenWrt reinstall. UART pad order and 1.8 V logic level are confirmed.

USB remains a deliberate deferral. Qualcomm NSS/ECM forwarding offload is not
available upstream for `qualcommax`; the measured performance gap to stock is
therefore expected and documented in `evidence/perf_summary_openwrt_vs_stock.md`.
Remaining hardware-documentation unknowns are listed in
`01-hardware-inventory.md`.
