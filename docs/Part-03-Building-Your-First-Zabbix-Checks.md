# Part 3 - Building Your First Zabbix Checks

*Part of the series: [Monitoring Wazuh with Zabbix](./README.md)*

---

## Introduction - From Design to Implementation

In Parts 1 and 2, we established the foundation:

- Wazuh detects threats
- Zabbix ensures that detection actually works
- The real risk is silent failure - a system that appears healthy while detection has stopped

We also introduced the **Visibility Assurance Model** and a key insight:

> **A running service does not guarantee a working detection pipeline.**

Now it is time to move from design to practice.

This article focuses on **building your first meaningful Zabbix checks for Wazuh** - not just collecting metrics, but creating operationally relevant signals that tell you when something is wrong.

---

## Prerequisite - Zabbix Agent on the Wazuh Host

Before creating monitoring items in Zabbix, the **Zabbix agent must be installed and running on your Wazuh server**.

The agent is the component that collects data from the monitored host and sends it to the Zabbix server. Without it, none of the items in this article will function.

**Quick check - verify the agent is running:**

```bash
systemctl status zabbix-agent2
```

**If not installed**, refer to the official Zabbix documentation for your distribution:
[https://www.zabbix.com/documentation/current/manual/installation/install_from_packages](https://www.zabbix.com/documentation/current/manual/installation/install_from_packages)

**Two important configuration points:**

1. The Zabbix agent configuration file (`/etc/zabbix/zabbix_agent2.conf`) must have the `Server` and `ServerActive` directives pointing to your Zabbix server IP or hostname.
2. Log file monitoring (used in this article) requires the agent to have **read permission** on the monitored log files. On a default Wazuh installation, Wazuh logs at `/var/ossec/logs/` are owned by the `wazuh` user. You may need to add the `zabbix` user to the `wazuh` group, or adjust permissions accordingly.

```bash
# Example: add zabbix user to wazuh group
usermod -aG wazuh zabbix
systemctl restart zabbix-agent2
```

Once the agent is running and your Wazuh host is added to Zabbix under **Configuration → Hosts**, you are ready to build checks.

---

## What a Good Check Looks Like

Before creating items in Zabbix, it helps to be clear about what we are actually building.

A check is not just a metric. A good check answers an operationally meaningful question:

- "Is Wazuh processing events right now?"
- "Has the alert pipeline stalled?"
- "Is the disk about to become a problem?"

### The four layers of a complete check

| Layer | Purpose | Example |
|-|-|-|
| Metric | Raw data collection | `wazuh-manager` process count |
| Logic | Interpretation of the data | Count = 0 means the service is down |
| Alert | Action trigger | Notify SOC if down for more than 2 minutes |
| Response | Defined human or automated action | Restart the service; escalate if unresolved |

> **Metric collection alone is not monitoring.**  
> Monitoring only exists when all four layers are in place.

### Apply the decision framework from Part 2

Every check should satisfy at least one of these:

- Detects complete failure
- Detects silent degradation
- Predicts future failure

If the answer is none of these, the check is likely noise.

---

## Zabbix Monitoring Components

Zabbix monitoring is built from two core components. Understanding both before you start will make the rest of this article much clearer.

### Items

An **item** is a data point that Zabbix collects from a monitored host. Items define *what* is collected and *how often*.

Examples:
- Number of running processes matching a name
- Last modification timestamp of a file
- Percentage of disk space used
- Log entries matching a pattern

### Triggers

A **trigger** evaluates item data and determines whether a problem exists. When a trigger condition becomes true, Zabbix generates a problem event - which can then fire notifications.

> Items collect the data.  
> Triggers decide whether the data indicates a problem.

In Zabbix, navigate to items and triggers via:

```
Configuration → Hosts → [Your Wazuh Host] → Items
Configuration → Hosts → [Your Wazuh Host] → Triggers
```

---

## Building the Monitoring Items

We will now create five items covering the three layers of the Visibility Assurance Model.

---

### Item 1 - Wazuh Manager Process (Platform Health)

The Wazuh manager is the core component responsible for processing events and generating alerts. If it stops, detection stops completely.

**Navigate to:** `Configuration → Hosts → [Wazuh Host] → Items → Create Item`

| Field | Value |
|-|-|
| Name | `Wazuh Manager - process count` |
| Type | `Zabbix agent` |
| Key | `proc.num[wazuh-manager]` |
| Type of information | `Numeric (unsigned)` |
| Units | *(leave blank)* |
| Update interval | `60s` |
| Description | Returns the number of running processes named wazuh-manager. Expected value: 1 or more. |

**What this returns:**

- `1` or higher → process is running
- `0` → process has stopped; event processing has halted

---

### Item 2 - Alert Log Freshness (Data Flow Integrity)

This is the most important item in this set. It monitors whether Wazuh is actively generating alerts - not just whether the service appears to be running.

Wazuh writes every alert to:

```
/var/ossec/logs/alerts/alerts.json
```

If this file's modification timestamp stops advancing, something in the detection pipeline has failed.

| Field | Value |
|-|-|
| Name | `Wazuh - alerts.json last modification time` |
| Type | `Zabbix agent` |
| Key | `vfs.file.time[/var/ossec/logs/alerts/alerts.json,modify]` |
| Type of information | `Numeric (unsigned)` |
| Units | `unixtime` |
| Update interval | `60s` |
| Description | Returns the Unix timestamp of the last modification to alerts.json. Used to detect pipeline stalls. |

**What this returns:**

A Unix timestamp. Zabbix will display this as a human-readable date. If the value stops updating while the system is active, the detection pipeline has stalled.

---

### Item 3 - ERROR Entries in ossec.log (Platform Health / Data Flow)

The main Wazuh log file frequently contains early indicators of problems before they cause visible failures.

```
/var/ossec/logs/ossec.log
```

This item uses **active log monitoring**, which requires the Zabbix agent to be configured for active checks (`ServerActive` in `zabbix_agent2.conf`).

| Field | Value |
|-|-|
| Name | `Wazuh - ERROR entries in ossec.log` |
| Type | `Zabbix agent (active)` |
| Key | `log[/var/ossec/logs/ossec.log,"ERROR"]` |
| Type of information | `Log` |
| Update interval | `60s` |
| Description | Captures log lines containing ERROR from ossec.log. Each matching line is stored as a separate log entry in Zabbix. |

**Extension:** You can create additional items using the same pattern to capture `CRITICAL` or `failed` entries, giving you layered visibility into Wazuh's internal state.

---

### Item 4 - Disk Usage (Infrastructure Capacity)

Disk space exhaustion is one of the most common causes of monitoring failures. When the disk fills, Wazuh cannot write logs or alerts, and services may eventually crash.

| Field | Value |
|-|-|
| Name | `Disk usage on /` |
| Type | `Zabbix agent` |
| Key | `vfs.fs.size[/,pused]` |
| Type of information | `Numeric (float)` |
| Units | `%` |
| Update interval | `60s` |
| Description | Percentage of disk space used on the root filesystem. Adjust the path if Wazuh data is on a separate mount (e.g. /var). |

**Note:** If your Wazuh installation uses a dedicated partition for `/var/ossec` or `/var`, create a second item monitoring that partition specifically. Disk exhaustion on a secondary mount will not be caught by monitoring only `/`.

---

### Item 5 - CPU Utilisation (Infrastructure Capacity)

Sustained high CPU load can delay event processing, increase alert latency, and eventually cause pipeline stalls that look exactly like silent failures.

| Field | Value |
|-|-|
| Name | `CPU utilisation` |
| Type | `Zabbix agent` |
| Key | `system.cpu.util[,avg1]` |
| Type of information | `Numeric (float)` |
| Units | `%` |
| Update interval | `60s` |
| Description | Average CPU utilisation over the last 1 minute. |

---

## Defining Triggers - Making Data Actionable

With items collecting data, we now define triggers that determine when a condition represents a problem.

**Navigate to:** `Configuration → Hosts → [Wazuh Host] → Triggers → Create Trigger`

---

### Trigger 1 - Wazuh Manager Process Down

| Field | Value |
|-|-|
| Name | `Wazuh Manager is not running` |
| Expression | `last(/[Wazuh Host]/proc.num[wazuh-manager])=0` |
| Severity | `High` |
| Description | The wazuh-manager process has stopped. Security events are no longer being processed. Restart the service immediately and investigate the cause. |
| Manual close | Enabled |

**Design note:** Severity is set to `High` rather than `Disaster` because a brief process restart (e.g. during a software update) should not trigger the highest-priority response. If your environment requires immediate paging for any downtime, increase to `Disaster`.

---

### Trigger 2 - Alert Log Stale (Silent Failure Detection)

This trigger detects the silent failure scenario described in Part 2 - the pipeline has stopped generating alerts without any process crash.

| Field | Value |
|-|-|
| Name | `Wazuh - no alerts generated in the last 10 minutes` |
| Expression | `(now()-last(/[Wazuh Host]/vfs.file.time[/var/ossec/logs/alerts/alerts.json,modify]))>600` |
| Severity | `High` |
| Description | The alerts.json file has not been updated for 10 minutes. Possible causes: manager crash, pipeline stall, disk write failure, or resource exhaustion. Investigate immediately - detection may have stopped. |
| Manual close | Enabled |

**Threshold guidance:** 10 minutes is a reasonable starting point for most environments. In high-volume deployments where alerts are generated continuously, you may reduce this to 5 minutes. In low-volume environments (e.g. small lab setups with few agents), consider increasing to 15-20 minutes to avoid false positives during quiet periods.

---

### Trigger 3 - ERROR Messages in Log

| Field | Value |
|-|-|
| Name | `Wazuh - ERROR entries detected in ossec.log` |
| Expression | `count(/[Wazuh Host]/log[/var/ossec/logs/ossec.log,"ERROR"],5m)>0` |
| Severity | `Warning` |
| Description | One or more ERROR-level messages have appeared in ossec.log in the last 5 minutes. Review the log entries to determine whether this indicates a recoverable condition or an escalating problem. |
| Manual close | Enabled |

**Design note:** This trigger is set to `Warning` because ERROR entries in ossec.log range in severity from minor (a temporary connection issue) to significant (a component failure). Use the Zabbix log viewer to review the actual entries before escalating. If specific ERROR patterns are consistently critical in your environment, create dedicated triggers for those patterns.

---

### Trigger 4 - Low Disk Space

| Field | Value |
|-|-|
| Name | `Disk usage above 85% on /` |
| Expression | `last(/[Wazuh Host]/vfs.fs.size[/,pused])>85` |
| Severity | `Warning` |
| Description | Disk usage has exceeded 85%. If this continues, Wazuh will be unable to write logs or alerts. Free disk space or extend the filesystem before the situation becomes critical. |
| Manual close | Enabled |

**Consider adding a second, higher-severity trigger:**

| Field | Value |
|-|-|
| Name | `Disk usage critical - above 95% on /` |
| Expression | `last(/[Wazuh Host]/vfs.fs.size[/,pused])>95` |
| Severity | `High` |
| Description | Disk usage is critical. Wazuh log writes may fail imminently. Immediate action required. |

Using two thresholds (warning + critical) gives operators time to act before the situation becomes an outage.

---

### Trigger 5 - Sustained High CPU

| Field | Value |
|-|-|
| Name | `CPU utilisation above 90% for 5 minutes` |
| Expression | `min(/[Wazuh Host]/system.cpu.util[,avg1],5m)>90` |
| Severity | `Warning` |
| Description | CPU utilisation has remained above 90% for at least 5 minutes. This may delay event processing and increase alert latency. Investigate the cause - check for runaway processes or unusual event volume. |
| Manual close | Enabled |

**Design note:** The `min()` function over a 5-minute window means the trigger only fires if CPU has been consistently above 90% for the entire period - not just for one polling interval. This eliminates false positives from brief spikes and ensures alerts represent genuine sustained load.

---

## Defining Expected Responses

A trigger without a defined response is incomplete monitoring. For each trigger, you should document who owns it and what action is expected.

| Trigger | Owner | Expected action |
|-|-|-|
| Manager not running | Platform / SOC team | Restart `wazuh-manager`; investigate root cause; escalate if recurrence |
| Alert log stale | SOC team | Investigate pipeline; check disk, process state, and `ossec.log` for errors |
| ERROR in ossec.log | SOC / Ops team | Review log entries; determine if escalation is warranted |
| Disk > 85% | Operations team | Free space or extend filesystem before reaching 95% |
| CPU > 90% sustained | Operations team | Identify high-load process; investigate event volume spikes |

Without clearly defined ownership, alerts may be seen but not acted upon.

---

## Testing Your Monitoring Setup

Before relying on monitoring in production, verify that everything behaves correctly. Testing is not optional - small configuration errors often only appear under real conditions.

### Step 1 - Verify item data collection

Navigate to `Monitoring → Latest Data` and filter by your Wazuh host.

Confirm that:
- All five items are present
- Values are updating at the expected interval
- The `alerts.json` timestamp reflects a recent time
- Disk and CPU values are plausible

If an item shows **"Not supported"**, the agent cannot collect that data. Check the agent log for details:

```bash
tail -f /var/log/zabbix/zabbix_agent2.log
```

Common causes: incorrect key, file not found, permission denied.

---

### Step 2 - Simulate a process failure

Stop the Wazuh manager and confirm the trigger fires:

```bash
sudo systemctl stop wazuh-manager
```

In Zabbix, navigate to `Monitoring → Problems` and verify that the "Wazuh Manager is not running" trigger appears within 60-90 seconds.

Restore the service and verify the problem clears:

```bash
sudo systemctl start wazuh-manager
```

---

### Step 3 - Simulate alert log staleness

You can simulate a stale alert log by temporarily making the file unwritable, which prevents Wazuh from updating it:

```bash
sudo chmod 000 /var/ossec/logs/alerts/alerts.json
```

Wait 10-11 minutes. Verify the "no alerts generated" trigger fires in Zabbix.

Restore permissions immediately:

```bash
sudo chmod 640 /var/ossec/logs/alerts/alerts.json
sudo systemctl restart wazuh-manager
```

**Important:** This test should only be run in a non-production environment or during a scheduled maintenance window.

---

### Step 4 - Inject a test ERROR entry

You can write a synthetic ERROR line to `ossec.log` to test the log monitoring item:

```bash
echo "$(date '+%Y/%m/%d %H:%M:%S') wazuh-modulesd: ERROR - test error entry for monitoring validation" | sudo tee -a /var/ossec/logs/ossec.log
```

Verify within 60 seconds that the ERROR trigger fires in Zabbix. Check `Monitoring → Problems` and `Monitoring → Latest Data → [log item]` to confirm the entry was captured.

---

## Practical Configuration Tips

**Keep item names descriptive and consistent.**  
Use a naming convention such as `Wazuh - [what it measures]`. This makes filtering and troubleshooting significantly easier as your monitoring grows.

**Do not over-poll.**  
Most checks work well at 60-second intervals. More frequent polling increases agent and server load without meaningfully improving detection speed for the failure scenarios we are monitoring.

**Start with these five items and get them right.**  
It is better to have five well-understood, well-tested checks than twenty checks of uncertain quality. Add new items when real operational experience identifies a gap - not because a metric is available.

**Document your trigger logic.**  
For each trigger, record why it exists, what threshold was chosen, and what the expected response is. This documentation becomes invaluable during incidents when there is no time to reconstruct the reasoning.

---

## Summary

In this article, you built a functional monitoring foundation for Wazuh in Zabbix:

- Five items covering all three layers of the Visibility Assurance Model
- Five triggers that turn raw data into actionable alerts
- Defined response expectations for each trigger
- A tested, verified configuration you can trust

The monitoring items and triggers in this article form a complete, minimal baseline. They are designed to catch the failure scenarios that matter most - process failures, silent pipeline stalls, capacity exhaustion - without generating noise.

> **Detection without monitoring is fragile.  
> Monitoring without tested triggers is wishful thinking.**

---

## Looking Ahead

Your Zabbix instance is now collecting data and generating alerts when defined conditions occur. But monitoring only becomes operationally effective when alerts reliably reach the right people and lead to action.

In Part 4, we focus on the alerting layer:

- designing meaningful severity levels
- configuring notification channels
- building escalation paths
- ensuring alert delivery reliability - including what happens when notification channels themselves fail

> **A monitoring system that generates alerts but cannot reliably deliver them is indistinguishable from no monitoring at all.**

---

*[← Part 2: Designing Effective Monitoring for Wazuh](./Part-02-Designing-Effective-Monitoring-for-Wazuh.md) · [Part 4 → From Metrics to Action - Alerting Strategies](./Part-04-From-Metrics-to-Action.md)*
