
# Redis Enterprise Monitoring Script (Zabbix‑Ready)

This repository provides a portable Bash script (`re_summary_zabbix.sh`) to fetch and summarize key **Redis Enterprise** metrics from the **Prometheus v2** endpoint.  
It runs on **Linux** and **macOS** (BSD awk compatible) and integrates smoothly with **Zabbix** via a master text item + dependent items.

---

## Features

- ✅ **Cluster overview**
  - Exporter status
  - Cluster name
  - Total number of databases
  - Ingress / Egress traffic totals
- ✅ **Per‑database details**
  - Memory used (MB)
  - Memory usage (% of `maxmemory`)
  - Replication / HA flag (Y/N → 1/0)
  - Status (`0 = active`)
  - Latency **p50 (µs)** (derived from histogram)
  - Ingress / Egress bytes

---

## Script Usage

Fetch metrics from a live endpoint (self‑signed TLS example):

```bash
./re_summary_zabbix.sh --endpoint https://<your-cluster>:8070/v2 --insecure
```

Or read from a saved scrape file:

```bash
./re_summary_zabbix.sh --from-file scrape_out.txt
```

Example output:

```
==== Redis Enterprise Summary @ 2025-09-25 17:08:55 UTC ====
Exporter up:  yes
Cluster:  gabs.redisdemo.com
Databases:  13
DBs not active:  0
Total ingress (bytes):  1394948
Total egress  (bytes):  541359

DB_UID    DB_NAME                     mem_used_MB  mem_used_%  HA  status  p50_us  ingress   egress
22        cache-soccer-workshop       3            0.1         N   0       1       107148    17978
24        aa-adiq                     7            7.4         Y   0       528     115904    32500
...
```

> Requirements: `curl` and `awk` (GNU awk or BSD awk).  
> If your Prometheus endpoint doesn’t expose `redis_server_used_memory` and `redis_server_maxmemory`, the script cannot compute memory %. In that case, check exporter coverage or scrape another node that exports those metrics.

---

# Tutorial — Integrate with Zabbix Agent 2

This section explains how to wire the script into **Zabbix Agent 2** and extract metrics with **dependent items**.

> ⚠️ Assumes you already have a working **Zabbix Server** and **Zabbix Agent 2** connected.

## 1) Install the script on the Agent host

```bash
sudo mkdir -p /etc/zabbix/scripts
sudo curl -o /etc/zabbix/scripts/re_summary_zabbix.sh   https://raw.githubusercontent.com/gacerioni/redis-enterprise-zabbix-monitor/main/re_summary_zabbix.sh
sudo chmod 755 /etc/zabbix/scripts/re_summary_zabbix.sh
sudo chown zabbix:zabbix /etc/zabbix/scripts/re_summary_zabbix.sh
```

## 2) Add a UserParameter

Create `/etc/zabbix/zabbix_agent2.d/redis_enterprise.conf`:

```ini
# Master item: fetches the full Prometheus text once per call.
# The item key passes arguments that map to --endpoint ($1) and optional flag ($2).
UserParameter=redis.re.summary[*],/etc/zabbix/scripts/re_summary_zabbix.sh --endpoint $1 $2
```

Restart the agent:

```bash
sudo systemctl restart zabbix-agent2
```

Test locally:

```bash
zabbix_agent2 -t 'redis.re.summary[https://<your-cluster>:8070/v2,--insecure]'
```

## 3) Create the Master Item (Text) in the Zabbix UI

On the **host where the agent runs**:

- **Name:** `RE v2 summary (text)`  
- **Type:** `Zabbix agent`  
- **Key:**
  ```
  redis.re.summary[https://<your-cluster>:8070/v2,--insecure]
  ```
- **Type of information:** `Text`  
- **Update interval:** `60s`  
- **History:** `1d`  
- **Trends:** `0`  

This item stores the full table your script prints.

## 4) Create Dependent Items (Regex preprocessing)

Each metric becomes a **Dependent item** that parses the **master** text item.

### 4.1 Cluster‑level metrics

Create dependent items with **one** preprocessing step (**Regular expression**) and **Output** `\1`:

- **DB count**  
  - Key: `redis.re.cluster.db_count`  
  - Type: `Numeric (unsigned)`  
  - Regex:
    ```
    (?m)^Databases:\s+([0-9]+)
    ```

- **DBs not active**  
  - Key: `redis.re.cluster.db_not_active`  
  - Type: `Numeric (unsigned)`  
  - Regex:
    ```
    (?m)^DBs not active:\s+([0-9]+)
    ```

- **Total ingress**  
  - Key: `redis.re.cluster.ingress_bytes`  
  - Units: `B`  
  - Type: `Numeric (unsigned)`  
  - Regex:
    ```
    (?m)^Total ingress \(bytes\):\s+([0-9]+)
    ```

- **Total egress**  
  - Key: `redis.re.cluster.egress_bytes`  
  - Units: `B`  
  - Type: `Numeric (unsigned)`  
  - Regex:
    ```
    (?m)^Total egress\s+\(bytes\):\s+([0-9]+)
    ```

### 4.2 Per‑DB metrics (use your own DB UIDs)

Your table columns are:

```
1=DB_UID  2=DB_NAME  3=mem_used_MB  4=mem_used_%  5=HA  6=st  7=p50_us  8=ingress  9=egress
```

Pick a **DB UID** from your output (example **<DB_UID>**), and create these items.
All are **Dependent items** (master = `RE v2 summary (text)`)
and each has **Regex** with **Output** `\1`:

- **Memory MB**  
  - Key: `redis.re.db.mem_mb.<DB_UID>`  
  - Type: `Numeric (float)` • Units: `MB`  
  - Regex:
    ```
    (?m)^\s*<DB_UID>\s+\S+\s+([0-9.]+)\s+
    ```

- **Memory %**  
  - Key: `redis.re.db.mem_pct.<DB_UID>`  
  - Type: `Numeric (float)` • Units: `%`  
  - Regex:
    ```
    (?m)^\s*<DB_UID>\s+\S+\s+\S+\s+([0-9.]+)\s+
    ```

- **HA (1/0)**  
  - Key: `redis.re.db.ha.<DB_UID>`  
  - Type: `Numeric (unsigned)`  
  - Preprocessing:
    1) **Regex**
       ```
       (?m)^\s*<DB_UID>\s+\S+\s+\S+\s+\S+\s+([YN])\s+
       ```
       Output: `\1`  
    2) **JavaScript**
       ```js
       return value === 'Y' ? 1 : 0;
       ```

- **Status**  
  - Key: `redis.re.db.status.<DB_UID>`  
  - Type: `Numeric (unsigned)`  
  - Regex:
    ```
    (?m)^\s*<DB_UID>\s+\S+(?:\s+\S+){4}\s+([0-9]+)\s+
    ```

- **Latency p50 (µs)**  
  - Key: `redis.re.db.p50_us.<DB_UID>`  
  - Type: `Numeric (float)` • Units: `us`  
  - Regex:
    ```
    (?m)^\s*<DB_UID>\s+\S+(?:\s+\S+){5}\s+([0-9.]+)\s+
    ```

- **Ingress bytes**  
  - Key: `redis.re.db.ingress.<DB_UID>`  
  - Type: `Numeric (unsigned)` • Units: `B`  
  - Regex:
    ```
    (?m)^\s*<DB_UID>\s+\S+(?:\s+\S+){6}\s+([0-9.]+)\s+
    ```

- **Egress bytes**  
  - Key: `redis.re.db.egress.<DB_UID>`  
  - Type: `Numeric (unsigned)` • Units: `B`  
  - Regex:
    ```
    (?m)^\s*<DB_UID>\s+\S+(?:\s+\S+){7}\s+([0-9.]+)\s*$
    ```

> 🔎 Tip: Create the first DB item, then **Clone** and just change the key + regex UID.

## 5) (Optional) Triggers

Examples (replace `<Host>` and `<DB_UID>`):

- **Mem usage > 80% (Warning)**
  ```
  {<Host>:redis.re.db.mem_pct.<DB_UID>.last()}>80
  ```

- **Mem usage > 90% (High)**
  ```
  {<Host>:redis.re.db.mem_pct.<DB_UID>.last()}>90
  ```

- **p50 > 2ms for 5m (Warning)**
  ```
  {<Host>:redis.re.db.p50_us.<DB_UID>.avg(5m)}>2000
  ```

- **Not HA (High)**
  ```
  {<Host>:redis.re.db.ha.<DB_UID>.last()}=0
  ```

- **DB not active (High)**
  ```
  {<Host>:redis.re.db.status.<DB_UID>.last()}<>0
  ```

## 6) Ask Users for Output (to discover UIDs)

Every environment is different. Ask users to run:

```bash
./re_summary_zabbix.sh --endpoint https://<their-cluster>:8070/v2 --insecure | head -n 40
```

Then pick the **DB_UID** values from the table and create items for those.

---

## Troubleshooting

- **Item “not supported”** → Regex mismatch. Open the master item’s latest value, copy a row, and test the regex in the item’s preprocessing “Test” dialog.  
- **No memory %** → Endpoint lacks `redis_server_used_memory/maxmemory`. Scrape another node/exporter that exposes these metrics.  
- **Timeouts** → Increase Zabbix “Timeout” (Administration → General → Other), or bump item update interval.

---

## License

MIT
