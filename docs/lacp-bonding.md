# LACP bonding (802.3ad) on the Turing Pi 2 uplinks

This guide covers aggregating both external NICs (`ge0` + `ge1`) into a
hardware-offloaded LACP bond, choosing the right transmit hash policy, and —
critically — configuring **multi-connection transports** on the nodes so the
2 Gb aggregate is actually realized under real workloads.

## What you get (and what you don't)

* Each node has a single internal 1 GbE PHY into the RTL8370MB switch:
  **a single node is always capped at 1 Gb/s**. No aggregation scheme changes
  that.
* With the bond, both uplinks carry traffic simultaneously (instead of STP
  blocking one), giving the four nodes a shared **2 Gb/s aggregate**.
* Distribution is **per-flow hashing**, not utilization-aware load balancing.
  A single TCP connection always rides exactly one physical link. Getting an
  even split across the two links therefore depends on having *many* flows —
  which is what the multi-connection transport section below is about.
* Link failure is handled seamlessly: flows re-hash onto the surviving link
  within LACP timeout (~1 s with `lacp-rate fast`).

## Prerequisites

1. Firmware built from this tree with:
   * `tp2bmc/patches/linux/switch-lag-support.patch` — DSA LAG offload for
     the RTL8365MB/RTL8370MB driver (keeps the data path in the switch ASIC;
     without it a bond would funnel all node traffic through the 1 Gb CPU
     port).
   * `CONFIG_BONDING=y` in `tp2bmc/board/tp2bmc/linux_defconfig` — the
     kernel bonding driver runs the 802.3ad LACP state machine; LACPDUs are
     trapped to the CPU by the driver while data frames stay in hardware.
2. A managed switch on the other end with **dynamic 802.3ad LACP** support
   (e.g. any UniFi switch: Port Manager → select both ports → Aggregate).
3. The userspace pieces are already in the stock firmware: ifupdown-ng
   (with the `bond` executor) and full iproute2.

Build the firmware as usual:

```sh
./scripts/git configure
./scripts/git build
```

Flash via the web UI OTA (`.tpu`) or SD recovery image.

## BMC configuration

A ready-made configuration ships in the image at
`/etc/network/interfaces.lacp-example`.

> **ORDER MATTERS — configure the switch LAG first.**
> An 802.3ad bond only carriers up once an LACP partner answers on the
> other end. If you activate this configuration while the upstream switch
> ports are still plain (or only one cable is connected, or the LAG is
> disabled), `bond0` stays **NO-CARRIER**, `br0` gets no uplink, and the
> BMC is unreachable over the network. That is normal 802.3ad behavior —
> it is not evidence that the bond is broken. Always create the
> aggregation group on the switch *before* switching the BMC config, and
> keep **both** cables connected.

To activate it:

```sh
cp /etc/network/interfaces /etc/network/interfaces.bak
cp /etc/network/interfaces.lacp-example /etc/network/interfaces
reboot
```

### Network failsafe (automatic revert)

The firmware ships a watchdog (`/etc/init.d/S41netfailsafe`) that arms
itself at boot whenever `/etc/network/interfaces` differs from the firmware
default. If the BMC has not obtained **any IPv4 address within 5 minutes**,
it:

1. writes a diagnostic snapshot (links, addresses, `/proc/net/bonding/*`,
   `dmesg`, the failing config) to `/etc/network/failsafe-diag.log`
   (persisted across reboots),
2. moves the failing config to `/etc/network/interfaces.failed`,
3. restores `/etc/network/interfaces.bak` — or the pristine firmware
   default if no backup exists — and reboots.

So a bad network config can cost you at most ~6 minutes of downtime, never
the board. After a revert, fetch `/etc/network/failsafe-diag.log` to see
why the bond did not come up. Opt out with
`touch /root/.disable_net_failsafe`; tune the timeout via
`TIMEOUT=<seconds>` in `/etc/default/netfailsafe`.

The relevant part:

```
auto bond0
iface bond0
  use bond
  bond-members ge0 ge1
  bond-mode 802.3ad
  bond-xmit-hash-policy layer3+4
  bond-lacp-rate fast
  bond-miimon 100
  bond-min-links 1

auto br0
iface br0 inet dhcp
  bridge-ports node1 node2 node3 node4 bond0
  ...
```

Notes:

* `br0` now bridges the four node ports and `bond0` — `ge0`/`ge1` must *not*
  appear in `bridge-ports` anymore; they are enslaved by the bond.
* The boot-time migration script (`S00dsa`) leaves this file alone because
  `iface br0` and `bridge-ports` are both present.
* The STP `post-up` lines used to guard against the dual-uplink loop are no
  longer needed: the bond is a single logical port, so no loop can form. They
  are harmless if you keep them.
* Do **not** combine this with VLAN pinning of `ge0`/`ge1` on the same ports —
  pick one uplink scheme per deployment. (VLAN filtering on the *node* ports
  for tenant isolation remains compatible.)
* The hardware offloads **one** LAG (the two trunk groups share a single
  hash-mapping table). A second bond would silently fall back to software
  forwarding — don't create one.

### Switch side (UniFi example)

1. UniFi Network → Ports → select the two ports the BMC uplinks are plugged
   into → **Aggregate** (this creates a dynamic 802.3ad LAG; UniFi does not
   do static LAGs, which is what you want here).
2. Leave the ports untagged/native in the management VLAN, same as before.
   The BMC sends its DHCP requests **untagged** — it neither knows nor cares
   which VLAN it lands in; the LAG's *native VLAN* on the switch decides
   that. If your BMC management network is not the default VLAN (e.g.
   VLAN 2), the native VLAN of the **aggregated** port must be set to it.
3. **Re-check the port profile after aggregating.** When UniFi creates an
   aggregation, the LAG takes over the *first* port's profile — and in some
   controller versions it silently reverts to the default profile (native
   VLAN 1). A port that was manually pinned to native VLAN 2 can lose that
   setting the moment it becomes a LAG member. Open the aggregated port's
   settings after creating the LAG and confirm the native VLAN / port
   profile is still your management VLAN, on the LAG itself.
4. After the BMC reboots, the LAG shows as active and both ports as
   participating; the previous MAC-conflict warnings disappear.

## Hash policy: why `layer3+4`

The bond's `xmit-hash-policy` decides which physical link each *egress* flow
uses. The switch ASIC supports three policies, mapped by the driver:

| Policy      | Hash inputs                     | Granularity                              |
|-------------|---------------------------------|------------------------------------------|
| `layer2`    | src+dst MAC                     | one bucket per node↔gateway *pair* — far too coarse |
| `layer2+3`  | src+dst MAC + src+dst IP        | one bucket per node↔host pair — all flows from one node to the NAS share a link |
| `layer3+4`  | src+dst IP + src+dst L4 port    | one bucket per **TCP/UDP connection** — finest available |

With a single 10 Gb storage target (all nodes talking to one IP), `layer2+3`
degenerates to 4 hash inputs total and collisions are common. `layer3+4`
distinguishes individual connections, so combined with multi-connection
transports (below) the distribution converges to a real ~2 Gb.

Remember the **return direction is hashed by the other switch**: the
NAS→node direction (your bulk-read case) is distributed according to the
UniFi LAG hash, which is L2/L3/L4-based and not user-configurable — it is
likewise per-connection, so the same "many flows" rule applies in both
directions.

## Multi-connection transports on the nodes

This is the piece that makes the scenario "two nodes silent, two nodes
saturating" actually deliver ~1 Gb to *each* busy node. With one connection
per node, two flows across two links collide with 50 % probability (both land
on the same link → 500 Mb/s each). With 4+ connections per node, each node's
traffic spreads over both uplinks and the per-node PHY (1 Gb) becomes the
only limit again.

### NFS — `nconnect`

Kernel ≥ 5.3 on the node. Mount the UNAS export with multiple TCP
connections; the NFS client round-robins requests across them:

```sh
mount -t nfs -o vers=4.1,nconnect=8 unas.local:/volume1/share /mnt/share
```

Or in `/etc/fstab`:

```
unas.local:/volume1/share  /mnt/share  nfs  vers=4.1,nconnect=8,_netdev  0 0
```

Prefer `nconnect=8` over 4. The hash placement of each connection is
deterministic (per 4-tuple), and the NFS client round-robins requests
*equally* across its connections — so the node's throughput is gated by the
most congested link, multiplied by the connection count. With only 4
connections per node an unlucky hash can pin 3 of them to one uplink
(measured in practice: a stable 25/75 link split and ~540 Mb/s per node);
8 connections smooth the distribution by the law of large numbers (measured:
~55/45 split). Values above 8 add nothing on a 1 Gb PHY. Verify with
`ss -tn dst <nas-ip> and dport = :2049` — you should see 8 established
connections with distinct source ports.

Note that `nconnect` round-robin does **not** speed up a strictly
single-threaded, queue-depth-1 reader (e.g. `dd iflag=direct`) — use
buffered reads (kernel readahead issues parallel RPCs) or parallel streams
to realize the gain.

### SMB / CIFS — multichannel

The Linux CIFS client (kernel ≥ 5.8, SMB 3.x) can open multiple channels
even over a single NIC:

```sh
mount -t cifs -o vers=3.1.1,multichannel,max_channels=4,credentials=/etc/smb-cred //unas.local/share /mnt/share
```

The server must advertise multichannel support. Check it negotiated:

```sh
cat /proc/fs/cifs/DebugData   # look for "multichannel" and channel count
```

If the server does not support multichannel, SMB traffic from one node stays
a single flow — prefer NFS with `nconnect` for the bulk-transfer datasets.

### iSCSI

Create multiple sessions to the same target and run dm-multipath over them
(`nr_sessions` in iscsid, or multiple ifaces), or set
`node.session.nr_sessions = 4` in `iscsid.conf`. Each session is a separate
TCP connection and hashes independently.

### Generic tools

Anything that opens parallel TCP streams benefits automatically:
`rsync` invocations in parallel, `aria2c -x4`, `bbcp`, object-storage
clients with multipart transfers, etc. Single-stream tools (`scp`, plain
`rsync`, `dd | nc`) stay on one link per node — which is still fine, as one
node cannot exceed one link's capacity anyway.

## Validation

On the BMC, after reboot:

```sh
# LACP negotiated, both slaves active, correct hash policy:
cat /proc/net/bonding/bond0
#   Bonding Mode: IEEE 802.3ad Dynamic link aggregation
#   Transmit Hash Policy: layer3+4 (1)
#   ... both ge0 and ge1 with "MII Status: up", same Aggregator ID

# Offload took effect (no CPU-port bottleneck): node traffic must NOT
# appear on the BMC CPU interface while nodes talk to the LAN:
ip -s link show dsa    # counters should stay near-idle during node iperf3
```

Load test from the nodes (with the NAS or another host as server):

```sh
# on two nodes simultaneously, 4 parallel streams each:
iperf3 -c <server> -P 4 -t 30      # node A
iperf3 -c <server> -P 4 -t 30 -R   # node B (reverse = download direction)
```

While running, watch the per-uplink counters on the BMC:

```sh
watch -n1 'ip -s link show ge0 | sed -n 4,6p; ip -s link show ge1 | sed -n 4,6p'
```

Expected: both `ge0` and `ge1` byte counters climbing, each busy node
reporting ~940 Mb/s, ~1.9 Gb/s combined. If one link stays idle, the flows
hashed onto one bucket — increase `-P`, or verify `layer3+4` is active on
the bond and the client really opened multiple connections (`ss -tn`).

Failover check: pull one uplink cable mid-transfer — throughput drops to
~1 Gb/s total and recovers on replug, with no session resets.

## Troubleshooting an unreachable board

Symptom checklist when the BMC does not come back after activating the
bond config:

* **Switch LAG not (yet) formed** — the most common case. Without an LACP
  partner the bond never carriers up (see the warning above). On UniFi,
  check the port table: both ports must show as members of an *active*
  aggregation. Fix the switch side and power-cycle the BMC — or simply
  wait: the failsafe reverts the config after 5 minutes.
* **Bond exists but has no members / wrong mode** (firmware images built
  before the ifupdown-ng `bond` executor fix): with `CONFIG_BONDING=y` the
  kernel auto-creates an unconfigured `bond0` (balance-rr, no members) at
  boot, and the stock executor bails out when the interface already exists —
  leaving `br0` without any uplink no matter what the switch does. Signature:
  `/proc/net/bonding/bond0` shows `balance-rr` and no slave interfaces. This
  tree ships a patched executor
  (`tp2bmc/board/tp2bmc/overlay/usr/libexec/ifupdown-ng/bond`) that
  reconfigures the pre-existing device in place; on an affected board, update
  `/usr/libexec/ifupdown-ng/bond` from the overlay (or reflash) before
  activating the bond config.
* **LAG formed but wrong native VLAN** — the LAG is active and the bond
  shows carrier, yet DHCP never answers. Creating a UniFi aggregation can
  reset the port profile, dropping a manually configured native VLAN (e.g.
  VLAN 2) back to VLAN 1. The BMC's untagged traffic then lands in the
  wrong network. Verify the *aggregated* port's native VLAN matches your
  management VLAN. Signature in `failsafe-diag.log`: `bond0` and `br0` both
  UP with carrier, LACP negotiated in `/proc/net/bonding/bond0`, but no
  IPv4 address — the L2 path works, the L3 network behind it is the wrong
  one.
* **Failsafe already reverted** — if the board reappears with its old
  config after ~6 minutes, read `/etc/network/failsafe-diag.log` and
  `/etc/network/interfaces.failed` for the evidence of what went wrong.
* **Board stays dark and never reverts** (firmware without the failsafe,
  or a kernel-level failure): recover via one of:
  1. **Serial console** — connect a 3.3 V USB-UART adapter to the BMC
     UART header (115200 8N1), log in and restore the backup:
     `cp /etc/network/interfaces.bak /etc/network/interfaces && reboot`.
     Before rebooting, collect `dmesg`, `ip -d link`, and
     `/proc/net/bonding/bond0` — that data identifies the root cause.
  2. **SD-card recovery** — write the `sdcard.img` release to an SD card
     and boot from it to reflash. Note this wipes the overlay, i.e. all
     local configuration returns to defaults.
* **IPv6 spot-check from a neighboring board**: even with DHCP failed, a
  live BMC answers IPv6 link-local. From another BMC on the same L2:
  `ip -6 neigh show dev br0` after pinging the all-nodes multicast
  address — if the board's MAC is absent, its uplink is truly down (or
  the kernel crashed), not just unaddressed.

## Reverting

```sh
cp /etc/network/interfaces.bak /etc/network/interfaces
reboot
```

(or re-copy the stock file; the migration script will re-add
`bridge-ports node1 node2 node3 node4 ge0 ge1` if needed).
