# wazuh-documents
## 🏗️ Architecture Overview: Argus Hierarchical XDR

The **Argus** project is designed as a **True Distributed Enterprise Architecture** (Federated SOC). It transforms a traditional centralized SIEM into a highly scalable, hierarchical XDR system. 

The architecture is divided into three main layers, separating data ingestion (Heavy Lifting) from orchestration and analysis (Command & Control).

---

### 1. 🏢 Remote Clusters Layer (Collection & Processing)
This layer operates independently within each branch, handling log ingestion, normalization, and local storage to optimize bandwidth and ensure zero data loss during network outages.

*   **Wazuh Manager:** Receives raw logs and security events directly from the local branch devices (Endpoints).
*   **Graylog (The Bridge):** Acts as the core processing engine within each branch. 
    *   *Implementation:* The Wazuh Manager forwards logs to Graylog. Graylog then performs **Normalization** (standardizing field names across all log sources) before passing them on.
*   **Wazuh Indexer (Data Node):** After cleaning the logs, Graylog forwards them to the local Indexer (hosted on the same server) to be stored locally.
*   **Telegraf (Health Agent):** Installed on branch servers to monitor hardware metrics (CPU, RAM) and forward this telemetry to the Headquarters.

---

### 2. 🏛️ Headquarters Layer (HQ - Node 1) - Command & Control
The HQ is the brain of the operation. It hosts all Visualization, Automation, and Case Management tools, providing a single pane of glass without storing branch logs locally.

*   **Grafana (Global Visibility):** 
    *   *Implementation:* Connected to the **Central Wazuh Indexer**. Utilizing **Cross-Cluster Search (CCS)**, Grafana seamlessly displays a unified dashboard containing data from all branches simultaneously—without communicating directly with the remote branches.
*   **Shuffle (SOAR):** 
    *   *Implementation:* The Shuffle Webhook URL is configured inside the `ossec.conf` file of every remote Wazuh Manager.
    *   *Result:* If any branch detects an attack, it instantly pushes an alert to Shuffle in the HQ for automated triage.
*   **TheHive & Cortex (Incident Management):** 
    *   *Implementation:* When Shuffle receives a high-severity alert, it automatically triggers TheHive to open a detailed Case for the SOC analyst.
*   **MISP & OpenCTI (Threat Intelligence):** 
    *   *Implementation:* Hosted centrally. Graylog instances in the branches send API queries to the HQ to check if a specific IP/Hash is malicious *before* storing the log.
*   **InfluxDB & Uptime Kuma (Monitoring):** 
    *   *Implementation:* Kuma monitors the uptime of APIs and Dashboards across all nodes (alerting via Telegram if a branch goes down). InfluxDB receives Telegraf telemetry and visualizes it in Grafana.

---

### 3. 🕵️‍♂️ DFIR Layer: Velociraptor (Isolation & Investigation)
To achieve full XDR capabilities, Velociraptor is integrated for Digital Forensics and Incident Response (DFIR) across the distributed network.

*   **Velociraptor Server:** Hosted centrally in the HQ.
*   **Velociraptor Agents:** Installed on endpoint devices across all branches.

#### ⚡ Automated Response Workflow (Argus in Action):
1. A malicious attack occurs in **Branch B**.
2. **Wazuh Manager B** detects the threat and sends an alert to **Shuffle** in the HQ.
3. **Shuffle** parses the alert and automatically opens a Case in **TheHive**.
4. The SOC Analyst opens TheHive and triggers the **Cortex Responder**.
5. **Cortex** communicates with the **Velociraptor Server** in the HQ.
6. The Velociraptor Server sends a direct execution command to the Agent on the infected device in Branch B to perform a **Network Quarantine** and extract a **Memory Dump** for forensics.

---

### 🚀 Executive Summary
Argus is not just a collection of security tools; it is a **Distributed Enterprise Architecture**. 
- **The Branches** handle the heavy lifting (Local Collection, Normalization via Graylog, and Local Storage via the Indexer). 
- **The Headquarters** handles Orchestration & Visualization (Global display via Grafana, Cross-Cluster Search, Case Management via TheHive, and Automation via Shuffle).

