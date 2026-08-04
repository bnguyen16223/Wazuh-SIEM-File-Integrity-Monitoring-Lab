# 🛡️ Enterprise SIEM & Real-Time File Integrity Monitoring (FIM) Lab with Wazuh

[![Wazuh](https://img.shields.io/badge/SIEM-Wazuh%20v4.x-0055FF?style=for-the-badge&logo=wazuh&logoColor=white)](https://wazuh.com)
[![Ubuntu](https://img.shields.io/badge/OS-Ubuntu%20Linux-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)](https://ubuntu.com)
[![Windows](https://img.shields.io/badge/OS-Windows%2010%2F11-0078D6?style=for-the-badge&logo=windows&logoColor=white)](https://microsoft.com)
[![VirtualBox](https://img.shields.io/badge/Hypervisor-Oracle%20VirtualBox-183A61?style=for-the-badge&logo=virtualbox&logoColor=white)](https://virtualbox.org)

## 📌 Executive Summary
This project documents the deployment of an enterprise-grade **Wazuh Security Information and Event Management (SIEM)** solution in a virtualized lab environment. 

The lab features a **Wazuh Central Manager & Indexer** deployed on an Ubuntu Linux server, paired with a **Wazuh Security Agent** on a Windows endpoint. In addition to successful deployment, this project highlights real-world system recovery, troubleshooting package manager locks (`dpkg`/`apt`), and configuring **Real-Time File Integrity Monitoring (FIM)** to detect unauthorized file creations and modifications instantly.

---

## 🛠️ Lab Architecture & Network Topology

```
                       +---------------------------------------+
                       |          Oracle VirtualBox            |
                       |       Bridged Network Adapter         |
                       +-------------------+-------------------+
                                           |
                   +-----------------------+-----------------------+
                   |                                               |
                   v                                               v
  +---------------------------------+             +---------------------------------+
  |      Wazuh Manager VM           |             |      Windows Endpoint VM        |
  |  OS: Ubuntu Server 22.04 LTS    |             |  OS: Windows 10 / 11            |
  |  IP: 192.168.1.167              |             |  IP: 192.168.1.168              |
  |  Role: SIEM Indexer & Dashboard |             |  Role: Monitored Security Agent |
  +----------------+----------------+             +----------------+----------------+
                   ^                                               |
                   |=== Encrypted Telemetry / FIM Alert Logs ======|
                   |               (Port 1514 / 1515)              |
```

| Component | Operating System | IP Address | Hostname / Role |
| :--- | :--- | :--- | :--- |
| **SIEM Manager** | Ubuntu 22.04 LTS | `192.168.1.167` | `wazuh-manager` (Indexer, Server & Dashboard) |
| **Endpoint Agent** | Windows 10/11 | `192.168.1.168` | `win-endpoint-01` (Monitored Host) |
| **Hypervisor** | Host Machine | Bridged Subnet | Oracle VirtualBox |

---

## 🚀 Step-by-Step Implementation Guide

### 1. Environment & Network Configuration
1. Created two virtual virtual machines in Oracle VirtualBox using standard ISO images for Ubuntu Server and Windows.
2. Set network interfaces to **Bridged Adapter** mode to ensure direct IP reachability between hosts on the subnet (`192.168.1.0/24`).
3. Installed core Linux networking and utility tools (`curl`, `apt-transport-https`, `gnupg`, `net-tools`).

---

### 🧠 2. Real-World Engineering & Troubleshooting (Wazuh Recovery)

> [!NOTE]
> During initial deployment, the Wazuh Manager service crashed due to resource constraints and broken configuration scripts. The recovery steps below demonstrate technical problem-solving and Linux package management repair.

#### The Challenge Encountered:
- **Resource Exhaustion:** Wazuh Manager failed to start initially due to insufficient RAM allocations on the Ubuntu VM.
- **Binary Path Loss:** Accidental script errors broke `/var/ossec/bin/wazuh-control`, resulting in `"command not found"`.
- **Package Manager Lockup:** A concatenated command syntax error (`sudo apt-get remove ... sudo apt-get clean`) caused `dpkg` to crash and lock `/var/lib/dpkg/lock-frontend`.
- **Repository GPG Disconnection:** Force-clearing repositories removed official Wazuh GPG keys.

#### Technical Resolution Procedure:
```bash
# 1. Force-release locked package manager locks
sudo rm /var/lib/dpkg/lock-frontend
sudo rm /var/lib/apt/lists/lock
sudo dpkg --configure -a

# 2. Re-import Official Wazuh GPG Key and Repository
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list

# 3. Perform clean update & rebuild Wazuh Indexer and Dashboard
sudo apt-get update
sudo apt-get install --reinstall wazuh-manager wazuh-dashboard -y
sudo systemctl restart wazuh-manager
```
*Outcome:* Wazuh Manager & Web Dashboard fully recovered and restored to operational health.

---

### 3. Agent Deployment & Registration
1. Downloaded and executed the **Wazuh Agent installer** on the Windows Endpoint (`192.168.1.168`).
2. Configured agent authentication connecting to the Manager IP (`192.168.1.167`) via Port 1515.
3. Verified successful registration in the Wazuh Dashboard under **Active Agents**.

---

### 4. Configuring Real-Time File Integrity Monitoring (FIM)

To monitor critical directories on the Windows Endpoint for unauthorized file tampering, the syscheck configuration file (`ossec.conf`) was updated.

#### Configuration Snippet (`C:\Program Files (x86)\ossec-agent\ossec.conf`):
Opened `ossec.conf` as Administrator in Notepad and added the target lab monitoring path under `<syscheck>`:

```xml
<syscheck>
  <!-- Real-Time File Integrity Monitoring for Lab Directory -->
  <directories real_time="yes" check_all="yes">C:\Users\User\Desktop\FIM_Test_Directory</directories>
</syscheck>
```

Restarted the Wazuh Agent service to load updated configuration:
```cmd
net stop wazuh
net start wazuh
```

---

## 🧪 Verification & Alert Testing

1. **Test Execution:** Navigated to `C:\Users\User\Desktop\FIM_Test_Directory` on the Windows host and created a test document (`unauthorized_change_test.txt`) containing sensitive text.
2. **Detection Result:** Wazuh's `syscheck` engine detected the file creation instantly in real-time.
3. **SIEM Alert Inspection:** Opened the Wazuh Dashboard $\rightarrow$ Security Events $\rightarrow$ FIM tab to view the generated alert logs.

### Alert Metadata Captured:
- **Rule ID:** `554` (File added to system) / `550` (File modified)
- **Agent Name:** `win-endpoint-01` (`192.168.1.168`)
- **File Path:** `C:\Users\User\Desktop\FIM_Test_Directory\unauthorized_change_test.txt`
- **Integrity Attributes:** Recorded file hash (MD5/SHA256), timestamp, permissions, and owner user ID.

---

## 📸 Screenshots & Evidence

Below is the complete visual documentation of the lab environment, from endpoint pairing and service verification to real-time File Integrity Monitoring (FIM) alerts:

| Phase / Verification | Description | Visual Evidence |
| :--- | :--- | :--- |
| **1. Endpoint Security & Inventory Summary** | Wazuh Web Dashboard showing active agent `seim-windows` (`002`), IP `192.168.1.168`, Windows 11 system inventory, and PCI-DSS compliance widgets. | `![Wazuh Endpoint Security Overview](./images/01_wazuh_endpoint_security_overview.png)` |
| **2. Windows Agent Connection & IP Config** | Windows CMD `ipconfig` output (`192.168.1.168`) paired with Wazuh Agent Manager GUI in `Running` status connected to Manager IP `192.168.1.167`. | `![Windows Agent Connection](./images/02_windows_agent_connection.png)` |
| **3. Manager Service Verification (Linux)** | Ubuntu Server terminal output confirming `systemctl status wazuh-manager` in `Active: active (running)` state with sub-daemons (`syscheckd`, `analysisd`, `remoted`). | `![Ubuntu Wazuh Manager Status](./images/03_ubuntu_wazuh_manager_status.png)` |
| **4. FIM Configuration in `ossec.conf`** | Notepad Administrator editing `ossec.conf`, configuring real-time monitoring: `<directories realtime="yes">C:\Users\vboxuser\Downloads\WAZUH-TEST</directories>`. | `![FIM Config in ossec.conf](./images/04_ossec_conf_fim_config.png)` |
| **5. Real-Time FIM Events Overview** | Wazuh Dashboard table displaying live `File added` (Level 5) and `File deleted` (Level 7) alerts generated inside `C:\Users\vboxuser\Downloads\WAZUH-TEST`. | `![Real-Time FIM Events Overview](./images/05_wazuh_fim_events_overview.png)` |
| **6. Detailed Alert Log Breakdown** | Deep-dive Document Details modal inspecting `syscheck_deleted` rule ID, exact file path, `agent.ip: 192.168.1.168`, and `Mode: realtime` payload. | `![FIM Alert Document Details](./images/06_wazuh_fim_document_details.png)` |
| **7. SIEM Dashboard Health Check** | Wazuh Web UI confirming successful API connection, index pattern validation, and cluster statistics checks. | `![Wazuh Dashboard Health Check]<img width="772" height="486" alt="images:07_wazuh_dashboard_healthcheck" src="https://github.com/user-attachments/assets/a655a1d5-8db7-4b38-9ebf-ed3a85aa9210" />
` |

---

## 💡 Key Takeaways & Lessons Learned

1. **Troubleshooting Resilience:** Understanding how Linux package managers (`apt`/`dpkg`) handle locks and repository keys is essential when resolving deployment failures in production SIEM infrastructure.
2. **Proactive File Integrity Monitoring:** Real-Time FIM is vital for compliance (NIST CSF / PCI-DSS) to catch ransomware file encryption, web shell drops, and unauthorized privilege escalation.
3. **Log Correlation Power:** Centralizing agent logs into a single SIEM dashboard dramatically reduces Incident Response (IR) time compared to manual Windows Event Viewer inspection.
