# ZTE T5400 — stock vs OpenWrt forwarding performance

iperf3 routed-forwarding tests between two separate endpoint machines in
different IP subnets, with the T5400 forwarding all traffic between them. The
router itself was **not** an iperf3 endpoint. `single-*` uses one TCP stream and
`four-*` uses four parallel streams. The same physical T5400 and test endpoints
were used for both firmware states; each scenario ran for ~40 s including setup.

OpenWrt numbers are measured with **no NSS/ECM hardware forwarding offload**
(not implemented upstream; see `../05-ethernet.md` / `../06-wifi.md`) — the gap below
is expected, not a port regression.

## Consolidated results

Receiver-reported throughput is used for the comparison below; sender and
receiver figures differ only negligibly in these runs. `OpenWrt / stock` shows
the forwarding throughput retained without NSS/ECM acceleration. Full sender
and receiver values remain in the per-suite files listed below.

| Path | Scenario | Stock recv Mb/s | OpenWrt recv Mb/s | OpenWrt / stock | OpenWrt retransmits |
|---|---|---:|---:|---:|---:|
| Ethernet | single-up | 946.7 | 501.6 | 53.0% | - |
| Ethernet | single-down | 946.8 | 383.8 | 40.5% | 0 |
| Ethernet | four-up | 946.9 | 344.3 | 36.4% | - |
| Ethernet | four-down | 946.8 | 316.6 | 33.4% | 84 |
| Wi-Fi 5 GHz (QCN9024) | single-up | 942.5 | 270.4 | 28.7% | - |
| Wi-Fi 5 GHz (QCN9024) | single-down | 839.8 | 266.0 | 31.7% | 42 |
| Wi-Fi 5 GHz (QCN9024) | four-up | 922.2 | 285.1 | 30.9% | - |
| Wi-Fi 5 GHz (QCN9024) | four-down | 786.2 | 163.7 | 20.8% | 231 |
| Wi-Fi 2.4 GHz (integrated) | single-up | 330.9 | 203.1 | 61.4% | - |
| Wi-Fi 2.4 GHz (integrated) | single-down | 205.0 | 182.5 | 89.0% | 53 |
| Wi-Fi 2.4 GHz (integrated) | four-up | 283.1 | 254.1 | 89.8% | - |
| Wi-Fi 2.4 GHz (integrated) | four-down | 209.5 | 143.4 | 68.4% | 399 |

## Raw per-suite data

Individual suite files (`suite_id`, sampler sample counts, rc codes)
this table was built from:

```text
evidence/summary_perf_eth.md    (stock ethernet,  20260911T050928Z-stock-ethernet)
evidence/summary_perf5.md       (stock wifi5,      20260911T053418Z-stock-wifi5)
evidence/summary_perf24.md      (stock wifi24,     20260911T052642Z-stock-wifi24)
evidence/openwrt_perf_eth.md    (openwrt ethernet, 20260911T023103Z-openwrt-ethernet)
evidence/openwrt_perf5.md       (openwrt wifi5,    20260911T035251Z-openwrt-wifi5)
evidence/openwrt_perf24.md      (openwrt wifi24,   20260911T034502Z-openwrt-wifi24)
```

## Notes

- All runs used the same physical T5400, the same two test endpoints, and the
  same iperf3 parameters (30 s duration, 3 s omit, 1 s sampling interval).
- `up` and `down` are the suite labels for the two opposite routed directions;
  the T5400 is a forwarding device in both cases, not an iperf3 endpoint.
- `-` in retransmits means the sender-side retransmit count was not available in
  the collected iperf3 result; a numeric value is the sender-reported TCP
  retransmit count.
- The four-stream OpenWrt Ethernet cases are slower than single-stream while the
  links remain Gigabit. Without NSS/ECM, forwarding is CPU-limited on the two
  Cortex-A53 cores; stock offloads established forwarding flows through its
  Qualcomm accelerated datapath.
- The 2.4 GHz results are less useful for isolating forwarding capacity because
  the client-side wireless link can become the limiting factor before the router
  forwarding path does.
