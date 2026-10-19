# Grafana Queries (PromQL) – EC2 Server Monitoring

Every query used in [`server-monitoring-dashboard.json`](server-monitoring-dashboard.json), grouped by section. Copy any query into a Grafana panel to build your own dashboards.

## How to use a query in Grafana

1. Open a dashboard → **Edit** → **Add** → **Visualization** and select the **Prometheus** data source.
2. In the query editor, switch from **Builder** to **Code** (top-right of the query box).
3. Paste the query into the box.
4. If a **Legend** is listed, open **Options** under the query, set **Legend** → **Custom**, and paste it.
5. If a panel has more than one query, click **+ Add query** for each extra query (B, C, ...).
6. In the right-hand panel options, set the **Visualization** type and **Standard options → Unit** as listed.
7. Click **Run queries**, then **Save dashboard**.

## Notes

- **`$__rate_interval`** is a Grafana variable that picks a suitable time window for `rate()`. It **only works in Grafana**. To run the same query in the Prometheus UI, replace it with a fixed window such as `[5m]`.
- **`{{mountpoint}}`, `{{device}}`, `{{job}}`, `{{name}}`** in legends are replaced with that label's value, so each line gets a readable name.
- **`job="node"`** selects metrics from Node Exporter (see `prometheus/prometheus.yml`).
- The filesystem filter `fstype!~"tmpfs|ramfs|squashfs|overlay|devtmpfs"` hides in-memory and snap (loop) filesystems, so only real disks are shown.
- The network filter `device!~"lo|veth.*|docker.*|br-.*"` hides the loopback and virtual interfaces. On EC2 the real interface is usually `ens5`.
- Unit names below are what you search for in Grafana's **Unit** dropdown:

| Unit in this file | Grafana unit name |
|---|---|
| `percent` | Misc → Percent (0-100) |
| `bytes` | Data → bytes(IEC) |
| `s` | Time → seconds (s) |
| `Bps` | Data rate → bytes/sec(IEC) |
| `bps` | Data rate → bits/sec(SI) |
| `iops` | Throughput → I/O ops/sec (iops) |
| `pps` | Data rate → packets/sec |
| `short` | Misc → Number |

---

## Overview

### Uptime

How long the server has been running since the last boot.

**Visualization:** Stat · **Unit:** `s`

Query A

```promql
node_time_seconds{job="node"} - node_boot_time_seconds{job="node"}
```

### CPU Cores

Number of vCPUs (t3.micro = 2).

**Visualization:** Stat · **Unit:** `short`

Query A

```promql
count(node_cpu_seconds_total{job="node",mode="idle"})
```

### Total RAM

Total physical memory.

**Visualization:** Stat · **Unit:** `bytes`

Query A

```promql
node_memory_MemTotal_bytes{job="node"}
```

### Root Disk Size

Total size of the root (/) filesystem.

**Visualization:** Stat · **Unit:** `bytes`

Query A

```promql
node_filesystem_size_bytes{job="node",mountpoint="/"}
```

### CPU Busy

Current CPU utilization across all cores.

**Visualization:** Gauge · **Unit:** `percent` · **Min/Max:** 0 / 100 · **Thresholds:** green < 70, orange ≥ 70, red ≥ 90

Query A

```promql
100 * (1 - avg(rate(node_cpu_seconds_total{job="node",mode="idle"}[$__rate_interval])))
```

### RAM Used

Percentage of RAM in use (excluding reclaimable cache).

**Visualization:** Gauge · **Unit:** `percent` · **Min/Max:** 0 / 100 · **Thresholds:** green < 70, orange ≥ 70, red ≥ 90

Query A

```promql
100 * (1 - node_memory_MemAvailable_bytes{job="node"} / node_memory_MemTotal_bytes{job="node"})
```

### Swap Used

Percentage of swap in use. Shows 'No data' if swap is not configured.

**Visualization:** Gauge · **Unit:** `percent` · **Min/Max:** 0 / 100 · **Thresholds:** green < 70, orange ≥ 70, red ≥ 90

Query A

```promql
100 * (1 - node_memory_SwapFree_bytes{job="node"} / node_memory_SwapTotal_bytes{job="node"})
```

### Root Disk Used

Percentage of the root (/) disk that is used.

**Visualization:** Gauge · **Unit:** `percent` · **Min/Max:** 0 / 100 · **Thresholds:** green < 70, orange ≥ 70, red ≥ 90

Query A

```promql
100 * (1 - node_filesystem_avail_bytes{job="node",mountpoint="/"} / node_filesystem_size_bytes{job="node",mountpoint="/"})
```

---

## Targets & Services

### Scrape Targets

Is Prometheus able to scrape each target? UP = healthy, DOWN = failing.

**Visualization:** Stat · **Unit:** `short` · **Value mappings:** `1` → UP (green), `0` → DOWN (red)

Query A – legend: `{{job}}`

```promql
up
```

### Services

Is each systemd service (prometheus, grafana-server, node_exporter) active? UP = running.

**Visualization:** Stat · **Unit:** `short` · **Value mappings:** `1` → UP (green), `0` → DOWN (red)

Query A – legend: `{{name}}`

```promql
node_systemd_unit_state{job="node",state="active"}
```

---

## CPU

### CPU Utilization

Overall CPU usage over time (100% = all cores fully busy).

**Visualization:** Time series · **Unit:** `percent` · **Min/Max:** 0 / 100

Query A – legend: `CPU busy %`

```promql
100 * (1 - avg(rate(node_cpu_seconds_total{job="node",mode="idle"}[$__rate_interval])))
```

### CPU Usage by Mode

Where CPU time goes: user = applications, system = kernel, iowait = waiting for disk, steal = taken by the hypervisor.

**Visualization:** Time series · **Unit:** `percent` · **Stack series:** Normal

Query A – legend: `{{mode}}`

```promql
100 * avg by (mode) (rate(node_cpu_seconds_total{job="node",mode!="idle"}[$__rate_interval]))
```

### Load Average

Average number of processes running or waiting. Consistently above the CPU core count = overloaded.

**Visualization:** Time series · **Unit:** `short`

Query A – legend: `1 min`

```promql
node_load1{job="node"}
```

Query B – legend: `5 min`

```promql
node_load5{job="node"}
```

Query C – legend: `15 min`

```promql
node_load15{job="node"}
```

Query D – legend: `CPU cores`

```promql
count(node_cpu_seconds_total{job="node",mode="idle"})
```

### CPU Steal & IOWait

Steal: CPU taken away by AWS (high when t3 CPU credits run out). IOWait: CPU idle waiting for disk.

**Visualization:** Time series · **Unit:** `percent`

Query A – legend: `steal`

```promql
100 * avg(rate(node_cpu_seconds_total{job="node",mode="steal"}[$__rate_interval]))
```

Query B – legend: `iowait`

```promql
100 * avg(rate(node_cpu_seconds_total{job="node",mode="iowait"}[$__rate_interval]))
```

---

## Memory

### Memory Usage

Used = Total - Available. 'Free' is completely unused RAM; 'Available' also counts cache Linux can release.

**Visualization:** Time series · **Unit:** `bytes`

Query A – legend: `Total`

```promql
node_memory_MemTotal_bytes{job="node"}
```

Query B – legend: `Used`

```promql
node_memory_MemTotal_bytes{job="node"} - node_memory_MemAvailable_bytes{job="node"}
```

Query C – legend: `Available`

```promql
node_memory_MemAvailable_bytes{job="node"}
```

Query D – legend: `Free (unused)`

```promql
node_memory_MemFree_bytes{job="node"}
```

Query E – legend: `Buffers + Cache`

```promql
node_memory_Buffers_bytes{job="node"} + node_memory_Cached_bytes{job="node"}
```

### Memory Utilization %

RAM usage percentage over time.

**Visualization:** Time series · **Unit:** `percent` · **Min/Max:** 0 / 100

Query A – legend: `RAM used %`

```promql
100 * (1 - node_memory_MemAvailable_bytes{job="node"} / node_memory_MemTotal_bytes{job="node"})
```

### Swap Usage

How much swap is in use.

**Visualization:** Time series · **Unit:** `bytes`

Query A – legend: `Swap total`

```promql
node_memory_SwapTotal_bytes{job="node"}
```

Query B – legend: `Swap used`

```promql
node_memory_SwapTotal_bytes{job="node"} - node_memory_SwapFree_bytes{job="node"}
```

### Swap Activity

Pages moved between RAM and swap per second. Constant activity = not enough RAM.

**Visualization:** Time series · **Unit:** `short`

Query A – legend: `pages swapped in /s`

```promql
rate(node_vmstat_pswpin{job="node"}[$__rate_interval])
```

Query B – legend: `pages swapped out /s`

```promql
rate(node_vmstat_pswpout{job="node"}[$__rate_interval])
```

### OOM Kills (24h)

Processes killed by the kernel for lack of memory in the last 24 hours. Should be 0.

**Visualization:** Stat · **Unit:** `short`

Query A

```promql
increase(node_vmstat_oom_kill{job="node"}[24h])
```

---

## Disk Space

### Disk Space Used % (per mount)

Current usage of each mounted filesystem.

**Visualization:** Bar gauge · **Unit:** `percent` · **Min/Max:** 0 / 100 · **Thresholds:** green < 70, orange ≥ 70, red ≥ 90

Query A – legend: `{{mountpoint}}`

```promql
100 * (1 - node_filesystem_avail_bytes{job="node",fstype!~"tmpfs|ramfs|squashfs|overlay|devtmpfs"} / node_filesystem_size_bytes{job="node",fstype!~"tmpfs|ramfs|squashfs|overlay|devtmpfs"})
```

### Disk Space Used

Bytes used on each filesystem.

**Visualization:** Time series · **Unit:** `bytes`

Query A – legend: `{{mountpoint}}`

```promql
node_filesystem_size_bytes{job="node",fstype!~"tmpfs|ramfs|squashfs|overlay|devtmpfs"} - node_filesystem_avail_bytes{job="node",fstype!~"tmpfs|ramfs|squashfs|overlay|devtmpfs"}
```

### Disk Space Available

Free space that normal users/processes can still write.

**Visualization:** Time series · **Unit:** `bytes`

Query A – legend: `{{mountpoint}}`

```promql
node_filesystem_avail_bytes{job="node",fstype!~"tmpfs|ramfs|squashfs|overlay|devtmpfs"}
```

### Disk Space Used % Over Time

Trend of disk usage. Useful to predict when the disk will fill up.

**Visualization:** Time series · **Unit:** `percent` · **Min/Max:** 0 / 100

Query A – legend: `{{mountpoint}}`

```promql
100 * (1 - node_filesystem_avail_bytes{job="node",fstype!~"tmpfs|ramfs|squashfs|overlay|devtmpfs"} / node_filesystem_size_bytes{job="node",fstype!~"tmpfs|ramfs|squashfs|overlay|devtmpfs"})
```

### Inodes Used %

Inodes = number of files. A disk can be 'full' from too many small files even with free space.

**Visualization:** Time series · **Unit:** `percent` · **Min/Max:** 0 / 100

Query A – legend: `{{mountpoint}}`

```promql
100 * (1 - node_filesystem_files_free{job="node",fstype!~"tmpfs|ramfs|squashfs|overlay|devtmpfs"} / node_filesystem_files{job="node",fstype!~"tmpfs|ramfs|squashfs|overlay|devtmpfs"})
```

---

## Disk I/O

### Disk Utilization (% busy)

Percentage of time the disk was busy with I/O. Near 100% = disk is a bottleneck.

**Visualization:** Time series · **Unit:** `percent`

Query A – legend: `{{device}}`

```promql
100 * rate(node_disk_io_time_seconds_total{job="node"}[$__rate_interval])
```

### Disk Throughput

Bytes read from and written to disk per second.

**Visualization:** Time series · **Unit:** `Bps`

Query A – legend: `{{device}} read`

```promql
rate(node_disk_read_bytes_total{job="node"}[$__rate_interval])
```

Query B – legend: `{{device}} write`

```promql
rate(node_disk_written_bytes_total{job="node"}[$__rate_interval])
```

### Disk IOPS

Read and write operations per second (gp3 EBS baseline = 3000 IOPS).

**Visualization:** Time series · **Unit:** `iops`

Query A – legend: `{{device}} reads`

```promql
rate(node_disk_reads_completed_total{job="node"}[$__rate_interval])
```

Query B – legend: `{{device}} writes`

```promql
rate(node_disk_writes_completed_total{job="node"}[$__rate_interval])
```

### Disk Latency

Average time per read/write operation. Gaps are normal when there is no I/O.

**Visualization:** Time series · **Unit:** `s`

Query A – legend: `{{device}} read`

```promql
rate(node_disk_read_time_seconds_total{job="node"}[$__rate_interval]) / rate(node_disk_reads_completed_total{job="node"}[$__rate_interval])
```

Query B – legend: `{{device}} write`

```promql
rate(node_disk_write_time_seconds_total{job="node"}[$__rate_interval]) / rate(node_disk_writes_completed_total{job="node"}[$__rate_interval])
```

---

## Network

### Network Traffic

Network bandwidth in bits per second.

**Visualization:** Time series · **Unit:** `bps`

Query A – legend: `{{device}} in`

```promql
rate(node_network_receive_bytes_total{job="node",device!~"lo|veth.*|docker.*|br-.*"}[$__rate_interval]) * 8
```

Query B – legend: `{{device}} out`

```promql
rate(node_network_transmit_bytes_total{job="node",device!~"lo|veth.*|docker.*|br-.*"}[$__rate_interval]) * 8
```

### Network Packets

Packets per second.

**Visualization:** Time series · **Unit:** `pps`

Query A – legend: `{{device}} in`

```promql
rate(node_network_receive_packets_total{job="node",device!~"lo|veth.*|docker.*|br-.*"}[$__rate_interval])
```

Query B – legend: `{{device}} out`

```promql
rate(node_network_transmit_packets_total{job="node",device!~"lo|veth.*|docker.*|br-.*"}[$__rate_interval])
```

### Network Errors & Drops

Should stay at 0. Non-zero values indicate network problems.

**Visualization:** Time series · **Unit:** `pps`

Query A – legend: `{{device}} rx errors`

```promql
rate(node_network_receive_errs_total{job="node",device!~"lo|veth.*|docker.*|br-.*"}[$__rate_interval])
```

Query B – legend: `{{device}} tx errors`

```promql
rate(node_network_transmit_errs_total{job="node",device!~"lo|veth.*|docker.*|br-.*"}[$__rate_interval])
```

Query C – legend: `{{device}} rx drops`

```promql
rate(node_network_receive_drop_total{job="node",device!~"lo|veth.*|docker.*|br-.*"}[$__rate_interval])
```

Query D – legend: `{{device}} tx drops`

```promql
rate(node_network_transmit_drop_total{job="node",device!~"lo|veth.*|docker.*|br-.*"}[$__rate_interval])
```

### TCP Connections

Number of TCP connections by state.

**Visualization:** Time series · **Unit:** `short`

Query A – legend: `established`

```promql
node_netstat_Tcp_CurrEstab{job="node"}
```

Query B – legend: `time_wait`

```promql
node_sockstat_TCP_tw{job="node"}
```

Query C – legend: `in use`

```promql
node_sockstat_TCP_inuse{job="node"}
```

---

## System

### Processes

Processes currently running or blocked.

**Visualization:** Time series · **Unit:** `short`

Query A – legend: `running`

```promql
node_procs_running{job="node"}
```

Query B – legend: `blocked (waiting on I/O)`

```promql
node_procs_blocked{job="node"}
```

### Context Switches & Interrupts

How often the CPU switches between tasks and handles hardware interrupts.

**Visualization:** Time series · **Unit:** `short`

Query A – legend: `context switches /s`

```promql
rate(node_context_switches_total{job="node"}[$__rate_interval])
```

Query B – legend: `interrupts /s`

```promql
rate(node_intr_total{job="node"}[$__rate_interval])
```

### Open File Descriptors

Number of open files/sockets system-wide.

**Visualization:** Time series · **Unit:** `short`

Query A – legend: `allocated`

```promql
node_filefd_allocated{job="node"}
```

### Time Sync Offset

Clock difference from the time server (NTP / Amazon Time Sync). Should stay close to 0.

**Visualization:** Time series · **Unit:** `s`

Query A – legend: `offset`

```promql
node_timex_offset_seconds{job="node"}
```

---

## Monitoring Stack (Prometheus & Grafana)

### Memory Used by Monitoring Stack

RAM used by Prometheus, Grafana and Node Exporter processes.

**Visualization:** Time series · **Unit:** `bytes`

Query A – legend: `{{job}}`

```promql
process_resident_memory_bytes{job=~"prometheus|grafana|node"}
```

### Prometheus Active Time Series

Number of time series Prometheus is tracking. More series = more RAM.

**Visualization:** Time series · **Unit:** `short`

Query A – legend: `series`

```promql
prometheus_tsdb_head_series{job="prometheus"}
```

### Prometheus Storage Size

Disk space used by Prometheus data in /var/lib/prometheus.

**Visualization:** Time series · **Unit:** `bytes`

Query A – legend: `blocks`

```promql
prometheus_tsdb_storage_blocks_bytes{job="prometheus"}
```

Query B – legend: `WAL`

```promql
prometheus_tsdb_wal_storage_size_bytes{job="prometheus"}
```
