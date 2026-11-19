# Jumbo frames for node-to-node traffic on the Turing Pi 2

This guide explains how MTU 9000 between the RK1 nodes is enabled by
decoupling the RTL8370MB switch's maximum frame length from the BMC's DSA
conduit MTU, and what limits still apply to the BMC's own traffic.

## Background: why the BMC caps out at MTU 3766

* The DSA *conduit* is the Allwinner T113's EMAC (`dwmac-sun8i` on the
  `stmmac` framework). Its RX path uses single-page skbs, so the kernel caps
  the conduit MTU at **3774** bytes on this 32-bit ARM build.
* DSA subtracts the 8-byte Realtek `rtl8_4` CPU tag when validating switch
  port MTUs: **3774 − 8 = 3766** is the hard cap for `node1`…`node4`, `ge0`
  and `ge1` *on the BMC*.
* Upstream, the switch driver tracks the largest user-port MTU in the
  switch's single global maximum RX frame length register
  (`RTL8365MB_CFG0_MAX_LEN_REG`), so hardware-switched frames were also
  limited to what the conduit allows — even though the RTL8370MB fabric can
  forward frames up to 0x3FFF (16383) bytes.

## What the patch does

`tp2bmc/patches/linux/switch-max-frame-len.patch` changes
`rtl8365mb_port_change_mtu()` to always program the switch's global maximum
RX frame length to its hardware maximum (0x3FFF) instead of tracking the DSA
MTU. Node-to-node frames are switched entirely in the RTL8370MB ASIC and
never cross the conduit, so they are no longer constrained by the EMAC's
3774-byte limit.

## Node configuration

Nothing needs to change on the BMC at runtime. On each RK1 node (the RK3588
GMAC supports MTU 9000):

```sh
ip link set eth0 mtu 9000
```

Verify with a no-fragment jumbo ping between two nodes (8972 = 9000 − 20
IPv4 header − 8 ICMP header):

```sh
ping -M do -s 8972 <other-node-ip>
```

## Caveats

* **Jumbo frames are not a universal win.** The RTL8370MB is a
  store-and-forward switch, so each hop must receive a full frame before
  forwarding it: at 1 GbE a 9000-byte frame adds ~72 µs of serialization
  delay per hop (vs ~12 µs at 1500), and large frames on the wire also delay
  small latency-sensitive packets behind them (head-of-line blocking).
  Bulk-throughput workloads benefit from a larger MTU; chatty,
  latency-sensitive workloads (e.g. small-message sync protocols) can get
  *worse*. Measure with your actual workload — testing on this cluster showed
  a sync-heavy workload performed better at MTU 1500 than at larger MTUs.
  Running the nodes at MTU 1500 is perfectly fine: the switch patch merely
  removes an artificial cap and has no effect on standard-sized traffic.
* **BMC ↔ node traffic is still capped at MTU 3766** — the conduit limit is
  unchanged. Keep the BMC-side interfaces at 3766 or below; this is
  management traffic and does not need jumbo frames.
* **Mixed-MTU networks:** if the uplinks (`ge0`/`ge1`) join a LAN that does
  not carry jumbo frames end-to-end, make sure node traffic leaving the
  cluster uses a path MTU ≤ 1500 (PMTUD or per-route `mtu lock`), as usual
  for jumbo deployments.
* **Flooded jumbo frames** (broadcast/unknown-unicast/multicast) also reach
  the CPU port and are dropped by the EMAC as oversized. This is harmless
  but shows up as RX length errors on the conduit interface.
