# ZTE T5400 — Wi-Fi

## Physical RF layout

```text
2.4 GHz  integrated IPQ5018 WLAN   2x2   antennas 2G_0 / 2G_1
5 GHz    external Qualcomm QCN9024 4x4   antennas 5G_0 / 5G_1 / 5G_2 / 5G_3
```

```text
Qualcomm IPQ5018
 ├─ integrated WLAN (AHB/WCSS) ── 2.4 GHz 2x2 ── 2G_0 / 2G_1
 └─ PCIe Gen2 x2 ── Qualcomm QCN9024 ── 5 GHz 4x4 ── 5G_0..5G_3
```

## Final contract at a glance

| Property | Integrated 2.4 GHz | External 5 GHz |
|---|---|---|
| Physical device | IPQ5018 integrated WLAN | Qualcomm QCN9024 |
| Spatial streams | 2x2 | 4x4 |
| Transport | AHB / WCSS | PCIe Gen2 x2 |
| Runtime device | `c000000.wifi` | `0000:01:00.0` |
| PCI ID | — | `17cb:1104` |
| ath11k profile | IPQ5018 hw1.0 | QCN9074 hw1.0 |
| QMI chip ID / board ID | `0` / `255 (0xff)` | `0` / `255 (0xff)` |
| Board variant | `ZTE-T5400` | `ZTE-T5400` |
| Stock BDF source | `bdwlan.b24` | `qcn9000/bdwlan.ba0` |
| BDF size | 0x20000 (131072 B) | 0x20000 (131072 B) |
| ART offset / length | `0x1000` / `0x20000` | `0x26800` / `0x20000` |
| Primary MAC | factory/base (offset 0), via NVMEM | factory/base **+10 at the fourth octet (`mac[3]`)**, via generic NVMEM MAC arithmetic |
| Firmware package | `ath11k-firmware-ipq5018` | `ath11k-firmware-qcn9074` + `kmod-ath11k-pci` |
| OpenWrt BDF source | `board-zte_t5400.ipq5018` | `board-zte_t5400.qcn9074`  |
| Functional state | validated | validated |

Both radios use `qcom,calibration-variant = "ZTE-T5400"` and the same
architectural separation:

```text
firmware          -> normal ath11k firmware interface
board identity     -> T5400-specific BDF (board data file)
RF calibration      -> exact per-unit ART slice
MAC identity         -> NVMEM
```

No final path patches a MAC address into calibration data.

## Naming discipline — do not conflate these

```text
physical 5 GHz chip       QCN9024
PCI identity               17cb:1104
stock software family      qcn9000
Linux / ath11k profile     QCN9074 hw1.0 (and QMI board ID 255 is a runtime
                            constant, not the "b24"/"ba0" stock filename suffix)
```

## OpenWrt target definition

```make
define Device/zte_t5400
  $(call Device/FitImageLzma)
  $(call Device/UbiFit)
  DEVICE_VENDOR := ZTE
  DEVICE_MODEL := T5400
  DEVICE_DTS_CONFIG := config@mp03.1
  SOC := ipq5018
  BLOCKSIZE := 128k
  PAGESIZE := 2048
  IMAGE_SIZE := 61440k
  NAND_SIZE := 256m
  DEVICE_PACKAGES := ath11k-firmware-ipq5018 \
    kmod-ath11k-pci \
    ath11k-firmware-qcn9074 \
    ipq-wifi-zte_t5400
endef
TARGET_DEVICES += zte_t5400
```

This is correct and matches the "Final contract" table above — `ipq-wifi-zte_t5400`
is a single OpenWrt **package** (`package/firmware/ipq-wifi/Makefile`'s
`generate-ipq-wifi-package,zte_t5400,...` macro generates exactly one
package for the device), but that package installs **two distinct BDF
files**, one per radio — the package is shared, the board-data content is
not. Confirm against the "Stock BDF source" row above: `bdwlan.b24` for the
integrated 2.4 GHz radio vs. `qcn9000/bdwlan.ba0` for the external QCN9024
— different source files with different payloads, packaged together only
for OpenWrt install-unit convenience. Four packages are needed in total for
full Wi-Fi function:

```text
ath11k-firmware-ipq5018   integrated 2.4 GHz radio firmware
kmod-ath11k-pci           PCI glue driver — needed because QCN9024 is PCIe-attached
ath11k-firmware-qcn9074   external 5 GHz radio firmware
ipq-wifi-zte_t5400        board-data package (contains a distinct BDF per radio)
```

`IMAGE_SIZE := 61440k` = 60 MiB, matching the `mtd17` write boundary in
`03-flash-layout.md`. `NAND_SIZE := 256m` matches the physical Winbond part.
`DEVICE_DTS_CONFIG := config@mp03.1` is the same FIT config referenced in
`04-uart-uboot.md`.

## Device-tree MAC wiring

```dts
&pcie0 {
  status = "okay";

  perst-gpios = <&tlmm 15 GPIO_ACTIVE_LOW>;

  pcie@0 {
    wifi@0,0 {
      status = "okay";

      /* QCN9024, exposed to ath11k as PCI device 17cb:1104. */
      compatible = "pci17cb,1104";
      reg = <0x00010000 0 0 0 0>;

      qcom,calibration-variant = "ZTE-T5400";

      /*
       * Stock derives the primary chip2 WLAN MAC by
       * incrementing factory MAC fourth octet (`mac[3]`) by 10 modulo 256.
       */
      nvmem-cells = <&macaddr_mac_0 0x0a0000>;
      nvmem-cell-names = "mac-address";
    };
  };
};

&q6v5_wcss {
  firmware-name = "ath11k/IPQ5018/hw1.0/q6_fw.mdt",
      "ath11k/IPQ5018/hw1.0/m3_fw.mdt";
};

&wifi {
  status = "okay";

  qcom,rproc = <&q6v5_wcss>;
  qcom,calibration-variant = "ZTE-T5400";
  qcom,ath11k-fw-memory-mode = <1>;

  /* Stock uses the factory MAC for the primary chip1 WLAN address. */
  nvmem-cells = <&macaddr_mac_0 0>;
  nvmem-cell-names = "mac-address";
};
```

`perst-gpios = <&tlmm 15 ...>` is GPIO15, PCIe reset for the QCN9024 — see
`02-gpio-inventory.md`. `reg = <0x00010000 ...>` matches PCI device
`17cb:1104` on the pcie0 root port.

### How `<&macaddr_mac_0 <offset>>` encodes the fourth-octet offset

`macaddr_mac_0` is a "mac-base" style nvmem provider: the second cell of the
phandle reference is not a normal nvmem argument, it's a **signed 48-bit
add value applied to the base MAC as a big integer**. Because a MAC is
6 bytes (`B0 B1 B2 B3 B4 B5`), adding a value shifted left by 16 bits is aligned
to the fourth octet (`mac[3]`):

```text
0x0a0000  =  0x0a << 16  ->  adds 0x0a (= 10) at mac[3], using normal 48-bit carry
0         =  0            ->  no change (integrated radio keeps the factory MAC as-is)
```

The offset matches the stock ZTE VAP/BSSID allocation on the tested unit. The
stock algorithm, documented from `libzte_wlan.so`'s
`zte_wlan_wifi_multi_bssid_get()`, updates only `bssid[3]` modulo 256:
`bssid[3] = base[3] + ssid_index`, plus an extra `+8` for SSIDs served by the
second (external, chip2) radio. For the QCN9024 primary AP, `2 + 8 = 10`, hence
the `0x0a0000` offset; the integrated radio's primary SSID uses offset `0`.

The generic `mac-base` provider does normal 48-bit arithmetic rather than
stock's isolated-octet wrap. It therefore diverges only if `mac[3] + 10`
crosses `0xff`; the tested unit is not near that boundary.

## Calibration policy

Calibration (ART slice) is strictly per-device and must never be published
or reused across units. The generic "board ID 255" board-data-file payload
is sufficient as BDF; it does not need per-unit BDF regeneration. OpenWrt
policy: firmware, board data (BDF) and RF calibration (ART) are three
separate concerns — never merge a MAC address into the calibration blob.

`wifi_fw` in the stock partition table is a **UBI volume**, not a physical
NAND partition — don't treat it as a separate flash region when reasoning
about the port.

## Integrated 2.4 GHz — key facts

```text
transport            AHB / WCSS
runtime path          c000000.wifi
ath11k identity        ipq5018 hw1.0
boot path              IPQ5018 -> q6v5_wcss remoteproc -> WCSS/Q6 firmware
                        -> ath11k AHB device -> mac80211/cfg80211
calibration            0:ART, offset 0x1000, length 0x20000
OpenWrt cal filename    ath11k/IPQ5018/hw1.0/cal-ahb-c000000.wifi.bin
MAC                    factory/base, via nvmem-cells "mac-address" (offset 0)
```

## External 5 GHz (QCN9024) — key facts

```text
transport             PCIe Gen2 x2, PCI 17cb:1104, PERST on GPIO15
ath11k identity         QCN9074 hw1.0
board selector          bus=pci,qmi-chip-id=0,qmi-board-id=255,variant=ZTE-T5400
calibration              0:ART, offset 0x26800, length 0x20000
MAC                      factory/base +10 at mac[3] via generic NVMEM MAC arithmetic
```

## Functional acceptance

Both radios validated: WCSS remoteproc running, ath11k probed with correct
hw1.0 profile, QMI board ID 0xff, PHY registered, RF/AP operation confirmed
on both bands.

## Performance reference (stock vs. OpenWrt/ath11k)

iperf3 routed-forwarding tests between two separate endpoint machines in
different IP subnets, with the T5400 forwarding traffic between them. The
router itself was not an iperf3 endpoint. `single-*` uses one TCP stream and
`four-*` uses four parallel streams; the same router and test endpoints were
used for both firmware states. OpenWrt numbers below are measured results, not
projections — see `evidence/perf_summary_openwrt_vs_stock.md` for the full
comparison including Ethernet.

| Band | Scenario | Stock recv Mb/s | Stock send Mb/s | OpenWrt recv Mb/s | OpenWrt send Mb/s | OpenWrt retransmits |
|---|---|---:|---:|---:|---:|---:|
| 5 GHz (QCN9024) | single-up | 942.5 | 942.6 | 270.4 | 270.4 | – |
| 5 GHz (QCN9024) | single-down | 839.8 | 839.6 | 266.0 | 266.0 | 42 |
| 5 GHz (QCN9024) | four-up | 922.2 | 922.2 | 285.1 | 285.3 | – |
| 5 GHz (QCN9024) | four-down | 786.2 | 786.4 | 163.7 | 163.7 | 231 |
| 2.4 GHz (integrated) | single-up | 330.9 | 331.0 | 203.1 | 203.1 | – |
| 2.4 GHz (integrated) | single-down | 205.0 | 205.1 | 182.5 | 182.3 | 53 |
| 2.4 GHz (integrated) | four-up | 283.1 | 283.0 | 254.1 | 255.1 | – |
| 2.4 GHz (integrated) | four-down | 209.5 | 209.7 | 143.4 | 143.1 | 399 |

`up` and `down` are the benchmark suite's two opposite routed directions; in
both cases the T5400 forwards traffic between the two endpoint machines.
OpenWrt numbers have no NSS/ECM hardware forwarding offload (not implemented
upstream, same as Ethernet), so the gap vs. stock is expected. Stock 5 GHz
single-down being markedly lower than stock single-up/four-up is a
stock-firmware characteristic observed while capturing the baseline — not
something this port needs to explain or fix.
