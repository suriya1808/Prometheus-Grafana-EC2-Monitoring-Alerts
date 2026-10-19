# Installing Grafana on AWS EC2 (Ubuntu) and Connecting It to Prometheus

This guide installs **Grafana OSS** on the **same EC2 `t3.micro` (Ubuntu) server** that already runs Prometheus. It then connects Grafana to Prometheus and builds a first dashboard.

Every command comes with an explanation of what it does and why it's needed.

> **Before you start:** Complete [Prometheus_Installation_Steps.md](Prometheus_Installation_Steps.md) first. Prometheus must be running on this server.

---

## Table of Contents

1. [What is Grafana?](#1-what-is-grafana)
2. [Architecture](#2-architecture)
3. [Prerequisites](#3-prerequisites)
4. [Step 1 – (Recommended) Add swap memory](#step-1--recommended-add-swap-memory)
5. [Step 2 – Install the required packages](#step-2--install-the-required-packages)
6. [Step 3 – Add Grafana's signing key and repository](#step-3--add-grafanas-signing-key-and-repository)
7. [Step 4 – Install Grafana](#step-4--install-grafana)
8. [Step 5 – Start and enable Grafana](#step-5--start-and-enable-grafana)
9. [Step 6 – Open port 3000 in the AWS Security Group](#step-6--open-port-3000-in-the-aws-security-group)
10. [Step 7 – Log in to Grafana](#step-7--log-in-to-grafana)
11. [Step 8 – Add Prometheus as a data source](#step-8--add-prometheus-as-a-data-source)
12. [Step 9 – Create the first dashboard ("Prometheus Up Time")](#step-9--create-the-first-dashboard-prometheus-up-time)
13. [Step 10 – (Optional) Import a ready-made dashboard](#step-10--optional-import-a-ready-made-dashboard)
14. [Step 11 – (Optional) Close port 9090](#step-11--optional-close-port-9090)
15. [Useful commands](#useful-commands)
16. [Troubleshooting](#troubleshooting)
17. [Next steps](#next-steps)
18. [Uninstall Grafana](#uninstall-grafana)

---

## 1. What is Grafana?

[Grafana](https://grafana.com/) is an open-source tool for **visualizing data and building dashboards**. It doesn't store metrics itself. It connects to **data sources**, here Prometheus, queries them, and shows the results as graphs, gauges, tables and alerts.

| Tool | Role |
|---|---|
| **Prometheus** | Collects and stores metrics (the database) |
| **Grafana** | Queries Prometheus and shows the metrics as dashboards (the UI) |

---

## 2. Architecture

```
 Your Laptop (browser)
        │
        │  http://<EC2-Public-IP>:3000
        ▼
 ┌──────────────────── AWS Security Group ────────────────────┐
 │  Inbound: TCP 3000 ← My IP     (TCP 9090 ← My IP, optional) │
 │                                                            │
 │   EC2 t3.micro (Ubuntu)                                    │
 │   ┌──────────────────────┐   http://localhost:9090        │
 │   │ grafana-server :3000 │ ───────────────────────┐       │
 │   └──────────────────────┘                        ▼       │
 │                                  ┌──────────────────────┐ │
 │                                  │ prometheus :9090     │ │
 │                                  └──────────────────────┘ │
 │   RAM: 1 GB  +  Swap: 1 GB (/swapfile)                     │
 └────────────────────────────────────────────────────────────┘
```

Grafana reaches Prometheus over **`localhost`** inside the server. The browser only needs to reach Grafana on port **3000**.

---

## 3. Prerequisites

| Item | Value |
|---|---|
| Server | The same EC2 `t3.micro` running Ubuntu |
| Prometheus | Installed and running (`curl -s localhost:9090/-/ready` → `Prometheus Server is Ready.`) |
| Access | EC2 Instance Connect (browser terminal in the AWS Console) |
| Ports | `3000` (Grafana web UI) |

---

## Step 1 – (Recommended) Add swap memory

### Why do we need swap?

A `t3.micro` has only **1 GB of RAM**. Ubuntu, Prometheus and Grafana together can use most of it, especially during package installs (`apt`), Grafana startup, or heavy dashboard queries.

When Linux runs out of RAM, the **OOM (Out-Of-Memory) killer** forcibly kills a process, often Prometheus or Grafana. The server can also become unresponsive, and even EC2 Instance Connect may stop working.

**Swap** is disk space that Linux uses as overflow memory. It's much slower than RAM, but it acts as a **safety net** that prevents crashes and freezes.

### Check current memory and swap

```bash
free -h
swapon --show
```

| Command | Explanation |
|---|---|
| `free -h` | Shows total, used and free RAM and swap. `-h` = human-readable sizes (Mi, Gi). |
| `swapon --show` | Lists active swap devices or files. **No output = no swap configured.** EC2 Ubuntu images ship without swap. |

### Create a 1 GB swap file

```bash
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
free -h
```

| Command | Explanation |
|---|---|
| `fallocate -l 1G /swapfile` | Reserves a **1 GB** file at `/swapfile` instantly. `-l` sets the length (size). |
| `chmod 600 /swapfile` | Only root can read and write the file. Swap can hold copies of memory, including passwords and keys, so other users must not read it. `mkswap`/`swapon` also warn about insecure permissions. |
| `mkswap /swapfile` | Formats the file as swap space. |
| `swapon /swapfile` | Turns the swap on **now**. |
| `echo '...' \| sudo tee -a /etc/fstab` | Adds a line to `/etc/fstab` so the swap is **turned on again after every reboot**. `-a` = append. Without it, the swap would disappear on reboot. |
| `free -h` | Checks the result. The `Swap:` row should now show **`1.0Gi`**. |

The `/etc/fstab` line is also in the repository: [`system/fstab-swap.conf`](system/fstab-swap.conf).

**The `/etc/fstab` line explained:** `/swapfile none swap sw 0 0`

| Field | Value | Meaning |
|---|---|---|
| 1 | `/swapfile` | The device or file to use |
| 2 | `none` | Mount point. Swap isn't mounted anywhere. |
| 3 | `swap` | Filesystem type |
| 4 | `sw` | Mount options (standard for swap) |
| 5 | `0` | `dump` backup flag (not used) |
| 6 | `0` | `fsck` check order (swap is never checked) |

> ⚠️ Run the `echo ... >> /etc/fstab` command **only once**. Running it again adds duplicate lines. Check with `cat /etc/fstab`.

> If `fallocate` fails with "Operation not supported", use this instead: `sudo dd if=/dev/zero of=/swapfile bs=1M count=1024`, then continue from `chmod`.

### (Optional) Tune swappiness

```bash
cat /proc/sys/vm/swappiness
sudo sysctl vm.swappiness=10
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swappiness.conf
```

| Command | Explanation |
|---|---|
| `cat /proc/sys/vm/swappiness` | Shows the current value. The default is `60`. |
| `sysctl vm.swappiness=10` | Makes Linux **prefer RAM** and use swap only when really needed. Applies immediately. |
| `tee /etc/sysctl.d/99-swappiness.conf` | Keeps the setting after reboots. |

The setting is also in the repository: [`system/99-swappiness.conf`](system/99-swappiness.conf).

**Why:** Swap on an EBS disk is slow. A low swappiness keeps active data in RAM and uses swap only as an emergency buffer.

---

## Step 2 – Install the required packages

```bash
sudo apt install -y apt-transport-https software-properties-common wget gpg
```

| Package | Why it's needed |
|---|---|
| `apt-transport-https` | Lets `apt` download packages over HTTPS. Built into newer Ubuntu versions; harmless to include. |
| `software-properties-common` | Tools for managing extra APT repositories. |
| `wget` | Downloads Grafana's GPG key. |
| `gpg` | Converts the key into the format `apt` expects. |

---

## Step 3 – Add Grafana's signing key and repository

```bash
sudo mkdir -p /etc/apt/keyrings
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
```

| Command | Explanation |
|---|---|
| `mkdir -p /etc/apt/keyrings` | Creates the standard directory for third-party repository keys. |
| `wget -q -O - <url>` | Downloads Grafana's public GPG key. `-q` = quiet, `-O -` = print to the screen (stdout) instead of saving to a file. |
| `gpg --dearmor` | Converts the key from text (ASCII-armored) to the binary format `apt` uses. |
| `sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null` | Saves the key as root, without printing it. |
| `echo "deb [...] https://apt.grafana.com stable main"` | The repository definition: **`deb`** = binary packages, **`signed-by=`** = only trust packages signed with Grafana's key, **`stable main`** = the stable release channel. |
| `sudo tee /etc/apt/sources.list.d/grafana.list` | Saves the repository definition so `apt` knows about it. |

The repository definition is also in this repo: [`system/grafana.list`](system/grafana.list).

**Why use the APT repository** instead of downloading a `.deb` file once? Future Grafana updates arrive with a normal `sudo apt update && sudo apt upgrade`, and every package is **checked against Grafana's signature**.

---

## Step 4 – Install Grafana

```bash
sudo apt update
sudo apt install -y grafana
```

| Command | Explanation |
|---|---|
| `apt update` | Refreshes the package lists, now including the Grafana repository. |
| `apt install -y grafana` | Installs **Grafana OSS**, the free, open-source edition. (`grafana-enterprise` is a separate package.) |

The package installs:

| Path | Purpose |
|---|---|
| `/usr/share/grafana` | Grafana program files |
| `/etc/grafana/grafana.ini` | Main configuration file |
| `/var/lib/grafana` | Data: the SQLite database with dashboards, users and data sources, plus plugins |
| `/var/log/grafana` | Log files |
| `grafana-server.service` | A ready-made systemd service. **Unlike Prometheus, you don't need to write one.** |

---

## Step 5 – Start and enable Grafana

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now grafana-server
sudo systemctl status grafana-server
```

| Command | Explanation |
|---|---|
| `systemctl daemon-reload` | Makes systemd load the newly installed `grafana-server.service`. |
| `systemctl enable --now grafana-server` | Starts Grafana **now** and at **every boot**. |
| `systemctl status grafana-server` | Shows whether it's running. Expect **`Active: active (running)`**. Press **`q`** to exit. |

### Check it on the server

```bash
curl -s localhost:3000/api/health
sudo ss -tlnp | grep 3000
```

| Command | Explanation |
|---|---|
| `curl -s localhost:3000/api/health` | Grafana's health endpoint. Expect JSON that includes `"database": "ok"` and the version. |
| `ss -tlnp \| grep 3000` | Confirms that `grafana` is listening on port `3000`. |

> Grafana can take **10–30 seconds** to start on a `t3.micro`. If `curl` returns nothing, wait a few seconds and try again.

---

## Step 6 – Open port 3000 in the AWS Security Group

Same steps as for port 9090 in the Prometheus guide:

1. **EC2 → Instances →** select the instance **→ Security** tab **→** click the security group.
2. **Edit inbound rules → Add rule**:

   | Field | Value |
   |---|---|
   | Type | `Custom TCP` |
   | Port range | `3000` |
   | Source | `My IP` |
   | Description | `Grafana UI` (optional) |

3. Click **Save rules**.

**About "My IP":** This is your laptop's **public** IP (what <https://checkip.amazonaws.com> shows), **not** the local `192.168.x.x` address. AWS fills it in automatically as `x.x.x.x/32`. If your ISP changes your IP, edit the rule and pick **My IP** again. See the [Prometheus guide](Prometheus_Installation_Steps.md#what-exactly-is-my-ip-public-ip-vs-local-ip) for the full explanation.

> Grafana has a login page, but keeping it limited to **My IP** is still safer. It protects against password-guessing attacks and against vulnerabilities in Grafana itself.

---

## Step 7 – Log in to Grafana

Open this in your laptop's browser:

```
http://<EC2-Public-IPv4>:3000
```

| Field | Default value |
|---|---|
| Username | `admin` |
| Password | `admin` |

Grafana then asks you to **set a new password**. Choose a strong one and don't skip this step: everyone knows the default `admin/admin`.

> The browser shows **"Not secure"** because the connection is plain HTTP. That's expected for this setup. For production, put Grafana behind HTTPS, e.g. Nginx with Let's Encrypt, or an AWS Load Balancer with an ACM certificate.

---

## Step 8 – Add Prometheus as a data source

1. In the left menu, go to **Connections → Data sources**.
2. Click **Add data source** (or **Add new data source**).
3. Choose **Prometheus**.
4. In **Prometheus server URL**, enter:

   ```
   http://localhost:9090
   ```

5. Leave everything else at the defaults, scroll down and click **Save & test**.

✅ Expected: **"Successfully queried the Prometheus API."**

### Why `localhost` and not the public IP?

| URL | Works? | Why |
|---|---|---|
| `http://localhost:9090` | ✅ Recommended | Grafana and Prometheus are on the **same server**. Traffic stays inside the machine and **doesn't pass through the Security Group**. |
| `http://<Public-IP>:9090` | ❌ Usually fails | The traffic leaves and re-enters through the Security Group, which only allows **your laptop's** IP, not the server's own IP. It also breaks when the public IP changes. |

Grafana's queries run **on the server** (the backend), not in your browser. That's why `localhost` here means the EC2 server.

---

## Step 9 – Create the first dashboard ("Prometheus Up Time")

1. Go to **Dashboards → New → New dashboard**.
2. Click **Add visualization** and select the **Prometheus** data source.
3. In the query editor, switch to **Code** mode (top-right of the query box) and enter:

   ```promql
   up
   ```

4. Click **Run queries**.
5. In the right-hand panel options, set **Title** to `Prometheus Up Time`.
6. Click **Save dashboard**, name it (e.g. `Prometheus Up Time`), and save.

![Prometheus Up Time panel](screenshots/grafana-first-panel-up.png)

### Understanding the result

The panel shows a line at **value `1`** with the legend:

```
{__name__="up", app="prometheus", instance="localhost:9090", job="prometheus"}
```

| Part | Meaning |
|---|---|
| `up` | A built-in metric that Prometheus creates **for every target** on each scrape. |
| Value **`1`** | The scrape **succeeded**: the target is up and healthy. ✅ |
| Value **`0`** | The scrape **failed**: the target is down or unreachable. ❌ |
| `job="prometheus"` | Comes from `job_name` in `prometheus.yml`. |
| `instance="localhost:9090"` | The address that was scraped. |
| `app="prometheus"` | A custom label set in the default `prometheus.yml`. |

**Why is the line short and only at the far right?** The dashboard's time range is **Last 6 hours**, but Prometheus has only been collecting data **since it was started**. There's no data for the earlier hours, so that part of the graph is empty. This is **normal**. Change the time picker (top-right) to **Last 15 minutes** and the line fills the panel. It grows across the graph as time passes.

> **Tip:** To make the legend shorter, open the panel editor and, under the query, set **Options → Legend → Custom** to `{{job}}` (or `{{instance}}`).

> **Tip:** For an up/down indicator, change the visualization type from **Time series** to **Stat**. Then, under **Value mappings**, map `1 → UP` (green) and `0 → DOWN` (red).

---

## Step 10 – (Optional) Import a ready-made dashboard

The Grafana community publishes ready-made dashboards that you import by ID.

1. Go to **Dashboards → New → Import**.
2. Enter the dashboard ID **`3662`** (*Prometheus 2.0 Overview*) and click **Load**.
3. Under **Prometheus**, select your data source and click **Import**.

This dashboard shows Prometheus' own internals: scrape duration, samples ingested, memory, storage and more.

> Some panels in older community dashboards may show "No data" on Prometheus 3.x because a few metric names have changed. That's expected.

---

## Step 11 – (Optional) Close port 9090

Now that Grafana shows the data, you no longer need to open the Prometheus UI from your laptop. Grafana reaches Prometheus through `localhost`.

**EC2 → Security group → Edit inbound rules →** delete the **9090** rule **→ Save rules**.

**Why:** Prometheus has **no authentication**. Fewer open ports means less attack surface. Grafana keeps working, because `localhost` traffic doesn't go through the Security Group.

> If you need the Prometheus UI again later, re-add the rule with **My IP**, or use an SSH tunnel.

---

## Useful commands

| Task | Command |
|---|---|
| Check service status | `sudo systemctl status grafana-server` |
| Start / stop / restart | `sudo systemctl start\|stop\|restart grafana-server` |
| Follow live logs | `sudo journalctl -u grafana-server -f` |
| Show the last 50 log lines | `sudo journalctl -u grafana-server -n 50 --no-pager` |
| Grafana log file | `sudo tail -f /var/log/grafana/grafana.log` |
| Edit the config | `sudo nano /etc/grafana/grafana.ini` (then `sudo systemctl restart grafana-server`) |
| Health check | `curl -s localhost:3000/api/health` |
| Check that the port is listening | `sudo ss -tlnp \| grep 3000` |
| Reset the admin password | `sudo grafana cli --homepath /usr/share/grafana --config /etc/grafana/grafana.ini admin reset-admin-password <NEW_PASSWORD>` |
| Upgrade Grafana | `sudo apt update && sudo apt install --only-upgrade grafana` |
| Check memory and swap | `free -h` |
| Top memory users | `ps aux --sort=-%mem \| head -n 10` |

---

## Troubleshooting

### "This site can't be reached" on port 3000

1. **Is Grafana running?** `sudo systemctl status grafana-server` → should be `active (running)`.
2. **Does it respond on the server?** `curl -s localhost:3000/api/health`. If it works here, the problem is the network or the Security Group.
3. **Use `http://`**, not `https://`.
4. **Use the Public IPv4**, not the private `172.31.x.x` address.
5. **Security Group:** check for a Custom TCP `3000` rule with source **My IP**.
6. **Has your public IP changed?** Edit the rule and pick **My IP** again.
7. **Has the EC2 public IP changed after a stop/start?** Check the instance Details tab. An **Elastic IP** gives a fixed address.

### "Save & test" fails on the data source

| Error | Fix |
|---|---|
| `connection refused` | Prometheus isn't running. Run `sudo systemctl status prometheus` and `curl -s localhost:9090/-/ready`. |
| Timeout using the public IP | Use `http://localhost:9090` instead (see [Step 8](#why-localhost-and-not-the-public-ip)). |
| Typo in the URL | It must be exactly `http://localhost:9090`, with no trailing path. |

### Panel shows "No data"

- Check that the **time range** covers the period since Prometheus started (try **Last 15 minutes**).
- Check that the panel uses the right **data source**.
- Run the same query in Grafana **Explore**, or in the Prometheus UI, to confirm the metric exists.

### Server is slow, unresponsive, or services get killed

This usually means you're out of memory.

```bash
free -h
sudo dmesg -T | grep -i -E "out of memory|killed process"
```

| Command | Explanation |
|---|---|
| `free -h` | Shows RAM and swap use. If the `Swap` total is `0B`, swap isn't configured. |
| `dmesg -T \| grep -i ...` | Searches the kernel log for **OOM-killer** events, which show which process was killed and when. `-T` prints readable timestamps. |

**Fix:** Add swap ([Step 1](#step-1--recommended-add-swap-memory)) and restart the affected services. If it keeps happening, move to a larger instance such as `t3.small` (2 GB).

### Forgot the admin password

```bash
sudo grafana cli --homepath /usr/share/grafana --config /etc/grafana/grafana.ini admin reset-admin-password <NEW_PASSWORD>
```

On older Grafana versions, use `sudo grafana-cli admin reset-admin-password <NEW_PASSWORD>`.

---

## Next steps

- **Node Exporter:** Monitor the EC2 server's own CPU, memory, swap, disk and network. Follow [Node_Exporter_Installation_Steps.md](Node_Exporter_Installation_Steps.md).
- **Alerting:** Set up Grafana Alerting or Prometheus Alertmanager to get notified (email, Slack, etc.) when `up == 0`.
- **HTTPS:** Put Grafana behind Nginx with a Let's Encrypt certificate.
- **Elastic IP:** Give the server a fixed public IP.

---

## Uninstall Grafana

```bash
sudo systemctl disable --now grafana-server
sudo apt purge -y grafana
sudo rm -rf /var/lib/grafana /var/log/grafana /etc/grafana
sudo rm /etc/apt/sources.list.d/grafana.list /etc/apt/keyrings/grafana.gpg
sudo apt update
```

| Command | Explanation |
|---|---|
| `systemctl disable --now grafana-server` | Stops Grafana and removes it from boot. |
| `apt purge -y grafana` | Uninstalls the package **and** its config files. |
| `rm -rf /var/lib/grafana ...` | ⚠️ Deletes all dashboards, users, data sources and logs. |
| `rm .../grafana.list .../grafana.gpg` | Removes the Grafana repository and its key. |
| `apt update` | Refreshes package lists without the Grafana repository. |

To also remove swap:

```bash
sudo swapoff /swapfile
sudo rm /swapfile
sudo sed -i '\|^/swapfile|d' /etc/fstab
```

| Command | Explanation |
|---|---|
| `swapoff /swapfile` | Turns the swap off. |
| `rm /swapfile` | Deletes the swap file. |
| `sed -i ... /etc/fstab` | Deletes the `/swapfile` line from `/etc/fstab`, so it isn't turned on at boot. |

Also remove the **port 3000 inbound rule** from the Security Group.
