# Wazuh SIEM Home Lab

This project demonstrates the deployment of a Security Information and Event Management (SIEM) system using Wazuh in a VirtualBox environment.

## 🌐 Network Architecture

All VMs use a dual-NIC layout to isolate management/logging traffic from the internet:
- **Adapter 1 (NAT):** Outbound internet access for packages and updates.
- **Adapter 2 (Internal Network - `intnet`):** Isolated private management plane for log shipping.

### Subnet Layout (`192.168.100.0/24`)
- `192.168.100.10` - Ubuntu Server 22.04.5 (Wazuh Manager & Dashboard)
- `192.168.100.20` - Debian (Wazuh Agent)
- `192.168.100.30` - Windows 10 (Wazuh Agent)
- `192.168.100.40` - Kali Linux (Attacker Node)
- `192.168.100.50` - Metasploitable2 (Target Agent)

## 🎯 Objectives

- **Log Collection:** Ingesting Windows Security Events and Linux syslogs into a central manager.
- **Attack Simulation:** Using Kali Linux to run brute-force and network scans against targets to generate alerts.
- **System Hardening:** Configuring firewalls and monitoring file integrity changes.

## 🛠️ Skills Demonstrated

- SIEM deployment and configuration
- Endpoint log monitoring (Windows & Linux)
- Security event analysis
- Basic incident detection and investigation
- Virtualized lab environment setup

## 📁 Repository Structure

```text
├── configs/          # Configuration files and settings
├── notes/            # Installation steps and command notes
├── reports/          # Incident analysis and findings
├── screenshots/      # Dashboard and alert evidence
└── troubleshooting/  # Technical breakdown of network/IP conflict resolutions
```

## ⚙️ Tools Used

- **Hypervisor:** VirtualBox
- **SIEM:** Wazuh
- **OS Platforms:** Ubuntu, Debian, Windows 10, Kali Linux, Metasploitable2

---

This project is part of a cybersecurity learning portfolio focused on SOC operations and SIEM analysis.

