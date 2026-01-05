# Splunk-homelab-and-work-set-up
Hey yall today i wanted to upload some information about what i have been doing check out this splunk set up!!
# Enterprise SIEM Deployment: 150-Node Splunk Architecture

## 🎯 Project Objective
This project documents the end-to-end deployment of a **Splunk Enterprise** indexing cluster designed to ingest, parse, and visualize security telemetry from a distributed 150-node Windows environment. 

By leveraging **Linux** for the backend and **RMM-based PowerShell automation** for the endpoints, I established a high-visibility security posture on resource-constrained hardware.

---

## 🏗️ System Architecture



* **Primary Indexer:** Headless Linux (Ubuntu) 
* **Endpoints:** 150 Windows Workstations
* **Log Transport:** Splunk TCP (Port 9997)
* **Automation:** PowerShell / RMM (Remote Monitoring & Management)

---

## 🛠️ Implementation Details

### 1. Linux Backend Optimization
To ensure the 4-core Linux environment could handle the concurrent streams of 150 PCs, I performed the following "bare-metal" optimizations:
* **Port Provisioning:** Enabled listening on TCP `9997` via CLI to receive forwarder data.
* **Kernel Tuning:** Modified system `ulimits` to support high-concurrency network handshakes and open file descriptors.
* **Service Hardening:** Configured automated service recovery for the `splunkd` daemon to ensure 24/7 ingestion.

### 2. Scaled PowerShell Deployment
I authored a custom deployment script to push the **Splunk Universal Forwarder (UF)** silently across the enterprise fleet. 

**Key Script Features:**
* **Silent Execution:** MSI installation with zero user-end interruption.
* **Dynamic Assignment:** Automated pointing of endpoints to the central Indexer IP.
* **Stream Provisioning:** Immediate activation of Windows Event Log stanzas upon installation.

```powershell
# Optimized Deployment Logic
$Indexer = "192.18.0.26:9997"
$MsiPath = "$env:TEMP\splunkforwarder.msi"

# Execute Silent Install with basic event stanzas enabled
$Args = "/i `"$MsiPath`" RECEIVING_INDEXER=`"$Indexer`" AGREETOLICENSE=Yes WINEVENTLOG_SEC_ENABLE=1 /quiet"
Start-Process msiexec.exe -ArgumentList $Args -Wait
