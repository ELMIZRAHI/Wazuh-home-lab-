## Troubleshooting: Debian Agent "Never Connected" Error

### Issue 1
The Debian agent registered with the Wazuh dashboard but remained stuck as `Never Connected`. Local services were running, but no data reached the manager.

### Troubleshooting Steps & Root Cause
1. **Host Resolution Check:** Inspected `/etc/hosts` on the manager to ensure `127.0.1.1` loops weren't breaking hostname resolution.
2. **Daemon Status:** Verified `wazuh-remoted` was running via `wazuh-control status`. Added `<bind_addr>0.0.0.0</bind_addr>` to `ossec.conf` to force the manager to listen across all network interfaces.
3. **Port Connectivity:** Ran `nc -zv` and `nc -zvu` from the Debian machine to the manager on port 1514. Received `Connection Refused` on both TCP and UDP.
4. **IP Conflict Discovery:** Checked the kernel routing table via `ip route` on Debian. Discovered an IP collision:
   `192.168.100.0/24 dev enp0s8 proto kernel scope link src 192.168.100.10`
   The Debian agent had assigned itself `192.168.100.10`, which was the exact static IP of the Ubuntu Wazuh manager. Traffic was routing back into itself.

### Solution
1. **Reassigned Static IP:** Edited `/etc/network/interfaces` on Debian to permanently change the static IP from `.10` to `.20` on the `enp0s8` adapter.
2. **Forced Interface Reset & Flush:** Ran a manual hard reset to force the OS to drop the stale configuration and apply the new IP:
   ```bash
   sudo ip link set enp0s8 down
   sudo ip addr flush dev enp0s8
   sudo ip link set enp0s8 up
   sudo ip addr add 192.168.100.20/24 dev enp0s8
   sudo systemctl restart networking
   ```
3. **Reset Crypto Keys:** Wiped the out-of-sync agent registry entry on the manager via `manage_agents`. Cleared the old key file on Debian: `sudo rm -f /var/ossec/etc/client.keys`.
4. **Manual Key Injection:** Manually imported the fresh cryptographic key on the Debian machine using the CLI flag to bypass enrollment script timeouts:
   ```bash
   sudo /var/ossec/bin/manage_agents -i "Wazuh-Generated-Key-String"
   ```
5. **Daemon Reload & Restart:** Reloaded systemd configurations and recycled the agent service:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl restart wazuh-agent
   ```
6. **Result:** Netcat validation passed and the Debian agent status successfully flipped to `Active` on the dashboard.


##Issue 2
# Troubleshooting Log: Missing CIS SCA Policy on Debian 11 Wazuh Agent (v4.7.5)

## Problem Summary
The Debian 11 endpoint had no Security Configuration Assessment (SCA) policies loaded in `/var/ossec/ruleset/sca/`. Attempts to download the official `cis_debian11.yml` policy from GitHub using `curl` repeatedly failed with:
* `Permission denied` / `No such file or directory`
* `curl: (6) Could not resolve host: githubusercontent.com`
* `curl: (3) URL rejected: Malformed input to a URL function`

---

## Root Causes
1. **Missing Directory & Restrictive Permissions:** The `/var/ossec/ruleset/sca/` directory did not exist yet, and `/var/ossec` is owned by `root:wazuh` (`750`), requiring `sudo` for directory creation, downloads, and permission changes.
2. **Clipboard/Terminal URL Truncation:** Pasting long GitHub URLs into the VM console automatically stripped `raw.` and the repository path (`/wazuh/wazuh/v4.7.5`), turning the URL into `https://githubusercontent.com` (which does not resolve via DNS) or introducing hidden characters/spaces that broke `curl`.

---

## Step-by-Step Resolution

### 1. Verified Network & DNS Connectivity
Confirmed the VM had internet access and could resolve GitHub's raw content domain:
```bash
ping -c 2 8.8.8.8
ping -c 2 ://githubusercontent.com
```

### 2. Created the SCA Directory with Proper Ownership
Created the missing `sca` folder inside Wazuh's ruleset directory and assigned it to `root:wazuh`:
```bash
sudo mkdir -p /var/ossec/ruleset/sca
sudo chown -R root:wazuh /var/ossec/ruleset/sca
sudo chmod 750 /var/ossec/ruleset/sca
```

### 3. Bypassed Clipboard Truncation Using Shell Variables
Split the domain and file path into two short variables so the full URL would not get altered during copy-paste, then downloaded the v4.7.5-matched policy file:
```bash
H="://githubusercontent.com"
P="wazuh/wazuh/v4.7.5/ruleset/sca/debian/cis_debian11.yml"
sudo curl -fLo /var/ossec/ruleset/sca/cis_debian11.yml https://H/P
```

### 4. Set File Permissions & Restarted the Wazuh Agent
Applied Wazuh's required ownership (`root:wazuh`) and permissions (`640`) to the YAML file, then restarted the agent to trigger the initial scan:
```bash
sudo chown root:wazuh /var/ossec/ruleset/sca/cis_debian11.yml
sudo chmod 640 /var/ossec/ruleset/sca/cis_debian11.yml
sudo systemctl restart wazuh-agent
```

---

## Verification
Confirmed the policy file was saved properly and verified in the agent logs that the SCA engine loaded and evaluated `cis_debian11.yml`:
```bash
ls -lh /var/ossec/ruleset/sca/
sudo grep -i "sca" /var/ossec/logs/ossec.log
```

## Key Takeaways
* Always match the CIS YAML ruleset branch/tag (e.g., `v4.7.5`) to the installed Wazuh agent version to avoid syntax mismatches.
* Avoid using `curl -s` (silent mode) when troubleshooting downloads, as it hides useful error messages.
* If a VM console mangles long pasted URLs, break the URL into smaller shell variables (`$H` and `$P`) before passing it to `curl`.
