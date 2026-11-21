## 🏗️ Architecture Overview: Argus Hierarchical XDR
<img width="2000" height="1454" alt="argus-architecture" src="https://github.com/user-attachments/assets/06fe6604-61c4-459b-b3e2-94502403b23a" />


The **Argus** project is designed as a True Distributed Enterprise Architecture (Federated SOC). It transforms a traditional centralized SIEM into a highly scalable, hierarchical XDR system.

The architecture is divided into three main layers, separating data ingestion and local storage (Heavy Lifting) from orchestration, threat intelligence enrichment, and analysis (Command & Control).

### 1. 🏢 Remote Clusters Layer (Endpoint & Network Clusters)
This layer operates independently within each cluster, handling log ingestion, rule evaluation/MITRE mapping, and local indexing to optimize bandwidth and ensure locally stored data remains intact and searchable during WAN outages.

It is composed of two types of Wazuh clusters, each joined to the HQ through Cross-Cluster Search (CCS):

- **Endpoint Clusters:** One per branch. Collect logs and security events from endpoints through Wazuh agents.
- **Network Cluster:** A dedicated cluster for network and core devices (switches, routers, firewalls). Collects logs through syslog, keeping network telemetry separated from endpoint data with its own sizing and retention.

Each cluster runs the following components:

- **Wazuh Manager:** Receives raw logs and security events from endpoints (via Wazuh agents) and from network/core devices (via syslog), evaluates rules, maps MITRE ATT&CK techniques, and forwards alerts/events to the local Wazuh Indexer and HQ automation.
- **Wazuh Indexer (Data Node):** Stores and indexes security alerts and events locally within each branch (`wazuh-alerts-*`), preserving data locality and retention policies per cluster.
- **Telegraf (Health Agent):** Installed on branch servers to monitor host and hardware telemetry (CPU, RAM, disk, network) and forward metrics to the Headquarters. It complements, and does not replace, Wazuh's own agent health monitoring.

### 2. 🏛️ Headquarters Layer (HQ - Command & Control)
The HQ is the brain of the operation. It hosts visualization, SOAR orchestration, case management, threat intelligence, and observability tools, providing a single pane of glass while keeping branch logs stored locally at their source clusters.

- **Grafana (Global Visibility):** Connected to the HQ Wazuh Indexer / Cross-Cluster Search (CCS) setup to query and display unified security dashboards across remote branch clusters alongside infrastructure telemetry.
- **Shuffle (SOAR):**
  - **Implementation:** The Shuffle Webhook URL is configured inside `ossec.conf` on every remote Wazuh Manager over HTTPS, filtered by alert `<level>` (e.g., 10+) so only high-severity alerts are forwarded.
  - **Result:** When a branch detects an attack, it pushes alerts to Shuffle in the HQ for automated triage, Threat Intelligence enrichment, and playbook execution.
- **TheHive & Cortex (Incident Management):**
  - **Implementation:** When Shuffle processes a high-severity or confirmed malicious alert, it automatically opens a detailed Case in TheHive for SOC investigation and triggers Cortex analyzers/responders when needed.
- **MISP & OpenCTI (Threat Intelligence):**
  - **Implementation:** Hosted centrally in the HQ. Shuffle (and/or Wazuh integrations/CDB lists) queries MISP and OpenCTI to enrich indicators (IPs, hashes, domains) during triage and incident response.
- **InfluxDB & Uptime Kuma (Monitoring):**
  - **Implementation:** Uptime Kuma monitors service/API/dashboard availability across nodes and sends Telegram alerts on outages. InfluxDB ingests Telegraf telemetry from the branches for visualization in Grafana.

### 3. 🕵️‍♂️ DFIR Layer: Velociraptor (Isolation & Investigation)
To achieve full XDR capabilities, Velociraptor is integrated for Digital Forensics and Incident Response (DFIR) across all distributed branches.

- **Velociraptor Server:** Hosted centrally in the HQ.
- **Velociraptor Agents:** Deployed on endpoint devices across remote branches to collect forensic artifacts and execute containment actions on demand.

---

## ⚡ Automated Response Workflow (Argus in Action)

1. A malicious attack occurs in **Branch B**.
2. **Wazuh Manager B** detects the threat, indexes the alert locally in **Branch B's Wazuh Indexer**, and pushes the alert to **Shuffle** in the HQ.
3. **Shuffle** parses the alert, enriches indicators via **MISP / OpenCTI**, and automatically opens a Case in **TheHive**.
4. The SOC Analyst reviews the Case in **TheHive** and triggers the **Cortex** responder.
5. **Cortex** communicates with the **Velociraptor Server** in the HQ.
6. The **Velociraptor Server** queues a **Network Quarantine** and **Memory Dump** collection for the affected endpoint in **Branch B**, delivered to the **Velociraptor Agent** on its next check-in, for forensic analysis.

---

## 🔐 Security Considerations

- **Transport security:** All Wazuh components (Manager, Indexer, Dashboard) and cross-cluster communication are secured with TLS using certificates generated by `wazuh-certs-tool`, with a shared trusted CA across clusters.
- **Integrations:** Inter-service integrations use HTTPS with API keys/tokens stored as environment secrets.
- **Network:** In production, branch-to-HQ traffic is expected to run over a site-to-site VPN (e.g., WireGuard).
- **Syslog hardening:** Because syslog has no built-in authentication or encryption, the Network Cluster's Wazuh Manager accepts syslog only from approved device IPs (`allowed-ips` in the `<remote>` block), and network devices send logs over a dedicated management network/VLAN isolated from user traffic.

---

## ⚠️ Limitations & Future Work

- **HQ as a single point of failure:** Orchestration is centralized in the HQ. Future work includes High Availability (HA) and a Disaster Recovery (DR) site.
- **Alert delivery during HQ outages:** Alerts destined for Shuffle may be delayed or lost if the HQ is unreachable. Future work includes a queue/retry mechanism.
- **Partial results in CCS:** If a branch cluster is unavailable, Cross-Cluster Search returns partial results from the remaining clusters.
- **Containment scope:** DFIR containment through Velociraptor covers endpoints only. Automated isolation of network devices (e.g., Shuffle disabling a switch port via SSH/API) is planned as future work.

---

## 🚀 Executive Summary

**Argus** is a distributed enterprise XDR architecture that decouples local collection and storage from centralized orchestration:

- **Remote Clusters (Endpoint Branches & Network Cluster):** Handle endpoint collection (agents) and network device collection (syslog), rule/MITRE analysis via **Wazuh Manager**, local storage via **Wazuh Indexer**, and host telemetry via **Telegraf**.
- **Headquarters (HQ):** Handles global querying and visualization via **Cross-Cluster Search (CCS) & Grafana**, automated triage and TI enrichment via **Shuffle + MISP/OpenCTI**, case management and response via **TheHive, Cortex & Velociraptor**, and availability/metrics monitoring via **Uptime Kuma & InfluxDB**.

*Argus differentiates itself from a standard Wazuh deployment by combining federated multi-cluster search, per-branch data sovereignty, and an integrated SOAR-to-DFIR response chain.*
