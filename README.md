# 🚀 Project "Franken-Splunk": From Scavenged Hardware to 150-Node SIEM

## 📝 The Mission
To build a production-grade Security Information and Event Management (SIEM) environment for a non-profit organization in a single 8-hour shift. This required a "Zero-Budget" approach: utilizing scavenged hardware, a 10GB/day community license, and automated deployment via existing RMM tools.

---

## 🛠️ Phase 1: Storage Engineering & The "Franken-PC"
The server was built from disparate physical disks. To ensure Splunk had the speed and capacity required for 150 endpoints, I utilized **Linux LVM (Logical Volume Management)**.

* **The Problem:** Multiple small disks that couldn't hold the index individually.
* **The Fix:** Aggregated `/dev/sda` and `/dev/sdc` into a unified Volume Group (`ubuntu-vg`).
* **The Result:** A high-capacity 930GB Logical Volume dedicated to `$SPLUNK_DB`, ensuring OS stability while maximizing data retention.

---

## 🏗️ Phase 2: Architecting the Ingest Pipeline
We stood up Splunk Enterprise 10.0.2 on Ubuntu 24.04.3 LTS with a focus on **Persistence and Reliability**.

* **Network Handshake:** Configured a dedicated TCP listener on **Port 9997**.
* **License Management:** Successfully implemented the **Splunk Pledge (10GB/day)**, allowing for full-scale ingestion of Windows audit logs.
* **Service Hardening:** Modified `splunk-launch.conf` to map data paths directly to the LVM mount point.

---

## 🔄 Phase 3: The "Ghost Server" Distribution (Engineering Pivot)
**Challenge:** Downloading a 100MB MSI 150 times would have saturated the organization's WAN and triggered rate-limiting.

**Solution:** I transformed the Splunk Indexer into a **Localized Binary Distribution Point**.
1.  **Staging:** Used `wget` to pull the Universal Forwarder (UF) MSI to the server.
2.  **Hosting:** Launched a **Python3 HTTP listener** on Port 80.
3.  **Persistence:** Utilized **GNU Screen** to keep the file server alive in a "Ghost Session," allowing for an asynchronous, batch-based rollout over several days.

---

## ⚡ Phase 4: Automated RMM Fleet Rollout
To hit the 150-node goal, I developed a **PowerShell wrapper** for deployment via **NinjaOne RMM**.

**Deployment Logic:**
1.  **Idempotency:** The script checks for the `SplunkForwarder` service before execution to prevent system drift.
2.  **LAN Speed Fetch:** Points the Windows client to the internal "Ghost Server" IP (`192.168.0.26`) for a sub-5-second download.
3.  **Silent Parameterization:** Uses `msiexec` with `AGREETOLICENSE=Yes` and `RECEIVING_INDEXER` flags to ensure zero-touch installation.

---

## 🔧 Engineering Pivots & Troubleshooting (The "Real World" Stuff)
During the build, we encountered several "real-world" hurdles that required immediate pivots:
* **LVM Recovery:** Fixed partition table conflicts using `wipefs` before the Volume Group would initialize.
* **Permission Hardening:** Resolved `sudo` requirements for the Python HTTP listener to ensure port 80 was accessible to the fleet.
* **Browser Security:** Bypassed "Insecure" warnings for internal LAN traffic by validating the **Digital Signature** of the MSI payload before execution.

---

## 📊 Final Result: SOC Visibility
At the end of the shift, the environment was successfully handshaking with the fleet.

* **Active Nodes:** 150 (Phased rollout)
* **Ingest Port:** TCP 9997
* **Primary Search for Heartbeat:**
    `index=_internal | stats count by host, source, version`

**This project proves that with Linux grit, PowerShell automation, and strategic resource pooling, enterprise-grade security visibility is possible on any budget.**
