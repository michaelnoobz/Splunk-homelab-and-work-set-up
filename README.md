🚀 Project "Franken-Splunk": 150-Node SIEM Deployment
📝 The Backstory
I had one 8-hour shift to stand up a production-ready Splunk Enterprise environment from scratch. No budget, no new hardware, and 150 PCs that needed monitoring. By scavenging old drives for an LVM pool, snagging the Splunk Pledge (10GB/day) license, and script-automating the rollout via RMM, I got the whole fleet talking to a central indexer before the end of the day.

🛠️ Phase 1: Storage Engineering (The LVM Grind)
I didn't have one big drive, so I had to make one. I used Logical Volume Management (LVM) to strip together a bunch of smaller physical disks into a unified 1TB pool. This kept the high-speed data flow from choking the OS partition.

The CLI Work:
Bash

# Cleaned the old partition tables off the scavenged drives
sudo wipefs -a /dev/sdX /dev/sdY

# Turned them into Physical Volumes
sudo pvcreate /dev/sdX /dev/sdY

# Smashed them together into one Volume Group
sudo vgcreate [VG_NAME] /dev/sdX /dev/sdY

# Carved out a 930GB Logical Volume for the actual Splunk DB
sudo lvcreate -L 930G -n splunk_data_lv [VG_NAME]

# Formatted and mounted it so Splunk could actually use it
sudo mkfs.ext4 /dev/[VG_NAME]/splunk_data_lv
sudo mount /dev/[VG_NAME]/splunk_data_lv /opt/splunk/var/lib/splunk
🏗️ Phase 2: Building the Pipe
I set up the Splunk Indexer on Ubuntu LTS to be the "brain" for all 150 nodes.

The Ingest: Fired up TCP Port 9997 to listen for incoming data.

Storage Mapping: Tweaked the config files to make sure $SPLUNK_DB lived on my new 930GB drive.

The License: Got the 10GB/day license active so we could actually log real security events without hitting a wall.

🔄 Phase 3: The "Ghost Server" Pivot
The Problem: Trying to download a 100MB installer 150 times would have absolutely killed our office internet. The Fix: I turned the Splunk server itself into a local file host using a Python HTTP listener inside a GNU Screen session.

Bash

# Start a persistent 'Ghost Session'
screen -S forwarder_deploy
cd /path/to/installer/
# Host the file locally so the PCs pull from the LAN, not the Web
sudo python3 -m http.server 80
# (Ctrl+A, D to let it run in the background)
⚡ Phase 4: The RMM "Full Blast" Rollout
I wrote a PowerShell wrapper and pushed it out through our RMM. The script tells every PC to grab the file from my local "Ghost Server" at lightning speed and install itself silently.

My Deployment Script:
PowerShell

# 1. Check if Splunk is already there (don't break what's working)
if (Get-Service "SplunkForwarder" -ErrorAction SilentlyContinue) {
    Write-Output "Already installed. Moving on."
    exit 0
}

# 2. Grab the MSI from my local Linux server (LAN speed, not WAN speed)
$url = "http://[INTERNAL_IP]/splunkforwarder.msi"
$dest = "C:\Windows\Temp\splunkforwarder.msi"
Invoke-WebRequest -Uri $url -OutFile $dest

# 3. Install it silently and point it straight to my Indexer on 9997
$args = "/i `"$dest`" AGREETOLICENSE=Yes RECEIVING_INDEXER=`"[INTERNAL_IP]:9997`" /quiet"
Start-Process msiexec.exe -ArgumentList $args -Wait

# 4. Clean up the temp file
Remove-Item $dest
🔍 Phase 5: Monitoring the Army
Once I hit "Run" in the RMM, I jumped into Splunk to watch the hosts start checking in.

The "Is it working?" Search:
Code snippet

| metadata type=hosts index=_internal 
| eval last_seen=strftime(lastTime, "%Y-%m-%d %H:%M:%S") 
| table host, last_seen
My First SOC Search (Hunting for Brute Force):
Code snippet

index=main EventCode=4625 
| stats count by TargetUserName, IpAddress, host 
| where count > 10 
| rename count as "Failed Attempts"
📈 The Win
100% Visibility: Every PC is now reporting in.

No Network Lag: Handled the whole rollout over the LAN without touching the WAN.

Zero Cost: Built the whole thing with scavenged parts and smart scripting.

SOC Ready: We went from "blind" to "threat hunting" in one day.
