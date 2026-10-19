# Monitoring the EC2 Server with Node Exporter, Prometheus and Grafana

This guide adds **full server monitoring** to the existing Prometheus and Grafana setup: CPU, RAM, swap, disk space, disk I/O, network, processes, systemd services, and the monitoring stack itself.

Every command comes with an explanation of what it does and why it's needed.

> **Before you start:** Complete these two guides first:
> 1. [Prometheus_Installation_Steps.md](Prometheus_Installation_Steps.md)
> 2. [Grafana_Installation_Steps.md](Grafana_Installation_Steps.md)

---

## Table of Contents

1. [What is Node Exporter?](#1-what-is-node-exporter)
2. [Architecture](#2-architecture)
3. [Files in this repository](#3-files-in-this-repository)
4. [What gets monitored](#4-what-gets-monitored)
5. [Step 1 – Create a dedicated user](#step-1--create-a-dedicated-user)
6. [Step 2 – Download the latest Node Exporter](#step-2--download-the-latest-node-exporter)
7. [Step 3 – Install the binary](#step-3--install-the-binary)
8. [Step 4 – Create the systemd service](#step-4--create-the-systemd-service)
9. [Step 5 – Start and enable Node Exporter](#step-5--start-and-enable-node-exporter)
10. [Step 6 – Update the Prometheus config](#step-6--update-the-prometheus-config)
11. [Step 7 – Check that Prometheus is scraping all targets](#step-7--check-that-prometheus-is-scraping-all-targets)
12. [Step 8 – Import the Grafana dashboard](#step-8--import-the-grafana-dashboard)
13. [Step 9 – (Optional) Build your own panels from the queries](#step-9--optional-build-your-own-panels-from-the-queries)
14. [Step 10 – (Optional) Import the community "Node Exporter Full" dashboard](#step-10--optional-import-the-community-node-exporter-full-dashboard)
15. [Security notes](#security-notes)
16. [Useful commands](#useful-commands)
17. [Troubleshooting](#troubleshooting)
18. [Uninstall Node Exporter](#uninstall-node-exporter)

---

## 1. What is Node Exporter?

Prometheus doesn't read server metrics by itself. It only **scrapes** HTTP endpoints that expose metrics.

[**Node Exporter**](https://github.com/prometheus/node_exporter) is the official Prometheus **exporter for Linux servers**. It reads system information from `/proc` and `/sys` and from systemd, and publishes it as metrics at `http://localhost:9100/metrics`. Prometheus then scrapes it like any other target.

| Component | Port | Role |
|---|---|---|
| **Node Exporter** | `9100` | Exposes server metrics (CPU, RAM, disk, network, ...) |
| **Prometheus** | `9090` | Scrapes and stores the metrics |
| **Grafana** | `3000` | Shows the metrics as dashboards |

---

## 2. Architecture

```
 Your Laptop (browser) ── http://<EC2-Public-IP>:3000 ──┐
                                                         │  (Security Group: 3000 ← My IP)
 ┌───────────────────── EC2 t3.micro (Ubuntu) ───────────┼───────────────┐
 │                                                       ▼               │
 │  ┌──────────────────┐   queries (localhost:9090)  ┌──────────────┐    │
 │  │ Prometheus :9090 │ ◄────────────────────────── │ Grafana :3000│    │
 │  └──────┬───────────┘                             └──────▲───────┘    │
 │         │ scrapes every 15s                               │            │
 │         ├──► localhost:9090/metrics  (Prometheus itself)  │            │
 │         ├──► localhost:9100/metrics  (Node Exporter) ─────┘ (server)   │
 │         └──► localhost:3000/metrics  (Grafana)                         │
 │                                                                       │
 │  ┌─────────────────────────┐   reads /proc, /sys, systemd             │
 │  │ Node Exporter           │ ◄──── CPU · RAM · swap · disks · network │
 │  │ 127.0.0.1:9100 (local)  │                                          │
 │  └─────────────────────────┘                                          │
 └───────────────────────────────────────────────────────────────────────┘
```

Node Exporter listens on **`127.0.0.1` only**, so it can't be reached from outside the server. **No new Security Group rule is needed.**

---

## 3. Files in this repository

| File | Copy to (on the server) | Purpose |
|---|---|---|
| [`prometheus/node_exporter.service`](prometheus/node_exporter.service) | `/etc/systemd/system/node_exporter.service` | systemd service that runs Node Exporter |
| [`prometheus/prometheus.yml`](prometheus/prometheus.yml) | `/etc/prometheus/prometheus.yml` | Prometheus config with 3 scrape targets: Prometheus, Node Exporter, Grafana |
| [`grafana/server-monitoring-dashboard.json`](grafana/server-monitoring-dashboard.json) | Imported in the Grafana UI | Ready-made "EC2 Server Monitoring" dashboard (39 panels) |
| [`grafana/queries.md`](grafana/queries.md) | Pasted into Grafana panels | Every PromQL query in the dashboard, with legend, unit and visualization type |

---

## 4. What gets monitored

| Section | Metrics |
|---|---|
| **Overview** | Uptime, CPU cores, total RAM, root disk size, CPU busy %, RAM used %, swap used %, root disk used % |
| **Targets & Services** | UP/DOWN status of every scrape target, and of the `prometheus`, `grafana-server`, `node_exporter` and `alertmanager` systemd services |
| **CPU** | CPU utilization %, usage by mode (user/system/iowait/steal/...), load average (1/5/15 min), CPU steal & iowait |
| **Memory** | Total / used / available / free / buffers+cache, RAM utilization %, swap total/used, swap in/out activity, OOM kills |
| **Disk Space** | Used % per mount point, used bytes, available bytes, usage trend, inode usage |
| **Disk I/O** | Disk utilization (% busy), read/write throughput, IOPS, read/write latency |
| **Network** | Traffic in/out (bits/s), packets/s, errors & drops, TCP connections |
| **System** | Running/blocked processes, context switches & interrupts, open file descriptors, clock (NTP) offset |
| **Monitoring Stack** | RAM used by Prometheus/Grafana/Node Exporter, Prometheus active series, Prometheus storage size |

> **Why CPU steal matters on `t3.micro`:** T3 instances are **burstable**. They earn CPU credits while idle and spend them when busy. When the credits run out, AWS throttles the CPU to the baseline (10% per vCPU for `t3.micro`), and this shows up as **steal** time. A high steal % means you're out of CPU credits.

---

## Step 1 – Create a dedicated user

```bash
sudo useradd --no-create-home --shell /usr/sbin/nologin node_exporter
```

| Part | Explanation |
|---|---|
| `useradd node_exporter` | Creates a system user named `node_exporter`. |
| `--no-create-home` | No home directory is needed. |
| `--shell /usr/sbin/nologin` | Nobody can log in as this user. |

**Why:** As with Prometheus, the service should **not run as root** (principle of least privilege). Node Exporter only *reads* system information, which a normal user is allowed to do.

---

## Step 2 – Download the latest Node Exporter

> ⚠️ **Download into your home folder (`~`), not `/tmp`.** On newer Ubuntu versions, `/tmp` is a small, RAM-based `tmpfs` (half the RAM, about 455 MB on a `t3.micro`) with a per-user limit. Large downloads there fail with *"Disk quota exceeded"*. Adding swap does **not** raise this limit.

```bash
cd ~
NE_VERSION=$(curl -s https://api.github.com/repos/prometheus/node_exporter/releases/latest | grep '"tag_name"' | cut -d '"' -f4 | sed 's/^v//')
echo "Installing Node Exporter $NE_VERSION"
wget https://github.com/prometheus/node_exporter/releases/download/v${NE_VERSION}/node_exporter-${NE_VERSION}.linux-amd64.tar.gz
tar xvf node_exporter-${NE_VERSION}.linux-amd64.tar.gz
```

| Command | Explanation |
|---|---|
| `cd ~` | Switches to your home folder (on the real disk) to download into. |
| `curl -s https://api.github.com/.../releases/latest` | Gets the latest Node Exporter release details from GitHub as JSON. |
| `grep '"tag_name"' \| cut -d '"' -f4 \| sed 's/^v//'` | Pulls out the version number, e.g. `v1.x.y` → `1.x.y`. Same technique as in the Prometheus guide. |
| `NE_VERSION=$( ... )` | Stores it in the variable `NE_VERSION`. |
| `echo ...` | Prints the detected version. **If it's empty**, see [Troubleshooting](#ne_version-variable-is-empty). |
| `wget <url>` | Downloads the Linux x86_64 (`amd64`) release archive. |
| `tar xvf <file>` | Extracts the archive. |

> `t3` instances are **x86_64**, so this uses `linux-amd64`. On Graviton (`t4g`) instances, use `linux-arm64`.

---

## Step 3 – Install the binary

```bash
sudo cp node_exporter-${NE_VERSION}.linux-amd64/node_exporter /usr/local/bin/
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
node_exporter --version
```

| Command | Explanation |
|---|---|
| `cp .../node_exporter /usr/local/bin/` | Copies the single binary into a directory on the system `PATH`. |
| `chown node_exporter:node_exporter` | Makes the `node_exporter` user the owner. |
| `node_exporter --version` | Confirms the install by printing the version. |

Clean up the downloaded files:

```bash
rm -rf ~/node_exporter-*
```

---

## Step 4 – Create the systemd service

The service file is in this repository: [`prometheus/node_exporter.service`](prometheus/node_exporter.service).

### Option A – Paste the file with `nano`

```bash
sudo nano /etc/systemd/system/node_exporter.service
```

1. Copy the whole contents of [`prometheus/node_exporter.service`](prometheus/node_exporter.service) and paste it into the editor. In the browser terminal, right-click → **Paste** or press `Ctrl+Shift+V`.
2. Save with **`Ctrl+O`**, then **`Enter`**.
3. Exit with **`Ctrl+X`**.

| Command | Explanation |
|---|---|
| `sudo nano <file>` | Opens (or creates) the file in the `nano` text editor as root. Files in `/etc/systemd/system` can only be written by root. |

### Option B – Create it with one command

```bash
sudo tee /etc/systemd/system/node_exporter.service > /dev/null <<'EOF'
[Unit]
Description=Prometheus Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter \
  --web.listen-address=127.0.0.1:9100 \
  --collector.systemd \
  --collector.systemd.unit-include=(prometheus|grafana-server|node_exporter|alertmanager).service
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

`sudo tee` and the `<<'EOF'` heredoc work the same way as in the Prometheus guide. They write everything between the `EOF` markers into the file as root.

### What the service file means

| Line | Explanation |
|---|---|
| `Wants=` / `After=network-online.target` | Starts after the network is up. |
| `User=` / `Group=node_exporter` | Runs as the non-root user from Step 1. |
| `ExecStart=/usr/local/bin/node_exporter` | Starts Node Exporter. Most collectors (CPU, memory, disk, filesystem, network, load, ...) are **enabled by default**. |
| `--web.listen-address=127.0.0.1:9100` | Listens on **localhost only**. Only processes on the server (Prometheus) can reach it, so your server metrics aren't exposed to the internet. The default would be all interfaces (`:9100`). |
| `--collector.systemd` | Turns on the **systemd collector** (off by default). It reports whether each service is active, failed, etc. This feeds the dashboard's **Services** panel. |
| `--collector.systemd.unit-include=(prometheus\|grafana-server\|node_exporter\|alertmanager).service` | Collects state **only for these services** (`alertmanager` is added for the [alerting guide](Alerting_Setup_Steps.md)). Collecting every systemd unit on the server would create hundreds of extra time series and waste RAM on a 1 GB instance. |
| `Restart=on-failure` | Restarts automatically if it crashes. |
| `WantedBy=multi-user.target` | Starts at boot (after `systemctl enable`). |

> **Monitoring more services:** Add names to the list, e.g. `(prometheus|grafana-server|node_exporter|alertmanager|nginx|ssh).service`, then run `sudo systemctl daemon-reload && sudo systemctl restart node_exporter`.

---

## Step 5 – Start and enable Node Exporter

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
sudo systemctl status node_exporter
```

| Command | Explanation |
|---|---|
| `systemctl daemon-reload` | Makes systemd read the new unit file. |
| `systemctl enable --now node_exporter` | Starts it **now** and at **every boot**. |
| `systemctl status node_exporter` | Should show **`Active: active (running)`**. Press **`q`** to exit. |

### Check it on the server

```bash
curl -s localhost:9100/metrics | head -n 20
curl -s localhost:9100/metrics | grep -E '^node_memory_MemAvailable_bytes|^node_filesystem_avail_bytes\{.*mountpoint="/"'
curl -s localhost:9100/metrics | grep '^node_systemd_unit_state.*state="active"'
```

| Command | Explanation |
|---|---|
| `curl -s localhost:9100/metrics \| head -n 20` | Shows the first 20 lines of the metrics page. You should see lines like `# HELP ...`, `# TYPE ...` and metric values. |
| `... \| grep -E '^node_memory_MemAvailable_bytes\|...'` | Shows the available RAM and the free space on `/`, in bytes, to confirm real data is being collected. |
| `... \| grep '^node_systemd_unit_state.*state="active"'` | Confirms the systemd collector works. Each of the 3 services should show the value `1`. |

---

## Step 6 – Update the Prometheus config

Prometheus doesn't know about Node Exporter yet. Replace the config with [`prometheus/prometheus.yml`](prometheus/prometheus.yml), which adds two new **scrape jobs**:

| Job | Target | What it collects |
|---|---|---|
| `prometheus` | `localhost:9090` | Prometheus' own metrics (already there) |
| `node` | `localhost:9100` | **Server metrics from Node Exporter** (new) |
| `grafana` | `localhost:3000` | **Grafana's own metrics** (new). Grafana exposes `/metrics` by default. |

> **Note:** The repository's `prometheus.yml` also contains the `alerting`, `rule_files` and `alertmanager` sections from the [alerting guide](Alerting_Setup_Steps.md). It's safe to use them before Alertmanager is installed. Prometheus ignores a rule file that doesn't exist yet, and the `alertmanager` target simply shows as DOWN until you complete that guide.

### 6.1 Back up the current config

```bash
sudo cp /etc/prometheus/prometheus.yml /etc/prometheus/prometheus.yml.bak
```

**Why:** If something goes wrong, you can restore the working config with `sudo cp /etc/prometheus/prometheus.yml.bak /etc/prometheus/prometheus.yml`.

### 6.2 Empty the file and paste the new config

```bash
sudo truncate -s 0 /etc/prometheus/prometheus.yml
sudo nano /etc/prometheus/prometheus.yml
```

| Command | Explanation |
|---|---|
| `truncate -s 0 <file>` | Empties the file (size 0) while keeping its owner (`prometheus`) and permissions. This is easier than deleting every line in `nano`. |
| `nano <file>` | Opens the empty file. Paste the full contents of [`prometheus/prometheus.yml`](prometheus/prometheus.yml), then press **`Ctrl+O`**, **`Enter`**, **`Ctrl+X`**. |

> ⚠️ YAML is **indentation-sensitive** and allows **spaces only, no tabs**. Paste the file exactly as it is.

### 6.3 Validate and reload

```bash
promtool check config /etc/prometheus/prometheus.yml
sudo systemctl reload prometheus
```

| Command | Explanation |
|---|---|
| `promtool check config <file>` | Checks the new config. It must print **`SUCCESS`**. If it reports an error, fix the line it mentions, usually an indentation problem. |
| `systemctl reload prometheus` | Sends a HUP signal (the `ExecReload` line in `prometheus.service`) so Prometheus loads the new config **without restarting** and without losing data. |

---

## Step 7 – Check that Prometheus is scraping all targets

### From the server

```bash
curl -s localhost:9090/api/v1/targets | grep -o '"scrapeUrl":"[^"]*"\|"health":"[a-z]*"'
```

| Part | Explanation |
|---|---|
| `curl -s localhost:9090/api/v1/targets` | Calls the Prometheus API that lists every scrape target and its health, as JSON. |
| `grep -o '...'` | Prints only the URL and health of each target, for readability. |

Expected: three targets, each `"health":"up"`:

```
"scrapeUrl":"http://localhost:3000/metrics"
"health":"up"
"scrapeUrl":"http://localhost:9100/metrics"
"health":"up"
"scrapeUrl":"http://localhost:9090/metrics"
"health":"up"
```

### From the browser

- **Prometheus UI** (if port 9090 is still open): **Status → Target health**. `grafana`, `node` and `prometheus` should all be **UP**.
- **Grafana:** **Explore** → select Prometheus → run `up`. You should see 3 series with the value `1`.

---

## Step 8 – Import the Grafana dashboard

The file [`grafana/server-monitoring-dashboard.json`](grafana/server-monitoring-dashboard.json) contains the complete **"EC2 Server Monitoring"** dashboard.

1. In Grafana, go to **Dashboards → New → Import**.
2. Either:
   - click **Upload dashboard JSON file** and select `server-monitoring-dashboard.json`, **or**
   - open the file, copy its **entire** contents, and paste them into **Import via dashboard JSON model**.
3. Click **Load**.
4. Under **Prometheus**, select your Prometheus data source.
5. Click **Import**.

**Why the data source is picked at import time:** The JSON uses the placeholder `${DS_PROMETHEUS}` instead of a fixed data source ID. Grafana replaces it with the data source you choose, so the same file works in any Grafana installation.

### What it looks like

![Dashboard – overview, targets, services and CPU](screenshots/dashboard-overview-cpu.png)

![Dashboard – load, CPU steal and memory](screenshots/dashboard-cpu-memory.png)

![Dashboard – swap and disk space](screenshots/dashboard-swap-disk.png)

![Dashboard – network, system and monitoring stack](screenshots/dashboard-network-system.png)

### Dashboard layout

| Section | Panels |
|---|---|
| Overview | Uptime · CPU Cores · Total RAM · Root Disk Size · CPU Busy · RAM Used · Swap Used · Root Disk Used |
| Targets & Services | Scrape Targets (UP/DOWN) · Services (UP/DOWN) |
| CPU | CPU Utilization · CPU Usage by Mode · Load Average · CPU Steal & IOWait |
| Memory | Memory Usage · Memory Utilization % · Swap Usage · Swap Activity · OOM Kills (24h) |
| Disk Space | Used % per mount · Used · Available · Used % over time · Inodes Used % |
| Disk I/O | Disk Utilization · Throughput · IOPS · Latency |
| Network | Traffic · Packets · Errors & Drops · TCP Connections |
| System | Processes · Context Switches & Interrupts · Open File Descriptors · Time Sync Offset |
| Monitoring Stack | Memory used by Prometheus/Grafana/Node Exporter · Active Series · Storage Size |

Dashboard settings: the default time range is **last 1 hour**, it **auto-refreshes every 30 s**, and gauges turn **orange at 70%** and **red at 90%**.

> **Tip:** Hover over the ⓘ icon next to a panel's title to see what it shows.

> **Tip:** Graphs only show data **from the time Node Exporter was started**. Older time ranges stay empty, exactly like the first "Prometheus Up Time" panel.

---

## Step 9 – (Optional) Build your own panels from the queries

Every query in the dashboard is listed in [`grafana/queries.md`](grafana/queries.md), with its **legend**, **unit**, **visualization type** and **thresholds**. To add a panel by hand:

1. Open a dashboard → **Edit → Add → Visualization** → select **Prometheus**.
2. Switch the query editor to **Code** and paste the query.
3. Set the legend under **Options → Legend → Custom**, if one is listed.
4. Set the **Visualization** type and **Standard options → Unit**.
5. **Run queries → Save dashboard**.

> Queries use `$__rate_interval`, which **only works in Grafana**. To test a query in the Prometheus UI, replace `$__rate_interval` with `5m`.

---

## Step 10 – (Optional) Import the community "Node Exporter Full" dashboard

The Grafana community dashboard **Node Exporter Full** (ID **`1860`**) shows hundreds of extra metrics in detail.

1. **Dashboards → New → Import**.
2. Enter **`1860`** and click **Load**.
3. Select your Prometheus data source and click **Import**.
4. At the top of the dashboard, set **Job** = `node`.

You can keep both dashboards. They read the same data.

---

## Security notes

| Item | Setting | Why |
|---|---|---|
| Node Exporter port `9100` | Bound to `127.0.0.1`, **no** Security Group rule | Server metrics (disk layout, processes, network) shouldn't be public. Only Prometheus, on the same machine, needs it. |
| Prometheus port `9090` | Only **My IP**, or closed | No authentication. Grafana uses `localhost`, so the port can be closed. |
| Grafana port `3000` | Only **My IP** | Login-protected, but limiting access still reduces risk. |
| Services | Each runs as its own non-login user | Least privilege. A compromised service can't take over the server. |

---

## Useful commands

| Task | Command |
|---|---|
| Node Exporter status | `sudo systemctl status node_exporter` |
| Restart Node Exporter | `sudo systemctl restart node_exporter` |
| Node Exporter logs | `sudo journalctl -u node_exporter -f` |
| View raw server metrics | `curl -s localhost:9100/metrics \| less` (press `q` to quit) |
| Search for a metric | `curl -s localhost:9100/metrics \| grep node_filesystem` |
| Check all scrape targets | `curl -s localhost:9090/api/v1/targets \| grep -o '"health":"[a-z]*"'` |
| Validate and reload the Prometheus config | `promtool check config /etc/prometheus/prometheus.yml && sudo systemctl reload prometheus` |
| Restore the previous Prometheus config | `sudo cp /etc/prometheus/prometheus.yml.bak /etc/prometheus/prometheus.yml && sudo systemctl reload prometheus` |
| Status of all 3 services | `systemctl is-active prometheus grafana-server node_exporter` |
| Memory used by each service | `ps -o rss,comm -C prometheus,grafana,node_exporter` (RSS in KB) |

---

## Troubleshooting

### `node` target is DOWN

1. Is Node Exporter running? Run `sudo systemctl status node_exporter`.
2. Does it respond? Run `curl -s localhost:9100/metrics | head`.
3. Check the logs: `sudo journalctl -u node_exporter -n 30 --no-pager`.
4. Check that the target in `prometheus.yml` is `localhost:9100`, then reload Prometheus.

### `grafana` target is DOWN

1. Run `curl -s localhost:3000/metrics | head`. You should see metrics.
2. If you get a 404 or nothing, the metrics endpoint is disabled. In `/etc/grafana/grafana.ini`, make sure the `[metrics]` section has `enabled = true` (the default), then run `sudo systemctl restart grafana-server`.

### `promtool check config` fails after pasting

Usually an **indentation** problem. Compare your file with [`prometheus/prometheus.yml`](prometheus/prometheus.yml) and use **spaces only**, no tabs. Or restore the backup:

```bash
sudo cp /etc/prometheus/prometheus.yml.bak /etc/prometheus/prometheus.yml
```

### "Services" panel shows "No data"

- Check that the systemd collector is enabled: `curl -s localhost:9100/metrics | grep -c node_systemd_unit_state` should print a number greater than 0.
- If it prints `0`, check that `--collector.systemd` is in `/etc/systemd/system/node_exporter.service`, then run `sudo systemctl daemon-reload && sudo systemctl restart node_exporter`.

### "Swap Used" shows "No data" or NaN

Swap isn't configured, so swap total = 0 and the percentage can't be calculated. Add swap as described in [Grafana_Installation_Steps.md – Step 1](Grafana_Installation_Steps.md#step-1--recommended-add-swap-memory).

### "Disk Latency" panel has gaps

This is normal. Latency = time ÷ number of operations. When there are **no reads or writes** in a period, the division has no result and the graph shows a gap.

### Panels show "No data" after import

- Check that the right data source was selected during import. Open the panel → **Edit** → check the data source.
- Change the time range to **Last 15 minutes**. Data only exists from when Node Exporter started.
- Run `up{job="node"}` in **Explore**. If there's no result, Prometheus isn't scraping Node Exporter; see [Step 7](#step-7--check-that-prometheus-is-scraping-all-targets).

### `NE_VERSION` variable is empty

The GitHub API call failed (rate limit or network). Look up the latest version at <https://github.com/prometheus/node_exporter/releases> and set it by hand:

```bash
NE_VERSION=1.x.y     # replace with the actual latest version, without the leading "v"
```

---

## Uninstall Node Exporter

```bash
sudo systemctl disable --now node_exporter
sudo rm /etc/systemd/system/node_exporter.service
sudo systemctl daemon-reload
sudo rm /usr/local/bin/node_exporter
sudo userdel node_exporter
```

| Command | Explanation |
|---|---|
| `systemctl disable --now node_exporter` | Stops it and removes it from boot. |
| `rm .../node_exporter.service` + `daemon-reload` | Deletes the service definition and makes systemd forget it. |
| `rm /usr/local/bin/node_exporter` | Removes the binary. |
| `userdel node_exporter` | Removes the service user. |

Then remove the `node` job from `/etc/prometheus/prometheus.yml` and reload Prometheus.
