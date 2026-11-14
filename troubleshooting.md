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
 Troubleshooting Log: Missing CIS SCA Policy on Debian 11 Wazuh Agent (v4.7.5)

## Issue #2: Configuring CIS SCA Policy on Debian 13.4 for Wazuh Agent (v4.7.5)

### Problem Summary
The Debian endpoint had no Security Configuration Assessment (SCA) policies loaded in `/var/ossec/ruleset/sca/`. While setting up the CIS benchmark policy, we encountered three distinct issues:
1. **Download Failures (`curl` errors 3 & 6):** `Could not resolve host: githubusercontent.com` and `URL rejected: No host part in the URL`.
2. **Permission Errors:** `Permission denied` and `No such file or directory` when writing to `/var/ossec/ruleset/sca/`.
3. **Skipped SCA Scans:** Even after downloading `cis_debian11.yml`, Wazuh logged `Skipping policy '/var/ossec/ruleset/sca/cis_debian11.yml': 'Check Debian version.'` and nothing appeared in the Wazuh dashboard.

---

### Root Causes
1. **Missing Directory & Restrictive Permissions:** `/var/ossec/ruleset/sca/` did not exist, and `/var/ossec` is locked to `root:wazuh` (`750`), requiring `sudo`.
2. **Clipboard/VM Console URL Stripping:** Pasting `://githubusercontent.com` into the VM terminal automatically stripped the `raw.` subdomain (turning it into `://githubusercontent.com`), causing `curl` to fail.
3. **OS Version Mismatch (Debian 13.4 vs. Wazuh v4.7.5):** Checking `/etc/debian_version` revealed the endpoint was running **Debian 13.4 (Trixie)**. Because Wazuh v4.7.5 predates Debian 13, no `cis_debian13.yml` existed in the v4.7.5 ruleset, and `cis_debian11.yml` skipped execution because its `requirements:` block strictly required Debian `11`.

---

### Step-by-Step Resolution

#### 1. Verified OS Version & Network Connectivity
Confirmed internet/DNS access and identified the exact Debian release (`13.4`):
```bash
ping -c 2 ://githubusercontent.com
cat /etc/debian_version
```

#### 2. Created the SCA Directory with Proper Ownership
Created the missing `sca` folder inside Wazuh's ruleset directory and assigned it to `root:wazuh`:
```bash
sudo mkdir -p /var/ossec/ruleset/sca
sudo chown -R root:wazuh /var/ossec/ruleset/sca
sudo chmod 750 /var/ossec/ruleset/sca
```

#### 3. Bypassed Clipboard Filtering & Downloaded `cis_debian12.yml`
Removed the mismatched Debian 11 policy, constructed the URL using split variables so the VM clipboard would not strip `raw.`, and downloaded the v4.7.5 `cis_debian12.yml` ruleset (the closest architectural match to Debian 13):
```bash
sudo rm -f /var/ossec/ruleset/sca/cis_debian11.yml
R="raw"
H="\$://githubusercontent.com"
P="wazuh/wazuh/v4.7.5/ruleset/sca/debian/cis_debian12.yml"
sudo curl -fLo /var/ossec/ruleset/sca/cis_debian12.yml https://H/P
```

#### 4. Updated Policy OS Requirements from Debian 12 to Debian 13
Modified the `requirements:` block at the top of `cis_debian12.yml` so Wazuh v4.7.5 would execute the checks on Debian 13.4 instead of skipping them:
```bash
sudo sed -i '1,35s/12/13/g' /var/ossec/ruleset/sca/cis_debian12.yml
```

#### 5. Set File Permissions & Restarted the Wazuh Agent
Applied Wazuh's required ownership (`root:wazuh`) and permissions (`640`), then restarted the agent to trigger the scan and sync results to the Wazuh manager:
```bash
sudo chown root:wazuh /var/ossec/ruleset/sca/cis_debian12.yml
sudo chmod 640 /var/ossec/ruleset/sca/cis_debian12.yml
sudo systemctl restart wazuh-agent
```

---

### Verification
Confirmed in `ossec.log` that the SCA module loaded `cis_debian12.yml`, evaluated the checks without skipping, and sent the results to the Wazuh dashboard:
```bash
sudo grep -i "sca" /var/ossec/logs/ossec.log | tail -n 10
```

### Key Takeaways
* **Always verify `/etc/debian_version` first:** Wazuh SCA policies will silently skip execution (`Skipping policy`) if the YAML `requirements:` section does not match the host OS version.
* **Watch for VM console clipboard mangling:** If pasting URLs drops subdomains or characters, assemble the URL in smaller shell variables (`$R`, `$H`, `$P`) before passing it to `curl`.
* **Adapting newer OS releases on older Wazuh versions:** When running an OS release newer than your Wazuh manager/agent (e.g., Debian 13 on Wazuh v4.7.5), use the closest available benchmark (`cis_debian12.yml`) and adjust the top `requirements:` version check.

