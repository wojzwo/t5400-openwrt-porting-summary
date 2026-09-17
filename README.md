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
command/env captures, stock forwarding-performance numbers) backing the
condensed docs; not required reading, kept for reproducibility.

## Status

Validated end to end: first install, persistent boot, sysupgrade (clean and
config-preserving), Ethernet (all 4 ports), both Wi-Fi radios, LEDs/buttons.
Open items: a full return-to-stock drill with confirmed stock
Ethernet/Wi-Fi operation before reinstalling OpenWrt, and the hardware
unknowns listed in [01-hardware-inventory.md](01-hardware-inventory.md)
(power/regulator BOM, UART voltage). USB support itself is deferred, not unknown — see the USB note in `01-hardware-inventory.md`.
