🚀 Project "Franken-Splunk": 150-Node SIEM Deployment
📝 Project Narrative
The mission was to architect and stand up a production-ready Splunk Enterprise environment for an organization in a single 8-hour shift. To achieve this with zero budget, I utilized scavenged physical drives, a community-tier Splunk Pledge (10GB/day) license, and automated deployment via an enterprise RMM (Remote Monitoring and Management) tool.

🛠️ Phase 1: Storage Engineering (LVM)
Because the server utilized multiple small physical disks, I implemented Logical Volume Management (LVM) to aggregate the hardware into a unified 1TB high-performance storage pool. This decoupled high-velocity data ingestion from the OS partition to ensure system stability.

Critical Commands Executed:
Bash

# 1. Prepare raw disks by wiping existing partition tables
sudo wipefs -a /dev/sdX /dev/sdY

# 2. Initialize Physical Volumes (PV)
sudo pvcreate /dev/sdX /dev/sdY

# 3. Aggregate disks into a unified Volume Group (VG)
sudo vgcreate [VG_NAME] /dev/sdX /dev/sdY

# 4. Provision a ~930GB Logical Volume (LV) for the Splunk DB
sudo lvcreate -L 930G -n splunk_data_lv [VG_NAME]

# 5. Format and mount the new volume to the Splunk data path
sudo mkfs.ext4 /dev/[VG_NAME]/splunk_data_lv
sudo mount /dev/[VG_NAME]/splunk_data_lv /opt/splunk/var/lib/splunk
🏗️ Phase 2: Architecting the Ingest Pipeline
I configured the Splunk Indexer on Ubuntu LTS to act as the central "receiver" for the 150-node fleet.

Receiving Tier: Enabled TCP Port 9997 as the dedicated ingest listener.

Storage Mapping: Modified configuration files to map $SPLUNK_DB directly to the LVM mount point.

License Management: Activated the Splunk Pledge 10GB/day license to accommodate high-fidelity security audit logging.

🔄 Phase 3: The "Ghost Server" Distribution (Engineering Pivot)
The Challenge: Downloading a 100MB installer over the WAN 150 times would have saturated the office network.

The Solution: I transformed the Splunk Indexer into a Localized Binary Distribution Point using a Python3 HTTP listener inside a persistent terminal session.

Bash

# Launching the persistent distribution point
screen -S forwarder_deploy
cd /path/to/installer/
sudo python3 -m http.server 80
# Detach session (Ctrl+A, D) to keep active in background
⚡ Phase 4: Automated RMM Fleet Rollout
I developed a PowerShell deployment wrapper and executed it via the organization's RMM. This script pulled the installer from the internal distribution point over the local LAN at wire-speed.

The Deployment Script:
PowerShell

# 1. Verify if agent is already present
if (Get-Service "SplunkForwarder" -ErrorAction SilentlyContinue) {
    Write-Output "Agent already installed. Exiting."
    exit 0
}

# 2. Fetch installer from the internal distribution server
$url = "http://[INTERNAL_IP]/splunkforwarder.msi"
$dest = "C:\Windows\Temp\splunkforwarder.msi"
Invoke-WebRequest -Uri $url -OutFile $dest

# 3. Silent Installation pointing back to the Indexer on Port 9997
$args = "/i `"$dest`" AGREETOLICENSE=Yes RECEIVING_INDEXER=`"[INTERNAL_IP]:9997`" /quiet"
Start-Process msiexec.exe -ArgumentList $args -Wait

# 4. Cleanup
Remove-Item $dest
🔍 Phase 5: Verification & SOC Use Cases
Once the fleet was live, I validated the ingestion pipeline using Splunk Search Processing Language (SPL) to verify telemetry from various nodes was being parsed correctly.

Heartbeat Check:
Code snippet

| metadata type=hosts index=_internal 
| eval last_seen=strftime(lastTime, "%Y-%m-%d %H:%M:%S") 
| table host, last_seen
Initial SOC Security Search (Brute Force Detection):
Code snippet

index=main EventCode=4625 
| stats count by TargetUserName, IpAddress, host 
| where count > 10 
| rename count as "Failed Attempts"
📈 Project Impact
Total Visibility: Centralized telemetry for 150 endpoints.

Network Efficiency: Zero WAN impact during rollout via localized distribution.

Infrastructure Resilience: Scalable LVM storage backend managed via CLI.

Cost Efficiency: $0 total hardware/software spend by leveraging scavenged resources.
