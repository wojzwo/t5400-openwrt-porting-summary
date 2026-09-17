# ZTE T5400 — OpenWrt port docs

Condensed, filtered documentation for the ZTE T5400 OpenWrt port. Files live
at the repo root.

## Table of contents

1. [Hardware Inventory](01-hardware-inventory.md) — BOM, hardware summary, block topology
2. [GPIO Inventory](02-gpio-inventory.md) — full GPIO map, LEDs, buttons
3. [Flash Layout](03-flash-layout.md) — NAND map, partition table, OpenWrt write boundary
4. [UART & U-Boot](04-uart-uboot.md) — UART wiring, using the stock U-Boot, RAM-boot/TFTP
5. [Ethernet](05-ethernet.md) — RJ45/QCA8337/GEPHY topology, DSA mapping, MAC
6. [Wi-Fi](06-wifi.md) — contract for the 2.4 GHz and 5 GHz radios, BDF/ART/MAC
7. [Installation & Recovery](07-install-recovery.md) — backup, first install, sysupgrade, recovery, return to stock
8. [OpenWrt Porting Overview](08-openwrt-porting.md) — porting principles, nand install → mtd17 → restart flow

`photos/` — board and component photos referenced by the docs above.
`evidence/` — supplementary raw evidence (stock partition table, U-Boot
command/env captures, and stock/OpenWrt forwarding-performance measurements)
backing the condensed docs; not required reading, kept for reproducibility.

## Status

Validated end to end on real hardware: first install, persistent cold boot,
clean and config-preserving sysupgrade, Ethernet (all 4 ports), both Wi-Fi
radios, LEDs/buttons, and a full return-to-stock recovery followed by a clean
OpenWrt reinstall. UART pad order and 1.8 V logic level are also confirmed.

USB support is deliberately deferred. Qualcomm NSS/ECM forwarding offload is
not available upstream for `qualcommax`, so routed throughput under OpenWrt is
lower than the stock firmware's accelerated datapath; measured comparisons are
kept under `evidence/`. Remaining hardware-documentation unknowns are listed in
[01-hardware-inventory.md](01-hardware-inventory.md).
