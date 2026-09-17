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

The stock firmware stores a single factory/base MAC address in the `mac`
partition and derives the Ethernet interface addresses from it.

Stock mapping:

```text
br-lan / bridge   factory base
eth1 / LAN        factory base with byte 3 decremented by 1
eth0 / WAN        factory base with byte 3 decremented by 2
```
Using `B0:B1:B2:B3:B4:B5` for the factory base:

```text
br-lan   B0:B1:B2:B3:B4:B5
eth1     B0:B1:B2:(B3 - 1):B4:B5
eth0     B0:B1:B2:(B3 - 2):B4:B5
```
The stock implementation modifies only the fourth octet (mac[3]), with
8-bit wraparound. This behavior was confirmed from the stock userspace and
cross-checked against live interface addresses.

The stock MAC-reading path returns the same six-byte factory value for the
Ethernet-related lookups. The LAN/WAN differentiation is applied later by
stock userspace.

OpenWrt represents the Ethernet addresses through the generic `mac-base`
NVMEM mechanism:
```text
eth1 / GMAC1   derived from the factory base
eth0 / GMAC0   derived from the factory base
LAN / br-lan   factory/base MAC, set by standard board initialization
label MAC      factory/base MAC
```
No custom OpenWrt MAC-derivation or lower-netdev rewrite code is required.

mac-base uses normal 48-bit MAC arithmetic rather than the stock
byte-3-only operation. For the validated T5400 unit both methods produce the
same addresses. They can differ only if the subtraction crosses the byte-3
underflow boundary.

#### Reproducing the stock analysis

To independently reproduce the stock MAC-address analysis, inspect these
binaries from the stock root filesystem:

```text
/usr/bin/router_msg
/usr/bin/nv_io
/usr/lib/libzte_router.so
/usr/lib/libzte_encrypt.so
```
`router_msg` contains the interface-specific MAC derivation logic, while
`nv_io` and the associated ZTE libraries implement access to the stored
factory MAC data.

## Performance reference (stock NSS/ECM vs. plain upstream DSA)

iperf3 routed-forwarding tests between two separate endpoint machines in
different IP subnets, with the T5400 forwarding the traffic between them. The
router itself was not an iperf3 endpoint. `single-*` uses one TCP stream and
`four-*` uses four parallel streams; each scenario ran for ~40 s including
setup. The same router and test endpoints were used for both firmware states.
See `evidence/perf_summary_openwrt_vs_stock.md` for the full comparison including
both Wi-Fi bands.

| Scenario | Stock recv Mb/s | Stock send Mb/s | OpenWrt recv Mb/s | OpenWrt send Mb/s | OpenWrt retransmits |
|---|---:|---:|---:|---:|---:|
| single-up | 946.7 | 946.7 | 501.6 | 501.6 | – |
| single-down | 946.8 | 947.0 | 383.8 | 383.7 | 0 |
| four-up | 946.9 | 946.9 | 344.3 | 344.2 | – |
| four-down | 946.8 | 947.3 | 316.6 | 316.7 | 84 |

Stock NSS/ECM routes at close to line rate using a hardware/firmware
forwarding accelerator; that's a vendor-stack feature, not a correctness
requirement for the upstream port, and reproducing it is out of scope here.
Without NSS/ECM, the upstream Linux forwarding path is CPU-limited on the two
Cortex-A53 cores; the four-stream cases are slower than the single-stream cases
while the physical links remain Gigabit. These are measured OpenWrt numbers,
not projections.

## Validation summary

| Capability | State |
|---|---|
| RJ45 1/2/3 → QCA8337 → lan1/2/3 | PASS |
| RJ45 4 → GEPHY → eth0 | PASS |
| QCA8337 rev.2 identity / CPU port 6 | PASS |
| SGMII / UNIPHY0 / GMAC1 | PASS |
| Ethernet factory MAC/identity derivation | PASS |
| Old "No xPCS found" / UNIPHY ref-clock issue | CLOSED — don't treat as open unless it reproduces |
