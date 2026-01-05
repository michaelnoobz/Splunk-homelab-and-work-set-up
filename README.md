<!DOCTYPE html>
<html>
<head>
<style>
    body { font-family: sans-serif; line-height: 1.6; color: #333; max-width: 900px; margin: auto; padding: 20px; }
    h1 { color: #2c3e50; border-bottom: 2px solid #3498db; padding-bottom: 10px; }
    h2 { color: #2980b9; margin-top: 30px; border-left: 5px solid #3498db; padding-left: 10px; }
    h3 { color: #16a085; }
    code { background-color: #f4f4f4; padding: 2px 5px; border-radius: 3px; font-family: monospace; }
    pre { background-color: #2d2d2d; color: #ccc; padding: 15px; border-radius: 5px; overflow-x: auto; font-family: monospace; }
    .highlight { color: #f39c12; font-weight: bold; }
    .impact { background-color: #e8f6f3; padding: 15px; border-radius: 5px; border: 1px solid #16a085; }
</style>
</head>
<body>

    <h1>🚀 Project "Franken-Splunk": 150-Node SIEM Deployment</h1>

    <p><b>Project Narrative:</b> The mission was to architect and stand up a production-ready <b>Splunk Enterprise</b> environment for a non-profit organization in a single 8-hour shift. To achieve this with zero budget, I utilized scavenged physical drives, a <b>Splunk Pledge (10GB/day)</b> community license, and automated deployment via <b>NinjaOne RMM</b>.</p>

    <h2>🛠️ Phase 1: Storage Engineering (LVM)</h2>
    <p>Because the server utilized multiple small physical disks, I implemented <b>Logical Volume Management (LVM)</b> to aggregate the hardware into a unified 1TB high-performance storage pool. This decoupled high-velocity data ingestion from the OS partition to ensure system stability.</p>
    
    <h3>Critical Commands Executed:</h3>
    <pre>
# 1. Prepare raw disks by wiping existing partition tables
sudo wipefs -a /dev/sda /dev/sdc

# 2. Initialize Physical Volumes (PV)
sudo pvcreate /dev/sda /dev/sdc

# 3. Aggregate disks into a unified Volume Group (VG)
sudo vgcreate ubuntu-vg /dev/sda /dev/sdc

# 4. Provision a 930GB Logical Volume (LV) for the Splunk DB
sudo lvcreate -L 930G -n splunk_data_lv ubuntu-vg

# 5. Format and mount the new volume to the Splunk data path
sudo mkfs.ext4 /dev/ubuntu-vg/splunk_data_lv
sudo mount /dev/ubuntu-vg/splunk_data_lv /opt/splunk/var/lib/splunk
    </pre>

    <h2>🏗️ Phase 2: Architecting the Ingest Pipeline</h2>
    <p>I configured the <b>Splunk Indexer</b> on Ubuntu 24.04.3 LTS to act as the central "receiver" for the 150-node fleet.</p>
    <ul>
        <li><b>Receiving Tier:</b> Enabled <b>TCP Port 9997</b> as the dedicated ingest listener.</li>
        <li><b>Storage Mapping:</b> Modified <code>splunk-launch.conf</code> to map <code>$SPLUNK_DB</code> directly to the LVM mount point.</li>
        <li><b>License Management:</b> Activated the <b>Splunk Pledge 10GB/day</b> license to accommodate high-fidelity audit logging.</li>
    </ul>

    <h2>🔄 Phase 3: The "Ghost Server" Distribution</h2>
    <p><span class="highlight">The Challenge:</span> Downloading a 100MB MSI over the WAN 150 times would have saturated the office network.</p>
    <p><span class="highlight">The Solution:</span> I transformed the Splunk Indexer into a <b>Localized Binary Distribution Point</b> using a Python3 HTTP listener inside a persistent <b>GNU Screen</b> session.</p>
    <pre>
# Launching the persistent distribution point
screen -S forwarder_deploy
cd /home/mlannen/
sudo python3 -m http.server 80
# Detach with Ctrl+A, then D
    </pre>

    <h2>⚡ Phase 4: Automated RMM Fleet Rollout</h2>
    <p>I developed a <b>PowerShell deployment wrapper</b> and executed it via <b>NinjaOne RMM</b>. This script pulled the MSI from the internal "Ghost Server" over the local LAN at wire-speed.</p>
    
    <h3>The Deployment Script:</h3>
    <pre>
# 1. Verify if Splunk is already present
if (Get-Service "SplunkForwarder" -ErrorAction SilentlyContinue) {
    Write-Output "Splunk already installed. Exiting."
    exit 0
}

# 2. Fetch MSI from the internal 'Ghost Server' (192.168.0.26)
$url = "http://192.168.0.26/splunkforwarder.msi"
$dest = "C:\Windows\Temp\splunkforwarder.msi"
Invoke-WebRequest -Uri $url -OutFile $dest

# 3. Silent Installation pointing back to the Indexer on Port 9997
$args = "/i `"$dest`" AGREETOLICENSE=Yes RECEIVING_INDEXER=`"192.168.0.26:9997`" /quiet"
Start-Process msiexec.exe -ArgumentList $args -Wait

# 4. Cleanup
Remove-Item $dest
    </pre>

    <h2>🔍 Phase 5: Verification & SOC Use Cases</h2>
    <p>Once the fleet was live, I validated the ingestion pipeline using <b>Splunk Search Processing Language (SPL)</b> to verify telemetry from nodes (e.g., <code>201-516</code>).</p>
    
    <h3>Heartbeat Check:</h3>
    <pre>
| metadata type=hosts index=_internal 
| eval last_seen=strftime(lastTime, "%Y-%m-%d %H:%M:%S") 
| table host, last_seen
    </pre>

    <h3>Initial SOC Security Search (Brute Force Detection):</h3>
    <pre>
index=main EventCode=4625 
| stats count by TargetUserName, IpAddress, host 
| where count > 10 
| rename count as "Failed Attempts"
    </pre>

    <div class="impact">
        <h2>📈 Project Impact</h2>
        <ul>
            <li><b>Total Visibility:</b> Centralized telemetry for 150 clinical endpoints.</li>
            <li><b>Network Efficiency:</b> Zero WAN impact during rollout via localized distribution.</li>
            <li><b>Infrastructure Resilience:</b> Scalable LVM storage backend.</li>
            <li><b>Cost Efficiency:</b> $0 total spend by leveraging scavenged hardware.</li>
        </ul>
    </div>

</body>
</html>
