# BMC sensors and Prometheus metrics

The firmware exposes the BMC's own health data in two ways:

* on the shell, through `sensors` (lm-sensors), `ethtool` and sysfs;
* over HTTP for Prometheus, through collectd's `write_prometheus` plugin at
  `http://<bmc>:9103/metrics`.

> **Status (2026-11-19):** this image was flashed to bmc-240 and tested on the
> running board. The T113 thermal zone and collectd endpoint are live, and the
> metrics below were confirmed there. The other three BMCs have not been
> flashed; keep the endpoint on the trusted LAN because it has no auth or TLS.

## What is collected

| Source | collectd plugin | Data |
| --- | --- | --- |
| T113 SoC thermal sensor | `thermal`, `sensors` | BMC chip temperature |
| System fan | `thermal`, `sensors` | cooling level; RPM only on boards with an EMC2301 (v2.4 mod) |
| Switch ports and uplinks | `interface` | bytes, packets, errors and drops for `node1`…`node4`, `ge0`, `ge1`, the conduit, bridges and bonds |
| SPI NAND | `ubi` | bad eraseblock count, maximum erase counter |
| Filesystems | `df` | space on `/` and `/mnt/sdcard` |
| BMC system | `cpu`, `load`, `memory`, `uptime` | CPU time per core and state, load average, memory, uptime |

The temperature is that of the BMC's own SoC. The board has no sensors for
node temperature or node power draw; those have to be collected on the nodes
themselves.

The SoC temperature appears twice, once from the `thermal` plugin and once
from `sensors`, because the kernel publishes the thermal zone as a hwmon
device as well.

## SoC temperature sensor

`sun8i-t113s-turing-pi2.dtsi` adds the T113 thermal sensor
(`allwinner,sun20i-d1-ths`), its calibration cell in the SID eFuse and a
`cpu-thermal` zone. Kernel 6.8.12 already has the driver
(`CONFIG_SUN8I_THERMAL`) but its SoC dtsi lacks these nodes; they follow the
ones added to mainline later. The nodes are shared by all board revisions and
are left out of the U-Boot build.

The zone's trip points (85 °C passive, 100 °C hot) are informational. Mainline
marks 100 °C as `critical`, which makes the kernel power the system off; that
is deliberately not done here, so that adding monitoring cannot shut a BMC
down.

On the BMC:

```sh
cat /sys/class/thermal/thermal_zone0/temp   # millidegrees Celsius
sensors
```

## Prometheus

collectd listens on TCP port 9103 on all addresses. A scrape configuration:

```yaml
scrape_configs:
  - job_name: turingpi-bmc
    static_configs:
      - targets:
          - 192.0.2.10:9103
          - 192.0.2.11:9103
```

Every metric carries an `instance` label holding the BMC's hostname. Prometheus
keeps its own `instance` label (the scrape address) and renames collectd's to
`exported_instance`. Set `honor_labels: true` in the scrape job only if every
BMC has a unique hostname.

Expected metric names:

| Metric | Labels | Meaning |
| --- | --- | --- |
| `collectd_thermal_temperature` | `thermal="thermal_zone0"` | SoC temperature, °C |
| `collectd_thermal_gauge` | `thermal="cooling_device0"` | fan cooling level |
| `collectd_sensors_temperature` | `sensors="<chip>"`, `type="temp1"` | SoC temperature, °C |
| `collectd_sensors_fanspeed` | `sensors="<chip>"`, `type="fan1"` | fan RPM (EMC2301 only) |
| `collectd_interface_if_octets_rx_total`, `…_tx_total` | `interface="node1"` | bytes per port |
| `collectd_interface_if_packets_…`, `if_errors_…`, `if_dropped_…` | `interface` | packets, errors, drops |
| `collectd_ubi_max_ec`, `collectd_ubi_bad_peb_count` | `ubi="ubi0"` | NAND wear |
| `collectd_df_df_complex` | `df="root"`, `type="used"` | filesystem space, bytes |
| `collectd_cpu_total` | `cpu="0"`, `type="user"` | CPU time, jiffies |
| `collectd_memory` | `memory="used"` | memory, bytes |
| `collectd_load_shortterm`, `…_midterm`, `…_longterm` | | load average |
| `collectd_uptime` | | uptime, seconds |

Counters end in `_total`; use `rate()` on them, for example
`rate(collectd_interface_if_octets_rx_total{interface="node1"}[5m]) * 8` for
bits per second received from node 1.

### Access control

The endpoint has no authentication or TLS, and it is reachable from the nodes
like the other BMC services. It is read-only. To bind it to a single address,
add a `Host` line to the plugin block in `/etc/collectd.conf`:

```
<Plugin write_prometheus>
	Host "192.0.2.10"
	Port "9103"
</Plugin>
```

## Configuration

The collectd configuration is `tp2bmc/board/tp2bmc/overlay/etc/collectd.conf`,
installed as `/etc/collectd.conf`. No plugin in it writes to flash; values are
held in memory only.

After editing the file on a BMC, run `/etc/init.d/S90collectd restart`. A
locally edited `/etc/collectd.conf` is kept across firmware updates, so such a
BMC will not pick up a newer default from the firmware.

## Building

The collectd plugins, `lm-sensors` and `ethtool` are enabled in
`tp2bmc/configs/tp2bmc_defconfig`. `write_prometheus` pulls in `libmicrohttpd`
and `protobuf-c`.

Buildroot does not rebuild packages when their options change. In a build
tree that has already been built, run this inside `buildroot/` before
`./scripts/build.sh`:

```sh
make collectd-dirclean
make linux-rebuild
```

## Checking a flashed board

```sh
cat /proc/device-tree/thermal-zones/cpu-thermal/trips/cpu-hot/type
cat /sys/class/thermal/thermal_zone0/temp
sensors
grep collectd /var/log/messages
curl -s http://localhost:9103/metrics | grep collectd_thermal
```
