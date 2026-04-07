# Monitoring Wazuh with Zabbix

**A six-part article series by [SECaaS.IT](https://security-as-a-service.io/)**  
*Published as part of the Wazuh Ambassadors Program*

---

## Core Principle

> **Wazuh detects threats.**  
> **Zabbix ensures the monitoring platform itself is reliable.**

Security monitoring systems fail in two ways: loudly, or silently. Loud failures are visible. Silent failures are not — and they are far more dangerous.

This series addresses one of the most consistently overlooked problems in security operations:

> **The real risk is not an attack. It is losing the ability to detect attacks.**

---

## What This Series Covers

This series explains how and why to monitor Wazuh with Zabbix from an operational and SOC-oriented perspective. It is structured around a single extended argument:

- Wazuh is a powerful threat detection platform — but only when it is working.
- Infrastructure monitoring is not optional for security teams. It is a fundamental control.
- Visibility must be actively verified, not assumed.

The series builds a complete operational model across six articles:

> **Detection → Monitoring → Alerting → Response → Maintenance**

Each article adds one layer. By the end, you have a framework for reliable, trustworthy security visibility — not just a monitoring setup.

---

## The Visibility Assurance Model

Throughout this series, monitoring is structured around three layers:

| Layer | Question it answers | Priority |
|-|-|-|
| Platform Health | Is Wazuh running? | Critical |
| Data Flow Integrity | Is Wazuh actively detecting? | High |
| Infrastructure Capacity | Can Wazuh continue operating? | High |

**Data Flow Integrity is the most important layer.** A running process does not guarantee a working detection pipeline. This distinction is the conceptual foundation of the entire series.

---

## Articles in This Series

| File                                                                           | Title                                               | Focus                                                                                     |
| -------------------------- | ----------------- | ------------------------------- |
| [Part 1](./Part-01-Why-Monitor-Wazuh-with-Zabbix.md)            | Why Monitor Wazuh with Zabbix                       | Silent failure, loss of visibility, the case for infrastructure monitoring                |
| [Part 2](./Part-02-Designing-Effective-Monitoring-for-Wazuh.md) | Designing Effective Monitoring for Wazuh            | The Visibility Assurance Model, three monitoring layers, designing from failure scenarios |
| [Part 3](./Part-03-Building-Your-First-Zabbix-Checks.md)        | Building Your First Zabbix Checks                   | Metric collection, trigger logic, alerting separation, signal quality                     |
| [Part 4](./Part-04-From-Metrics-to-Action.md)                     | From Metrics to Action — Alerting Strategies        | Severity design, notification reliability, escalation, alert delivery failure             |
| [Part 5](./Part-05-Operating-and-Maintaining-Monitoring.md)       | Operating and Maintaining Monitoring                | Signal quality, monitoring the monitoring system, ownership, continuous improvement       |
| [Part 6](./Part-06-From-Monitoring-to-Trust.md)                   | From Monitoring to Trust — Operating Wazuh at Scale | Distributed architectures, end-to-end pipeline validation, partial failure, SOC maturity  |

---

## Companion Implementation Reference

This series focuses on **why** and **how to design** monitoring for Wazuh.

For practitioners who want step-by-step implementation detail — Zabbix navigation paths, complete item configurations, trigger expressions, notification setup, and scaling procedures — a condensed companion reference is available:

📄 **[Companion-Reference](./Companion-Reference.md)**

The companion reference is designed to be used alongside the article series, not as a replacement for it. The series provides the reasoning; the reference provides the instructions.

---

## Who This Is For

- Security engineers operating Wazuh deployments
- SOC teams who need reliable early-warning indicators for monitoring failures
- System administrators responsible for security platform uptime
- DevOps and platform engineers integrating monitoring into security operations

Basic Linux and monitoring familiarity is assumed. No deep Zabbix expertise is required — the companion reference guides you through setup from scratch.

---

## Scope

The series uses a **single-node Wazuh deployment** as its baseline for all examples. Distributed and multi-node environments are addressed in Part 6.

Unless otherwise stated, examples assume:
- Wazuh installed at `/var/ossec/`
- Zabbix agent installed on the Wazuh host
- Zabbix server reachable from the monitored host

---

## About SECaaS.IT

[SECaaS.IT](https://security-as-a-service.io/) provides Security-as-a-Service for organisations that need enterprise-grade security operations without building everything in-house. This series reflects operational patterns and lessons from real Wazuh deployments.

*This content is published as part of the Wazuh Ambassador programme. No commercial endorsement is implied.*

---

## Licence

This documentation is published for educational and community use. Attribution appreciated.
