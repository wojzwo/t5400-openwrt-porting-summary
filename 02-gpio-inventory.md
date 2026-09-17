# ZTE T5400 — GPIO Inventory

Every GPIO with a currently established board-level function.

| GPIO | Function | Role | Polarity / state | Consumer |
|---:|---|---|---|---|
| 1 | white status LED | output | active-high | status LED |
| 4 | QPIC/QSPI data 3 | bidir | — | NAND |
| 5 | QPIC/QSPI data 2 | bidir | — | NAND |
| 6 | QPIC/QSPI data 1 | bidir | — | NAND |
| 7 | QPIC/QSPI data 0 | bidir | — | NAND |
| 8 | QPIC/QSPI CS | output | active-low (bus level) | NAND |
| 9 | QPIC/QSPI CLK | output | clock | NAND |
| 15 | QCN9024 PCIe PERST | output reset | active-low | external WLAN |
| 20 | primary UART RX | input | UART, pull-up | serial console |
| 21 | primary UART TX | output | UART, bias disabled | serial console |
| 32 | RESET button | input | active-low, 60 ms debounce | `KEY_RESTART` |
| 36 | MDIO1 MDC | output clock | — | QCA8337 management |
| 37 | MDIO1 MDIO | bidir | — | QCA8337 management |
| 38 | WPS button | input | active-low, 60 ms debounce | `KEY_WPS_BUTTON` |
| 39 | QCA8337 reset | output reset | active-low | QCA8337 |
| 44 | red status LED | output | active-high | status LED |

USB-C GPIOs (Type-C CC/orientation/VBUS control) are intentionally omitted
here — USB support is deferred in this port (see `08-openwrt-porting.md`).

## LEDs and buttons

```text
white LED   GPIO1   active-high
red LED     GPIO44  active-high
RESET       GPIO32  active-low  -> KEY_RESTART
WPS         GPIO38  active-low  -> KEY_WPS_BUTTON
```

Current OpenWrt diagnostic aliases:

```text
led-boot      -> red
led-failsafe  -> red
led-running   -> white
led-upgrade   -> red
```

The white LED is also used in software policy for `eth0` carrier state — that
is a runtime policy choice layered on top of the physical LED wiring, not a
hardware fact.

Stock front-LED behavior (reference only, not reproduced 1:1 in OpenWrt):

| Device state | Stock LED behavior |
|---|---|
| booting | red blinking |
| factory reset in progress | red blinking |
| WAN disconnected | red solid |
| WPS activation | white blinking |
| mesh formation | white blinking |
| WAN connected | white solid |
| not powered | off |
