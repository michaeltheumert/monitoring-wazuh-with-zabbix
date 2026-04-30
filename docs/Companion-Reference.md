# Companion Reference — Zabbix–Wazuh Integration Guide

*Practical implementation companion to the series: [Monitoring Wazuh with Zabbix](./README.md)*

---

## About This Reference

This document is the **step-by-step implementation companion** to the six-part article series *Monitoring Wazuh with Zabbix*.

Where the series explains the reasoning — why monitoring is designed the way it is, what failure scenarios each check addresses, how to think about alerting and operational ownership — this reference provides the **instructions**.

Use them together:

- Read the series to understand the design
- Use this reference to build and operate the implementation

> **Scope:** Single-node Wazuh deployment as baseline. Multi-node and distributed extensions are noted where they differ. For full architectural context on distributed environments, see [Part 6](./Part-6_From-Monitoring-to-Trust.md).

> Validated against Wazuh 4.14.x and Zabbix 7.4.9.

---

## Prerequisites

Before following this guide, confirm the following:

| Requirement | Verification command |
|---|---|
| Wazuh installed and running | `systemctl status wazuh-manager` |
| Zabbix server reachable from Wazuh host | `ping [zabbix-server-ip]` |
| Zabbix agent installed on Wazuh host | `systemctl status zabbix-agent2` |
| Zabbix agent communicating with server | Check `Monitoring → Latest Data` for the host |
| Wazuh host added in Zabbix | `Configuration → Hosts` |

---

## Part 1 — Installing and Configuring the Zabbix Agent

### 1.1 Install the Zabbix agent

On the Wazuh host (Ubuntu/Debian):

```bash
wget https://repo.zabbix.com/zabbix/7.4/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.4+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_latest_7.4+ubuntu24.04_all.deb
sudo apt update
sudo apt install zabbix-agent2
```

On the Wazuh host (RHEL/CentOS):

```bash
sudo rpm -Uvh https://repo.zabbix.com/zabbix/7.4/release/rhel/9/noarch/zabbix-release-latest-7.4.el9.noarch.rpm
sudo dnf install zabbix-agent2
```

> **Note:** Adjust the distribution version in the URL to match your system (e.g. `ubuntu22.04`, `el8`). For all distributions and the latest package URLs, refer to the official Zabbix download page:
> [https://www.zabbix.com/download](https://www.zabbix.com/download)

### 1.2 Configure the agent

Edit `/etc/zabbix/zabbix_agent2.conf`:

```ini
# Point to your Zabbix server
Server=<ZABBIX_SERVER_IP>
ServerActive=<ZABBIX_SERVER_IP>

# Set the hostname exactly as it will appear in Zabbix
Hostname=wazuh-server-01

# Required for active log monitoring
RefreshActiveChecks=120
```

#### 1.2.1 Enable PSK encryption (recommended)

By default, communication between the Zabbix agent and the Zabbix server is unencrypted. On isolated internal networks this may be acceptable, but for any environment where the Wazuh host and the Zabbix server communicate across a network segment that is not fully trusted — including cloud environments, DMZs, or paths that cross network boundaries — enabling encryption is strongly recommended.

Zabbix supports two encryption methods: certificate-based (TLS) and pre-shared key (PSK). PSK requires no certificate infrastructure and is the practical choice for most Wazuh monitoring deployments.

> **Performance note:** The overhead of PSK encryption on agent communication is negligible. There is no meaningful operational cost to enabling it.

> **LibreSSL note:** If your Zabbix packages were compiled against LibreSSL rather than OpenSSL or GnuTLS, PSK is not supported — only certificate-based encryption is available. Verify with `zabbix_agent2 --version`.

**Step 1 — Generate the PSK key**

On the Wazuh host, generate a 256-bit pre-shared key and save it directly to a file:

```bash
sudo openssl rand -hex 32 > /etc/zabbix/zabbix_agentd.psk
```

Restrict permissions so only the `zabbix` user can read it:

```bash
sudo chown zabbix:zabbix /etc/zabbix/zabbix_agentd.psk
sudo chmod 640 /etc/zabbix/zabbix_agentd.psk
```

Verify:

```bash
ls -la /etc/zabbix/zabbix_agentd.psk
# Expected: -rw-r----- 1 zabbix zabbix ...
```

> **Important:** The PSK *value* must be kept secret and never committed to version control. The PSK *identity* string (configured below) is transmitted unencrypted — do not put sensitive information in it.

**Step 2 — Add TLS directives to the agent configuration**

Add the following lines to `/etc/zabbix/zabbix_agent2.conf`:

```ini
# PSK encryption
TLSConnect=psk
TLSAccept=psk
TLSPSKFile=/etc/zabbix/zabbix_agentd.psk
TLSPSKIdentity=wazuh-server-01-psk
```

| Directive | Purpose |
|---|---|
| `TLSConnect=psk` | Agent uses PSK for outgoing connections (active checks) |
| `TLSAccept=psk` | Agent only accepts PSK-encrypted incoming connections |
| `TLSPSKFile` | Path to the file containing the PSK value |
| `TLSPSKIdentity` | Descriptive, non-secret identifier — visible in transit |

> **Critical:** Once `TLSAccept=psk` is set, the agent will **reject all unencrypted connections**. Configure the matching PSK in the Zabbix UI (Step 3 below) **before** restarting the agent, or monitoring will be interrupted.

**Step 3 — Configure PSK in the Zabbix frontend**

**Navigate to:** `Configuration → Hosts → [Wazuh Host] → Encryption`

| Field | Value |
|---|---|
| Connections to host | `PSK` |
| Connections from host | `PSK` |
| PSK identity | `wazuh-server-01-psk` (must match `TLSPSKIdentity`) |
| PSK | Paste the full 64-character hex string from the PSK file |

Click **Update**. The server applies the new settings when the configuration cache syncs — typically within 60 seconds.

**Step 4 — Restart the agent and verify**

```bash
sudo systemctl restart zabbix-agent2
```

Verify the encrypted connection from the Zabbix server:

```bash
zabbix_get -s <WAZUH_HOST_IP> -k system.hostname \
  --tls-connect=psk \
  --tls-psk-identity="wazuh-server-01-psk" \
  --tls-psk-file=/etc/zabbix/zabbix_agentd.psk
```

Check `Monitoring → Latest Data` to confirm items are still collecting. If they stop, check the agent log:

```bash
tail -50 /var/log/zabbix/zabbix_agent2.log | grep -i "tls\|psk\|error"
```

Common issues:

| Symptom | Likely cause |
|---|---|
| TLS handshake failure | PSK value mismatch between file and Zabbix UI |
| Items stop collecting after restart | Server config cache not yet synced — wait 60s |
| "Connection rejected" | `TLSAccept=psk` active but server not yet updated |
| `zabbix_get` fails | PSK identity string mismatch (case-sensitive) |

**Per-host PSK recommendation**

Each monitored host should have its own unique PSK. A single shared key across all hosts means a compromise of one key affects all monitored systems. When adding additional Wazuh nodes (indexer, dashboard), generate a new PSK for each and use a unique `TLSPSKIdentity` per host.

Official reference: [https://www.zabbix.com/documentation/current/en/manual/encryption/using_pre_shared_keys](https://www.zabbix.com/documentation/current/en/manual/encryption/using_pre_shared_keys)

---

### 1.3 Configure log file permissions

Wazuh log files are owned by the `wazuh` user. The Zabbix agent runs as the `zabbix` user and needs read access.

```bash
# Add zabbix user to wazuh group
sudo usermod -aG wazuh zabbix

# Verify group membership
groups zabbix

# Restart the agent to apply the change
sudo systemctl restart zabbix-agent2
```

Verify access:

```bash
sudo -u zabbix cat /var/ossec/logs/ossec.log | head -5
sudo -u zabbix cat /var/ossec/logs/alerts/alerts.json | head -5
```

Both commands should return content without a "permission denied" error.

### 1.4 Enable and start the agent

```bash
sudo systemctl enable zabbix-agent2
sudo systemctl start zabbix-agent2
sudo systemctl status zabbix-agent2
```

### 1.5 Add the Wazuh host in Zabbix

**Navigate to:** `Configuration → Hosts → Create host`

| Field | Value |
|---|---|
| Host name | `wazuh-server-01` (must match `Hostname` in agent config) |
| Visible name | `Wazuh Server 01` |
| Groups | `Wazuh Managers` (create if it does not exist) |
| Agent interface | IP address of the Wazuh host, port `10050` |

Click **Add**, then verify connectivity:

`Configuration → Hosts → [Host] → Availability` — the agent icon should turn green within 60 seconds.

---

## Part 2 — Creating Monitoring Items

All items are created via:

`Configuration → Hosts → [Wazuh Host] → Items → Create item`

---

### Item 1 — Wazuh Manager Process Count

| Field | Value |
|---|---|
| Name | `Wazuh Manager — process count` |
| Type | `Zabbix agent` |
| Key | `proc.num[wazuh-manager]` |
| Type of information | `Numeric (unsigned)` |
| Units | *(leave blank)* |
| Update interval | `60s` |
| History storage period | `90d` |
| Trend storage period | `365d` |
| Description | Returns the number of running wazuh-manager processes. Expected: 1 or more. |

**Verify:** After saving, navigate to `Monitoring → Latest Data` and confirm the item returns `1`.

---

### Item 2 — Alert Log Last Modification Time

| Field | Value |
|---|---|
| Name | `Wazuh — alerts.json last modification time` |
| Type | `Zabbix agent` |
| Key | `vfs.file.time[/var/ossec/logs/alerts/alerts.json,modify]` |
| Type of information | `Numeric (unsigned)` |
| Units | `unixtime` |
| Update interval | `60s` |
| History storage period | `90d` |
| Description | Unix timestamp of last modification to alerts.json. Stops advancing when the detection pipeline stalls. |

**Verify:** The value should match the current time (within the polling interval). Zabbix will display it as a human-readable timestamp.

---

### Item 3 — ERROR Entries in ossec.log

This item uses **active log monitoring**. Confirm `ServerActive` is set in the agent configuration before creating this item.

| Field | Value |
|---|---|
| Name | `Wazuh — ERROR entries in ossec.log` |
| Type | `Zabbix agent (active)` |
| Key | `log[/var/ossec/logs/ossec.log,"ERROR"]` |
| Type of information | `Log` |
| Update interval | `60s` |
| Log time format | *(leave blank)* |
| Description | Captures log lines containing ERROR from ossec.log. Each match is stored as a separate log entry. |

**Optional — add additional pattern items:**

| Pattern | Item name |
|---|---|
| `CRITICAL` | `Wazuh — CRITICAL entries in ossec.log` |
| `failed` | `Wazuh — failed entries in ossec.log` |

---

### Item 4 — Disk Usage on /

| Field | Value |
|---|---|
| Name | `Disk usage on /` |
| Type | `Zabbix agent` |
| Key | `vfs.fs.size[/,pused]` |
| Type of information | `Numeric (float)` |
| Units | `%` |
| Update interval | `60s` |
| History storage period | `90d` |
| Trend storage period | `365d` |
| Description | Percentage of disk space used on the root filesystem. |

**If Wazuh data is on a separate partition** (e.g. `/var`), create a second item:

| Field | Value |
|---|---|
| Name | `Disk usage on /var` |
| Key | `vfs.fs.size[/var,pused]` |

---

### Item 5 — CPU Utilisation

| Field | Value |
|---|---|
| Name | `CPU utilisation` |
| Type | `Zabbix agent` |
| Key | `system.cpu.util[,avg1]` |
| Type of information | `Numeric (float)` |
| Units | `%` |
| Update interval | `60s` |
| History storage period | `90d` |
| Trend storage period | `365d` |

---

### Item 6 — Memory Usage

| Field | Value |
|---|---|
| Name | `Memory usage` |
| Type | `Zabbix agent` |
| Key | `vm.memory.size[pused]` |
| Type of information | `Numeric (float)` |
| Units | `%` |
| Update interval | `60s` |

---

### Item 7 — Filebeat Process Count

Required if Filebeat is used to forward alerts to the indexer.

| Field | Value |
|---|---|
| Name | `Filebeat — process count` |
| Type | `Zabbix agent` |
| Key | `proc.num[filebeat]` |
| Type of information | `Numeric (unsigned)` |
| Update interval | `60s` |

---

### Item 8 — Monitoring Heartbeat

This item validates that the Zabbix agent is communicating and data collection is functioning end-to-end.

First, add a UserParameter to the agent configuration:

```bash
sudo nano /etc/zabbix/zabbix_agent2.conf
```

Add the following line:

```ini
UserParameter=monitoring.heartbeat,date +%s
```

Restart the agent:

```bash
sudo systemctl restart zabbix-agent2
```

Then create the item in Zabbix:

| Field | Value |
|---|---|
| Name | `Monitoring heartbeat` |
| Type | `Zabbix agent` |
| Key | `monitoring.heartbeat` |
| Type of information | `Numeric (unsigned)` |
| Units | `unixtime` |
| Update interval | `60s` |

---

## Part 3 — Creating Triggers

All triggers are created via:

`Configuration → Hosts → [Wazuh Host] → Triggers → Create trigger`

Replace `[Wazuh Host]` in expressions with the exact hostname as configured in Zabbix.

---

### Trigger 1 — Wazuh Manager Not Running

| Field | Value |
|---|---|
| Name | `Wazuh Manager is not running` |
| Expression | `last(/[Wazuh Host]/proc.num[wazuh-manager])=0` |
| Severity | `High` |
| Manual close | Enabled |
| Description | The wazuh-manager process has stopped. Security event processing has halted. Restart the service and investigate root cause. |

---

### Trigger 2 — Alert Log Stale

| Field | Value |
|---|---|
| Name | `Wazuh — no alerts generated in the last 10 minutes` |
| Expression | `(now()-last(/[Wazuh Host]/vfs.file.time[/var/ossec/logs/alerts/alerts.json,modify]))>600` |
| Severity | `High` |
| Manual close | Enabled |
| Description | alerts.json has not been updated for 10 minutes. Detection pipeline may have stalled. Check process state, disk space, and ossec.log for errors. |

**Threshold guidance:**
- High-volume environments (alerts every few seconds): reduce to `300` (5 minutes)
- Low-volume environments (few agents, infrequent events): increase to `900–1200` (15–20 minutes)

---

### Trigger 3 — ERROR Messages in Log

| Field | Value |
|---|---|
| Name | `Wazuh — ERROR entries detected in ossec.log` |
| Expression | `count(/[Wazuh Host]/log[/var/ossec/logs/ossec.log,"ERROR"],5m)>0` |
| Severity | `Warning` |
| Manual close | Enabled |
| Description | ERROR-level messages have appeared in ossec.log. Review entries to determine severity. |

---

### Trigger 4 — Disk Warning

| Field | Value |
|---|---|
| Name | `Disk usage above 85% on /` |
| Expression | `last(/[Wazuh Host]/vfs.fs.size[/,pused])>85` |
| Severity | `Warning` |
| Manual close | Enabled |
| Description | Disk usage has exceeded 85%. Free space or extend filesystem before reaching critical threshold. |

### Trigger 5 — Disk Critical

| Field | Value |
|---|---|
| Name | `Disk usage critical — above 95% on /` |
| Expression | `last(/[Wazuh Host]/vfs.fs.size[/,pused])>95` |
| Severity | `High` |
| Manual close | Enabled |
| Description | Disk usage is critical. Wazuh log writes may fail imminently. Immediate action required. |

---

### Trigger 6 — Sustained High CPU

| Field | Value |
|---|---|
| Name | `CPU utilisation above 90% for 5 minutes` |
| Expression | `min(/[Wazuh Host]/system.cpu.util[,avg1],5m)>90` |
| Severity | `Warning` |
| Manual close | Enabled |
| Description | CPU has been above 90% for 5 minutes. Event processing may be delayed. |

---

### Trigger 7 — Filebeat Not Running

| Field | Value |
|---|---|
| Name | `Filebeat is not running` |
| Expression | `last(/[Wazuh Host]/proc.num[filebeat])=0` |
| Severity | `High` |
| Manual close | Enabled |
| Description | Filebeat process has stopped. Alerts may not be reaching the Wazuh indexer. |

---

### Trigger 8 — Monitoring Heartbeat Lost

| Field | Value |
|---|---|
| Name | `Monitoring heartbeat lost — agent not communicating` |
| Expression | `nodata(/[Wazuh Host]/monitoring.heartbeat,300)=1` |
| Severity | `High` |
| Manual close | Enabled |
| Description | No data received from the monitoring heartbeat for 5 minutes. The Zabbix agent may have stopped or lost connectivity. All monitoring on this host is unreliable. |

---

### Trigger 9 — Item Data Collection Lost (nodata template)

Apply this pattern to any critical item where silent data collection failure must be detected:

| Field | Value |
|---|---|
| Name | `[Item name] — no data received for 5 minutes` |
| Expression | `nodata(/[Wazuh Host]/[item.key],300)=1` |
| Severity | `Warning` |
| Description | Zabbix has not received data from this item for 5 minutes. Verify agent status and item configuration. |

---

## Part 4 — Configuring Notifications

### 4.1 Configure a Media Type — Email

**Navigate to:** `Administration → Media types → Create media type`

| Field | Value |
|---|---|
| Name | `Email — Operations` |
| Type | `Email` |
| SMTP server | `[your SMTP server]` |
| SMTP port | `587` |
| SMTP helo | `[your domain]` |
| SMTP email | `zabbix@[your domain]` |
| Connection security | `STARTTLS` |
| Authentication | `Username and password` |

Click **Test** to verify delivery before saving.

---

### 4.2 Configure a Media Type — Webhook (Microsoft Teams)

**Navigate to:** `Administration → Media types → Create media type`

| Field | Value |
|---|---|
| Name | `Microsoft Teams` |
| Type | `Webhook` |
| Script | Use the Teams webhook script from the [Zabbix integrations repository](https://git.zabbix.com/projects/ZBX/repos/zabbix/browse/templates/media) |

Alternatively, use a simple webhook media type with a custom payload:

```javascript
var req = new HttpRequest();
req.addHeader('Content-Type: application/json');

var payload = {
    "text": "**" + value.subject + "**\n" + value.message
};

req.post(params.url, JSON.stringify(payload));
```

Parameters to define:
- `url` — your Teams incoming webhook URL

---

### 4.3 Assign Media to Users

**Navigate to:** `Administration → Users → [User] → Media → Add`

| Field | Value |
|---|---|
| Type | Select the media type (Email, Teams webhook, etc.) |
| Send to | Email address or webhook destination |
| When active | `1-7,00:00-24:00` (always) or restrict to working hours |
| Use if severity | Set minimum severity (e.g. `Warning` and above) |

---

### 4.4 Create a Trigger Action

**Navigate to:** `Configuration → Actions → Trigger actions → Create action`

**General tab:**

| Field | Value |
|---|---|
| Name | `Wazuh — Alert response` |
| Enabled | Yes |

**Conditions tab:**

| Condition | Value |
|---|---|
| Trigger severity | `≥ Warning` |
| Host group | `Wazuh Managers` |

**Operations tab — Step 1 (immediate):**

| Field | Value |
|---|---|
| Step | `1` |
| Step duration | `0` (immediate) |
| Operation type | `Send message` |
| Send to users | Select SOC team members |
| Send only to | Select messaging platform media type |

**Message template:**

```
Subject: [{TRIGGER.SEVERITY}] {HOST.NAME} — {TRIGGER.NAME}

Host:     {HOST.NAME} ({HOST.IP})
Problem:  {TRIGGER.NAME}
Severity: {TRIGGER.SEVERITY}
Time:     {EVENT.DATE} {EVENT.TIME}
Duration: {EVENT.AGE}

{TRIGGER.DESCRIPTION}

Event ID: {EVENT.ID}
Zabbix:   {TRIGGER.URL}
```

**Operations tab — Step 2 (escalation after 5 minutes if unacknowledged):**

| Field | Value |
|---|---|
| Step | `2` |
| Step duration | `300` (5 minutes) |
| Condition | `Not acknowledged` |
| Send to users | Platform team lead |
| Send only to | Email media type |

**Recovery operations tab:**

| Field | Value |
|---|---|
| Operation type | `Send message` |
| Send to users | SOC team |
| Custom message | `[RESOLVED] {HOST.NAME} — {TRIGGER.NAME} has cleared after {EVENT.DURATION}.` |

---

## Part 5 — Testing the Complete Setup

Run through the following tests before considering the monitoring setup production-ready. Document results for each test.

---

### Test 1 — Process monitoring

```bash
# Stop the manager
sudo systemctl stop wazuh-manager

# Wait 60-90 seconds, then check Monitoring → Problems in Zabbix
# Expected: "Wazuh Manager is not running" appears

# Restore
sudo systemctl start wazuh-manager

# Expected: problem clears; recovery notification delivered
```

---

### Test 2 — Alert log staleness

```bash
# Make alerts.json temporarily unwritable
sudo chmod 000 /var/ossec/logs/alerts/alerts.json

# Wait 10-11 minutes
# Expected: "no alerts generated in the last 10 minutes" trigger fires

# Restore immediately
sudo chmod 640 /var/ossec/logs/alerts/alerts.json
sudo systemctl restart wazuh-manager
```

> **Run this test in a non-production environment or during a scheduled maintenance window only.**

---

### Test 3 — ERROR log detection

```bash
# Inject a synthetic ERROR entry
echo "$(date '+%Y/%m/%d %H:%M:%S') wazuh-modulesd: ERROR - monitoring validation test entry" \
  | sudo tee -a /var/ossec/logs/ossec.log

# Wait 60 seconds
# Expected: ERROR trigger fires in Zabbix
# Check Monitoring → Latest Data → [log item] to confirm capture
```

---

### Test 4 — Notification delivery

```bash
# Test each media type directly
# Navigate to: Administration → Media types → [Media type] → Test
# Enter a test recipient and verify delivery
```

For webhook endpoints, also verify using curl:

```bash
curl -X POST "[your-webhook-url]" \
  -H "Content-Type: application/json" \
  -d '{"text": "Zabbix notification test"}'
```

If PSK encryption is enabled, verify the agent connection explicitly from the Zabbix server:

```bash
zabbix_get -s <WAZUH_HOST_IP> -k system.hostname \
  --tls-connect=psk \
  --tls-psk-identity="wazuh-server-01-psk" \
  --tls-psk-file=/etc/zabbix/zabbix_agentd.psk
```

---

### Test 5 — Heartbeat and nodata detection

```bash
# Stop the Zabbix agent
sudo systemctl stop zabbix-agent2

# Wait 5-6 minutes
# Expected: heartbeat nodata trigger fires

# Restore
sudo systemctl start zabbix-agent2
```

---

### Test checklist

| Test | Expected result | Pass / Fail | Date tested |
|---|---|---|---|
| Manager process down | Trigger fires within 90s | | |
| Alert log stale (10 min) | Trigger fires | | |
| ERROR log entry | Log item captures entry | | |
| Email notification | Email received | | |
| Webhook notification | Message received in channel | | |
| Recovery notification | Recovery message received | | |
| Agent heartbeat lost | nodata trigger fires within 5 min | | |
| PSK connection (if enabled) | `zabbix_get` with PSK flags succeeds | | |

---

## Part 6 — Scaling to Multiple Hosts

### 6.1 Create a Wazuh Manager template

**Navigate to:** `Configuration → Templates → Create template`

| Field | Value |
|---|---|
| Template name | `Template Wazuh Manager` |
| Groups | `Templates/Security` |

Within the template, recreate all items and triggers from Parts 2 and 3 above — replacing the host-specific references with template-relative references (Zabbix handles this automatically in the template context).

**Add template-level macros:**

`Configuration → Templates → [Template] → Macros`

| Macro | Default value | Purpose |
|---|---|---|
| `{$WAZUH_ALERT_LOG}` | `/var/ossec/logs/alerts/alerts.json` | Alert log path |
| `{$WAZUH_LOG}` | `/var/ossec/logs/ossec.log` | Main log path |
| `{$DISK_WARN}` | `85` | Disk warning threshold (%) |
| `{$DISK_CRIT}` | `95` | Disk critical threshold (%) |
| `{$CPU_WARN}` | `90` | CPU warning threshold (%) |
| `{$ALERT_STALE_SEC}` | `600` | Alert log staleness threshold (seconds) |

Update item keys and trigger expressions to use macros:

```
# Item key
vfs.file.time[{$WAZUH_ALERT_LOG},modify]

# Trigger expression
(now()-last(/Template Wazuh Manager/vfs.file.time[{$WAZUH_ALERT_LOG},modify]))>{$ALERT_STALE_SEC}
```

---

### 6.2 Link template to a host

**Navigate to:** `Configuration → Hosts → [Host] → Templates`

Click **Select** and choose `Template Wazuh Manager`.

Click **Update**.

All items and triggers from the template are immediately applied to the host. Override any macro at the host level if the default value does not apply:

`Configuration → Hosts → [Host] → Macros → Add`

---

### 6.3 Create additional templates

Following the same approach, create:

| Template | Key items |
|---|---|
| `Template Wazuh Indexer` | Cluster health (HTTP agent), JVM heap, unassigned shards, disk, CPU |
| `Template Wazuh Dashboard` | Process count, HTTP endpoint availability |

---

### 6.4 Wazuh Indexer — cluster health item (HTTP agent)

**Navigate to:** `Configuration → Hosts → [Indexer Host] → Items → Create item`

| Field | Value |
|---|---|
| Name | `Wazuh Indexer — cluster health state` |
| Type | `HTTP agent` |
| URL | `https://localhost:9200/_cluster/health` |
| Request method | `GET` |
| Authentication | `Basic` |
| User | `admin` (or your indexer admin user) |
| Password | `[indexer admin password]` |
| SSL verify peer | `No` |
| Type of information | `Text` |
| Update interval | `60s` |

> **SSL note:** `SSL verify peer: No` is appropriate for internal deployments using self-signed certificates. If your indexer uses a certificate issued by a trusted CA, set this to `Yes` for stronger security.

**Preprocessing:**

| Step | Type | Parameters |
|---|---|---|
| 1 | JSONPath | `$.status` |

**Triggers:**

```
# Degraded
last(/[Indexer Host]/http.agent[https://localhost:9200/_cluster/health])="yellow"
Severity: Warning

# Critical
last(/[Indexer Host]/http.agent[https://localhost:9200/_cluster/health])="red"
Severity: High
```

---

### 6.5 Wazuh Dashboard — HTTP endpoint item

| Field | Value |
|---|---|
| Name | `Wazuh Dashboard — HTTP endpoint reachable` |
| Type | `HTTP agent` |
| URL | `https://[dashboard-host]:443` |
| Request method | `HEAD` |
| Expected HTTP status codes | `200,301,302` |
| SSL verify peer | `No` |
| Update interval | `60s` |

> **SSL note:** As with the indexer item, set `SSL verify peer: Yes` if a valid CA-issued certificate is in use.

**Trigger:**

```
last(/[Dashboard Host]/http.agent[https://[dashboard-host]:443])<>200
Severity: High
```

---

### 6.6 Host groups

**Navigate to:** `Configuration → Host groups → Create host group`

Create the following groups and assign hosts accordingly:

| Group name | Assign |
|---|---|
| `Wazuh Managers` | All manager nodes |
| `Wazuh Indexers` | All indexer nodes |
| `Wazuh Dashboard` | Dashboard servers |
| `Security Infrastructure` | All of the above |

Use these groups as conditions in trigger Actions to route alerts to the correct team.

---

## Part 7 — Routine Maintenance Schedule

Use this schedule as a baseline. Adjust frequencies based on your environment's stability and alert volume.

| Task | How | Frequency |
|---|---|---|
| Review triggered alerts | `Monitoring → Problems` — look for noise, patterns, unacknowledged items | Weekly |
| Test notification delivery | `Administration → Media types → Test` each channel | Monthly |
| Simulate a process failure | `systemctl stop wazuh-manager` — verify trigger and notification | Monthly |
| Review and adjust thresholds | Compare trigger history against operational reality | Monthly |
| Verify item data collection | `Monitoring → Latest Data` — check for "Not Supported" items | Monthly |
| Review acknowledgment patterns | `Monitoring → Problems` — look for acknowledged-but-not-resolved | Monthly |
| Verify PSK connectivity (if enabled) | Run `zabbix_get` with PSK flags from Zabbix server to each host | Monthly |
| Remove or disable unused checks | Audit all items; disable any that have not triggered in 90 days and are not baseline checks | Quarterly |
| Update templates after Wazuh upgrades | Verify log paths, process names, and API endpoints after version updates | After each upgrade |
| Review ownership documentation | Confirm all alerts have current, correct owners | After team changes |

---

## Part 8 — Quick Reference

### Key file paths

| File | Purpose |
|---|---|
| `/var/ossec/logs/alerts/alerts.json` | Wazuh alert output — primary pipeline health signal |
| `/var/ossec/logs/ossec.log` | Wazuh main log — ERROR/CRITICAL patterns |
| `/etc/zabbix/zabbix_agentd.psk` | PSK key file — readable by `zabbix` user only |
| `/var/log/zabbix/zabbix_agent2.log` | Zabbix agent log — troubleshoot item collection failures |
| `/var/log/zabbix/zabbix_server.log` | Zabbix server log — troubleshoot notification failures |
| `/etc/zabbix/zabbix_agent2.conf` | Zabbix agent configuration |

### Key Zabbix navigation paths

| Task | Path |
|---|---|
| Create item | `Configuration → Hosts → [Host] → Items → Create item` |
| Create trigger | `Configuration → Hosts → [Host] → Triggers → Create trigger` |
| Create action | `Configuration → Actions → Trigger actions → Create action` |
| Configure media type | `Administration → Media types` |
| Assign media to user | `Administration → Users → [User] → Media` |
| Configure host encryption | `Configuration → Hosts → [Host] → Encryption` |
| View problems | `Monitoring → Problems` |
| View latest data | `Monitoring → Latest Data` |
| Test media type | `Administration → Media types → [Type] → Test` |
| Create template | `Configuration → Templates → Create template` |
| Link template to host | `Configuration → Hosts → [Host] → Templates` |
| Create maintenance window | `Configuration → Maintenance → Create maintenance period` |

### Useful diagnostic commands

```bash
# Check Zabbix agent status
systemctl status zabbix-agent2

# Test agent connectivity (unencrypted)
zabbix_agent2 -t system.hostname

# Test agent connectivity (PSK encrypted)
zabbix_get -s <WAZUH_HOST_IP> -k system.hostname \
  --tls-connect=psk \
  --tls-psk-identity="wazuh-server-01-psk" \
  --tls-psk-file=/etc/zabbix/zabbix_agentd.psk

# Check agent log for collection or TLS errors
tail -100 /var/log/zabbix/zabbix_agent2.log | grep -i "error\|tls\|psk"

# Verify alert log is being written
ls -la /var/ossec/logs/alerts/alerts.json

# Check Wazuh manager process
ps aux | grep wazuh-manager | grep -v grep

# Check Wazuh indexer cluster health
curl -sk -u admin:[password] https://localhost:9200/_cluster/health | python3 -m json.tool

# Check disk usage on key paths
df -h /var/ossec /var /

# Check Zabbix server log for notification errors
grep -i "media\|alert\|error" /var/log/zabbix/zabbix_server.log | tail -50
```

### Severity reference

| Severity | Meaning | Example | Response |
|---|---|---|---|
| Information | No action needed | Planned restart | Log only |
| Warning | Investigate soon | Disk > 85%; CPU spike | Next business day |
| High | Investigate promptly | Alert log stale; Filebeat down | Within 30 minutes |
| Disaster | Immediate response | Manager process down | Immediately |

### Trigger expression reference

| Pattern | Expression | Use case |
|---|---|---|
| Process down | `last(/host/proc.num[name])=0` | Service stopped |
| File not updated | `(now()-last(/host/vfs.file.time[path,modify]))>N` | Pipeline stall detection |
| Disk threshold | `last(/host/vfs.fs.size[/,pused])>N` | Capacity warning |
| Sustained CPU | `min(/host/system.cpu.util[,avg1],5m)>N` | Performance degradation |
| Log pattern match | `count(/host/log[path,"PATTERN"],5m)>0` | Error detection |
| No data received | `nodata(/host/item.key,300)=1` | Agent or collection failure |
| Percentage drop | `(last(/host/metric)/avg(/host/metric,1h))<0.8` | Sudden relative drop |

---

*[← Series Overview](./README.md) · [Part 1: Why Monitor Wazuh with Zabbix](./Part-1_Why-Monitor-Wazuh-with-Zabbix.md)*

---

*Published by [SECaaS.IT](https://security-as-a-service.io/) as part of the Wazuh Ambassador programme.*
