# ZTE T5400 — Ethernet

## Current state: functionally complete, all 4 ports validated

```text
Physical RJ45 sockets       4x Gigabit Ethernet
External switch              Qualcomm Atheros QCA8337-AL3C rev.2 (runtime ID 0x1302)
Standalone PHY path          IPQ5018 internal GEPHY
QCA8337 CPU port             port 6
QCA8337 CPU link             SGMII -> UNIPHY0 -> GMAC1 (25 MHz ref in validated path)
Standalone CPU path          internal GEPHY -> GMAC0
QCA8337 reset                GPIO39, active-low
MDIO1 MDC / MDIO             GPIO36 / GPIO37
OpenWrt switch model          qca8k / DSA
OpenWrt switch user ports     lan1, lan2, lan3
OpenWrt standalone port       eth0
Default OpenWrt LAN           lan1 + lan2 + lan3 -> br-lan
Default OpenWrt WAN           eth0
```

## Physical mapping (validated by cable-move + stock/OpenWrt cross-check)

| Enclosure | MDIO bus | PHY addr | Switch port | CPU-side path | OpenWrt name |
|---|---|---:|---|---|---|
| RJ45 1 | MDIO1 | 1 | QCA8337 port 2 | port 6 → SGMII → UNIPHY0 → GMAC1 | `lan1` |
| RJ45 2 | MDIO1 | 2 | QCA8337 port 3 | port 6 → SGMII → UNIPHY0 → GMAC1 | `lan2` |
| RJ45 3 | MDIO1 | 3 | QCA8337 port 4 | port 6 → SGMII → UNIPHY0 → GMAC1 | `lan3` |
| RJ45 4 | MDIO0 | 7 | n/a (standalone) | internal GEPHY → GMAC0 | `eth0` |

The router does **not** place all four RJ45 sockets behind the QCA8337 — two
independent CPU-side datapaths exist (GMAC0 direct, GMAC1 via switch). RJ45 4
being the default WAN is a software policy choice, not a hardware
restriction — it's a normal Gigabit port and can be bridged with the others.

## Numbering domains — never collapse these into one another

```text
enclosure socket          RJ45 1..4
IPQ5018 Ethernet MAC       GMAC0 / GMAC1
Linux lower netdev         eth0 / eth1
MDIO controller             MDIO0 / MDIO1
MDIO PHY address            0..7
QCA8337 switch port         0..6
OpenWrt DSA user device      lan1 / lan2 / lan3
```

## Final OpenWrt representation

The port intentionally does **not** reproduce the vendor QSDK Ethernet
lifecycle (two lower GMAC netdevs + SSDK VLAN programming + optional
NSS/ECM acceleration). Instead it uses the standard upstream DSA model:

```text
RJ45 1..3 -> QCA8337 ports 2/3/4 -> CPU port 6 -> SGMII/UNIPHY0 -> GMAC1/eth1
RJ45 4    -> internal GEPHY -> GMAC0/eth0
```

qca8k/DSA exposes `lan1`, `lan2`, `lan3` behind `eth1`. `eth0` is used
directly as the default WAN — there's no invented DSA `wan@eth0` device. No
vendor lower-netdev MAC rewrite or custom post-`config_generate` bridge hook
is used.

## MAC / identity

```text
eth1 / GMAC1     derived from DTS/NVMEM base
eth0 / GMAC0     derived from DTS/NVMEM base
LAN / br-lan     factory/base MAC, via standard board init (ucidef_set_interface_macaddr "lan")
label identity   factory/base MAC, via standard board metadata
```

No stock lower-netdev MAC rewrite and no custom bridge-MAC hook are needed —
this is handled by standard OpenWrt board initialization only.

### Why "derived from base" looks odd — it's not a whole-MAC `+N`

Stock does **not** apply a conventional whole-48-bit MAC offset (like
`mac-address-increment = <-1>`). Only **byte index 3** (the 4th octet) is
touched, and LAN vs. WAN get *different* small integer offsets on that one
byte, wrapping mod 256. Using generic bytes `B0:B1:B2:B3:B4:B5` for the
factory base (see `01-hardware-inventory.md` — real per-device MACs are
never reproduced in these docs):

```text
br-lan / base   B0:B1:B2: B3     :B4:B5   (unchanged)
eth1 / LAN      B0:B1:B2:(B3 - 1):B4:B5
eth0 / WAN      B0:B1:B2:(B3 - 2):B4:B5
```

This isn't guessed from behavior — it's read directly out of the stock
`router_msg` binary. Relevant evidence (full reverse-engineering trail kept
outside the condensed docs — see the project's `zte-t5400-stock-ethernet-mac-derivation.md`
and `zte-t5400-nv-io-mac-storage-analysis.md` for the complete walkthrough):

```text
binary    dump/partitions/extracted/mtd17_rootfs/filesystems/ubi_rootfs/usr/bin/router_msg
format    ELF32 LSB, ARM EABI5, musl-linked, stripped
disasm    llvm-objdump -d --triple=armv7-linux-gnueabi \
            --start-address=0x113b4 --stop-address=0x11710 <binary>
          (plain GNU objdump on the host couldn't decode the ARM machine
          code at all — needed llvm-objdump for this)
```

Key symbol/string offsets inside that binary:

```text
0x0556  lib_read_mac_from_flash   (reads the 6-byte base MAC)
0x1974  set_hw_addr               (the function that does the byte-3 math)
0x19cd  "bridge"
0x19d4  "lan"
0x19d8  "lan_wan"
```

The actual transform, once the selector string is matched (`bridge` / `lan`
/ `lan_wan`), boils down to these lines of the `lan` (LAN/eth1) case:

```asm
115c4: 05dd3027   ldrbeq  r3, [sp, #0x27]   ; load base MAC byte 3
115c8: 02433001   subeq   r3, r3, #1        ; byte3 -= 1  (8-bit wrap)
115f0: e5cd3027   strb    r3, [sp, #0x27]   ; store back into the result
```

`lan_wan` (eth0/WAN) is the same shape with `#2` instead of `#1`. `bridge`
skips the subtraction entirely — `br-lan` gets the untouched base. The
result is then formatted and applied with `ifconfig <iface> hw ether <mac>`
via a plain `system()` call — this isn't just a log message, it's the code
that actually sets the interface address.

Separately, the base MAC itself comes from a `nv_io` CLI/library
(`lib_read_mac_from_flash`) that reads `/dev/mtd16`. Reverse-engineering the
read path found it **ignores the `eth0`/`eth1`/`wifi` index argument
entirely** — all three logical reads return the same 6 bytes at offset 0 of
the first good eraseblock. In other words, `nv_io` itself doesn't produce
different addresses per interface; all of the LAN/WAN differentiation
happens later, in `router_msg`'s byte-3 subtraction.

We deliberately don't reproduce the full disassembly/decompile here for
simplicity — the two files named above carry the complete instruction-level
walkthrough, string tables, and cross-checks (including live ARP
confirmation) if this ever needs to be re-derived or ported to a different
captured unit.

## Performance reference (stock NSS/ECM vs. plain upstream DSA)

Stock NSS/ECM can route Gigabit at close to line rate (~946.7–946.9 Mbit/s)
with materially lower CPU cost, but that's a vendor-stack performance
feature, not a correctness requirement for the upstream port. Measured
upstream numbers: ~800 Mbit/s per physical socket bidirectional, up to
~937 Mbit/s through the full QCA8337 → GMAC1 → bridge → GMAC0 path.

## Validation summary

| Capability | State |
|---|---|
| RJ45 1/2/3 → QCA8337 → lan1/2/3 | PASS |
| RJ45 4 → GEPHY → eth0 | PASS |
| QCA8337 rev.2 identity / CPU port 6 | PASS |
| SGMII / UNIPHY0 / GMAC1 | PASS |
| Ethernet factory MAC/identity derivation | PASS |
| Old "No xPCS found" / UNIPHY ref-clock issue | CLOSED — don't treat as open unless it reproduces |
