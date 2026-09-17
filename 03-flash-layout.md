# ZTE T5400 — Flash Layout

256 MiB Winbond W25N02JWZEIF SPI-NAND, 128 KiB erase block, 2048+64 B page,
4-bit/512 B ECC via QPIC controller `0x079b0000` (8-bit bus).

## High-level map

```text
0x00000000  mtd0  SBL1            0.5 MiB
0x00080000  mtd1  MIBIB           0.5 MiB
0x00100000  mtd2  BOOTCONFIG      0.25 MiB
0x00140000  mtd3  BOOTCONFIG1     0.25 MiB
0x00180000  mtd4  QSEE            1 MiB
0x00280000  mtd5  QSEE_1          1 MiB
0x00380000  mtd6  DEVCFG          0.25 MiB
0x003c0000  mtd7  DEVCFG_1        0.25 MiB
0x00400000  mtd8  CDT             0.25 MiB
0x00440000  mtd9  CDT_1           0.25 MiB
0x00480000  mtd10 APPSBLENV       0.5 MiB
0x00500000  mtd11 APPSBL          1.375 MiB
0x00660000  mtd12 APPSBL_1        1.375 MiB
0x007c0000  mtd13 ART             1 MiB
0x008c0000  mtd14 TRAINING        0.5 MiB
0x00940000  mtd15 fota-flag       0.625 MiB
0x009e0000  mtd16 mac             0.5 MiB
0x00a60000  mtd17 rootfs          60 MiB   <- OpenWrt write target
0x04660000  mtd18 rootfs_1        60 MiB   <- preserved stock slot
0x08260000  mtd19 openwrt_data    25 MiB
0x09b60000  mtd20 cfg-param       15 MiB
0x0aa60000  mtd21 log             15 MiB
0x0b960000  mtd22 oops            0.625 MiB
0x0ba00000  mtd23 fota            65 MiB (UBI)
0x0fb00000  mtd24 reserved        5 MiB
0x10000000  end of 256 MiB NAND
```

## Partition table with policy

| MTD | Name | Offset | Size | Role | OpenWrt policy |
|---:|---|---:|---:|---|---|
| 0 | `0:SBL1` | 0x00000000 | 0.5 MiB | boot-stage payload | PRESERVE |
| 1 | `0:MIBIB` | 0x00080000 | 0.5 MiB | NAND partition metadata | PRESERVE |
| 2 | `0:BOOTCONFIG` | 0x00100000 | 0.25 MiB | boot config slot | PRESERVE |
| 3 | `0:BOOTCONFIG1` | 0x00140000 | 0.25 MiB | redundant boot config | PRESERVE |
| 4 | `0:QSEE` | 0x00180000 | 1 MiB | secure exec env | PRESERVE |
| 5 | `0:QSEE_1` | 0x00280000 | 1 MiB | redundant QSEE | PRESERVE |
| 6 | `0:DEVCFG` | 0x00380000 | 0.25 MiB | device config | PRESERVE |
| 7 | `0:DEVCFG_1` | 0x003c0000 | 0.25 MiB | redundant DEVCFG | PRESERVE |
| 8 | `0:CDT` | 0x00400000 | 0.25 MiB | board config data | PRESERVE |
| 9 | `0:CDT_1` | 0x00440000 | 0.25 MiB | redundant CDT | PRESERVE |
| 10 | `0:APPSBLENV` | 0x00480000 | 0.5 MiB | U-Boot env | PRESERVE, `saveenv` not needed |
| 11 | `0:APPSBL` | 0x00500000 | 1.375 MiB | primary U-Boot | PRESERVE |
| 12 | `0:APPSBL_1` | 0x00660000 | 1.375 MiB | redundant U-Boot | PRESERVE |
| 13 | `0:ART` | 0x007c0000 | 1 MiB | per-device WLAN calibration | READ-ONLY / MUST PRESERVE |
| 14 | `0:TRAINING` | 0x008c0000 | 0.5 MiB | vendor training data | READ-ONLY / MUST PRESERVE |
| 15 | `fota-flag` | 0x00940000 | 0.625 MiB | FOTA state/flag | PRESERVE |
| 16 | `mac` | 0x009e0000 | 0.5 MiB | raw 6-byte base MAC @ offset 0, device-unique | READ-ONLY / MUST PRESERVE |
| 17 | `rootfs` | 0x00a60000 | 60 MiB | stock UBI slot / **final OpenWrt UBI target** | WRITABLE — only normal write target |
| 18 | `rootfs_1` | 0x04660000 | 60 MiB | second stock UBI slot | PRESERVE, outside normal write path |
| 19 | `openwrt_data` | 0x08260000 | 25 MiB | historically used for isolated NAND/UBI validation | PRESERVE |
| 20 | `cfg-param` | 0x09b60000 | 15 MiB | app/config data | PRESERVE |
| 21 | `log` | 0x0aa60000 | 15 MiB | log region | PRESERVE |
| 22 | `oops` | 0x0b960000 | 0.625 MiB | crash/oops region | PRESERVE |
| 23 | `fota` | 0x0ba00000 | 65 MiB | FOTA UBI storage | PRESERVE |
| 24 | `reserved` | 0x0fb00000 | 5 MiB | vendor reserved | PRESERVE |

## Write boundary

```text
mtd0 .. mtd16     PRESERVE (bootloader, secure boot, factory data)
mtd17 rootfs      NORMAL OPENWRT WRITE TARGET
mtd18 .. mtd24    PRESERVE
```

Normal install/sysupgrade never touches: `SBL1, MIBIB, BOOTCONFIG(1), QSEE(_1),
DEVCFG(_1), CDT(_1), APPSBLENV, APPSBL(_1), ART, TRAINING, fota-flag, mac,
rootfs_1, openwrt_data, cfg-param, log, oops, fota, reserved`.

## Stock vs. OpenWrt use of mtd17

```text
stock mtd17 UBI:    kernel / wifi_fw / bt_fw / ubi_rootfs / web
OpenWrt mtd17 UBI:  kernel / rootfs / rootfs_data
```

Naming trap: the Linux root device `/dev/ubiblock0_1` is UBI device 0,
volume 1 (`rootfs`) inside `mtd17`. It is **not** the separate NAND partition
named `rootfs_1` (`mtd18`).

The captured stock `rootfs` and `rootfs_1` had different raw-image hashes but
matching hashes per logical volume — i.e. same firmware payload, not
byte-identical raw UBI images. A recovery/restore procedure must restore a
*known-good image as an image*; never try to reconstruct physical UBI PEB
placement from an old capture.

`mtd23/fota` is a separate stock UBI container, not part of the root-slot UBI.
The full vendor FOTA/rollback state machine is not established — do not touch
`fota`, `fota-flag`, or boot metadata just because their names suggest a
recovery role.
