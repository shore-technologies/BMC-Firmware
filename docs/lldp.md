# LLDP on the BMC switch

The firmware enables `lldpd` on the BMC's switch-facing and node-facing DSA
ports (`ge0`, `ge1`, and `node1`…`node4`). The Realtek switch traps LLDP frames
to the BMC CPU port, where `lldpd` can receive and advertise them on each
managed port.

## What this changes

LLDP uses the link-local destination `01:80:c2:00:00:0e`, which a conforming
bridge does not forward. The switch's current default allowed these frames to
cross the board; the firmware now explicitly traps the reserved group and the
LLDP ethertype classifier to the CPU instead. This lets `lldpd` report actual
neighbors per BMC port and advertise the BMC as a bridge-capable device.

This is LLDP termination on the BMC, not a transparent LLDP relay. It will not
make the BMC forward a node's LLDP advertisement to the uplink, or vice versa.
The BMC should appear as a neighbor on the links where it is attached, and the
nodes may see the BMC on their own ports.

## UniFi topology

UniFi's topology view is built from the LLDP neighbor tables and MAC tables
that UniFi devices report; the BMC's own tables are not visible to it. UniFi
nests a client under a third-party LLDP neighbor when the neighbor's chassis ID
is the MAC address UniFi already knows that device by and the client is
recorded on the same switch port as the neighbor.

With an LACP uplink this needs one extra step. UniFi records every MAC address
behind a LAG on the LAG's first (lowest-numbered) switch port. If the BMC
advertises on both uplinks, the switch holds two neighbor entries, UniFi places
the BMC on the second port, and the nodes stay on the first port, directly
under the switch. The firmware therefore transmits LLDP on `ge1` only and keeps
`ge0` receive-only.

This was observed on bmc-240 (Twisted LAG 12/24, UniFi Network on a UDM Pro
Max, 2026-11-19) through the controller API:

| BMC transmits LLDP on | Twisted neighbor entries | bmc-240 recorded on | rk-241…244 uplink |
|---|---|---|---|
| `ge0` and `ge1` | ports 12 and 24 | port 24 | Twisted, port 12 |
| `ge1` only | port 12 | port 12 | bmc-240 |

The change took effect about three minutes after `ge0` stopped transmitting,
once the port 24 entry had aged out (TTL 120 s). It was made with MAC address
port IDs; interface-name port IDs were not tried in the single-uplink case.

`ge1` must be the uplink wired to the lower-numbered switch port of the LAG.
Check with:

```sh
lldpcli show neighbors summary
```

`ge0` still receives, so both uplinks are listed with the switch port they are
connected to. If `ge0` is on the lower-numbered port, swap the cables or change
`ge0` to `ge1` in `/etc/lldpd.d/tp2bmc.conf`. A board with a single uplink must
use `ge1`, or have that line removed.

If UniFi still shows the nodes directly attached to the switch, check that the
switch lists the BMC as an LLDP neighbor on exactly one port, the first port of
the LAG.

## Configuration

The daemon reads `/etc/lldpd.d/*.conf` at startup. The firmware configuration
selects `ge*` and `node*` interfaces, uses the MAC address as LLDP port ID
(all ports share one MAC address, so the interface name in the port description
is what tells them apart), uses each bond slave's real MAC as the frame source,
and sets `ge0` to receive-only. CDP, FDP, EDP, and SONMP support is disabled;
standard LLDP remains enabled.

To inspect the running daemon:

```sh
lldpcli show configuration
lldpcli show neighbors details
```

To watch LLDP on a BMC port:

```sh
tcpdump -eni node1 'ether dst 01:80:c2:00:00:0e'
```

On a connected node, a capture should show LLDP sent by the BMC on that link;
the Twisted switch's LLDP advertisements should no longer traverse the board
to the node. That capture has not been taken yet.

## Security and verification

`lldpd` runs as root and parses link-local frames received from the node
interfaces. The agent should therefore be treated as part of the BMC's network
attack surface. Its advertised system and port information is visible to
directly connected devices.

If the BMC does not need to receive LLDP from the nodes, add this command to
`/etc/lldpd.d/tp2bmc.conf` to keep advertising on the node ports while ignoring
incoming LLDP there:

```text
configure ports node1,node2,node3,node4 lldp status tx-only
```

This reduces the frames `lldpd` parses from the nodes, but node neighbors will
no longer appear in `lldpcli show neighbors`.

On bmc-240 the trap, the uplink neighbors and the UniFi nesting were confirmed
on the running board. The port ID subtype and the receive-only setting were
applied there with `lldpcli` at runtime; reading them from the configuration
file at boot has not been tested. After flashing, check:

```sh
lldpcli show interfaces ports ge0,ge1 summary
```

`ge0` should report `RX only` and both ports a MAC address port ID.