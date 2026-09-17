# ZTE T5400 — Installation, Backup and Recovery

Accepted persistent design: OpenWrt lives **only** in `mtd17 / rootfs`.
All supported persistent OpenWrt writes target `mtd17 / rootfs` only. The
installation and sysupgrade paths do not modify the second stock slot (`mtd18`),
bootloader partitions, or factory data (see `03-flash-layout.md`).

## Safety classes

| Class | Meaning | Examples |
|---|---|---|
| `RAM-ONLY` | volatile, nothing written to NAND | `tftpboot`, `bootm`, unsaved `setenv`, initramfs boot |
| `READ-ONLY` | reads flash/runtime only | backups, preflight checks, hashes |
| `DESTRUCTIVE FIRST INSTALL` | rewrites `mtd17` only | guarded factory-image write |
| `NORMAL UPDATE` | standard OpenWrt maintenance | `sysupgrade` / `sysupgrade -n` |

The normal path does not require `saveenv` or any
MIBIB/BOOTCONFIG/APPSBL/ART/TRAINING, `mac`, `mtd18`, or FOTA metadata rewrite.

## What you need

- UART access + correctly wired USB-UART adapter (115200 8N1, no flow
  control, **1.8 V logic level — confirmed**; do not use a 3.3 V or 5 V
  adapter directly on these pads)
- Host on Ethernet + a TFTP server stock U-Boot can reach
- OpenWrt initramfs/NAND-recovery image, factory UBI image, sysupgrade image
- Local storage for the stock backup

Stock U-Boot defaults: `192.168.0.1` (router) / `192.168.0.22` (TFTP server)
/ `255.255.255.0` / load addr `0x44000000` / `bootcmd=bootipq`.

## 1. Back up the stock system (READ-ONLY, do this before anything destructive)

Back up at minimum: `mtd17 rootfs`, `mtd18 rootfs_1`, `/proc/mtd`, MTD
geometry/ECC metadata, running FDT, and SHA-256 hashes of all of the above.
Do **not** make raw ART/MAC/bootloader dumps a normal prerequisite — they're
outside the OpenWrt write set and contain per-device data; don't publish raw
stock rootfs images publicly.

Expected sizes: `mtd17` and `mtd18` are each exactly `62914560` bytes. Their
raw hashes don't need to match each other (see the UBI note in
`03-flash-layout.md`) — what matters is that you have a verified image of
each, moved off the router, before the first destructive write.

## 2. Verify the RAM environment (READ-ONLY)

Confirm the router is genuinely running from RAM (`root_type=tmpfs`,
board id `zte,t5400`) and that NAND geometry matches before enabling any
destructive step.

## 3. Inspect the first-install image (READ-ONLY)

Expected write boundary: target `mtd17`, size `62914560`, erasesize
`131072`, writesize `2048`, oobsize `64`, ECC `4/512`. A valid factory image
contains an OpenWrt UBI with volume 0 `kernel`, volume 1 `rootfs`
(SquashFS), volume 2 `rootfs_data` (UBIFS, autoresize on first boot). Always
verify the digest published for the specific artifact you're installing —
don't reuse an old hash from a different build.

## 4. First persistent installation (DESTRUCTIVE — mtd17 only)

```text
stock U-Boot -> RAM boot the NAND installer -> guarded write to mtd17 only
             -> reattach UBI -> verify
```

Post-write verification should show: UBI attached on mtd17, 0 bad PEBs, 0
corrupted PEBs, `kernel`/`rootfs`/`rootfs_data` volumes present, FIT magic
valid, SquashFS magic valid, no ECC failures. Don't substitute a free-form
`ubiformat /dev/mtdX` for the guarded board-specific installer unless you
fully understand the flash map and are deliberately debugging it.

## 5. Verify with stock U-Boot, then let it boot normally

```text
smeminfo
ubi info
ubi check kernel / rootfs / rootfs_data
ubi read 0x44000000 kernel
md.b 0x44000000 4          # expect FIT magic d0 0d fe ed
```

Then just power-cycle and let the unmodified `bootcmd=bootipq` run — no
manual TFTP or U-Boot command should be required from this point on. Root
selector is `root=/dev/ubiblock0_1`, mounting `/rom` (SquashFS ro) and
`/overlay` (UBIFS rw) under an overlayfs `/`. Here `ubiblock0_1` means UBI
device 0, volume 1 inside `mtd17`; it is unrelated to the separate `rootfs_1`
MTD partition (`mtd18`).

## 6. Persistent acceptance checklist

`ubi0` attached to `mtd17` with the 3 expected volumes; `/rom` read-only
SquashFS; `/overlay` read-write UBIFS; 0 NAND bad blocks / ECC failures in
the accepted baseline; Ethernet + both WLAN radios operational. Test with a
real cold boot (power fully removed), not just `reboot` — an unpowered but
still-connected USB-UART adapter has been observed to interfere with
unattended boot on the bench; disconnect it for a clean cold-boot test.

## 7. Normal sysupgrade (NORMAL UPDATE)

Uses the standard OpenWrt NAND helper — no custom T5400 flash algorithm:

```sh
CI_UBIPART="rootfs"
nand_do_upgrade "$1"
```

`sysupgrade -n` = clean `rootfs_data`; plain `sysupgrade` = config-preserving
(restores standard `/etc/config`, not arbitrary overlay files). Never
modifies MIBIB, BOOTCONFIG(1), the persistent U-Boot env, APPSBL, ART,
TRAINING, `mac`, or `mtd18`.

## 8. Recovery after a failed OpenWrt boot

1. Stop repeated blind write attempts.
2. Connect UART, interrupt stock U-Boot.
3. TFTP-boot the known-good RAM recovery/initramfs image.
4. Inspect `mtd17`, UBI and NAND health, read-only.
5. Verify the replacement factory or sysupgrade artifact (digest check).
6. Reinstall only through the guarded `mtd17` path.

The RAM recovery environment is independent of the persistent OpenWrt root,
so this works even if the persistent system is completely broken.

## 9. Return to stock (DESTRUCTIVE RECOVERY, restores mtd17 only)

Hardware-validated end to end: writing a user's own backed-up `mtd17` back
reproduces the stock 5-volume UBI, `mtd18` stays untouched, the router cold-boots
normal stock Linux with Ethernet and Wi-Fi operational, and a clean OpenWrt
installation can then be performed again through the same `mtd17`-only path.

1. Boot the RAM recovery environment via UART/TFTP.
2. Transfer the backed-up stock `mtd17` image to the router and verify its
   SHA-256 against the one recorded at backup time (read-only stage).
3. Run the guarded restore (requires an explicit confirmation token) —
   writes `/dev/mtd17` only, re-verifies the stock 5-volume UBI, and
   compares `mtd18`'s hash before/after to confirm it was untouched.
4. Power off completely, cold-boot stock, confirm the stock kernel/userspace
   and basic Ethernet/WLAN.

Do **not** restore, as part of this normal procedure: ART, `mac`, MIBIB,
BOOTCONFIG, APPSBL, `mtd18`, `fota`, `fota-flag`, or any other boot/factory
partition.
