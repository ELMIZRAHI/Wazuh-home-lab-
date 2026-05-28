## Troubleshooting: Debian Agent "Never Connected" Error

### Issue
The Debian agent registered with the Wazuh dashboard but permanently statused as `Never Connected`. Local services were running, but no data reached the manager.

### Troubleshooting Steps & Root Cause
1. **Host Resolution Check:** Inspected `/etc/hosts` on the manager to ensure `127.0.1.1` loops weren't breaking the hostname resolution.
2. **Daemon Status:** Verified `wazuh-remoted` was running via `wazuh-control status`. Added `<bind_addr>0.0.0.0</bind_addr>` to `ossec.conf` to force the manager to listen across all network interfaces.
3. **Port Connectivity:** Ran `nc -zv` and `nc -zvu` from the Debian machine to the manager on port 1514. Received `Connection Refused` on both TCP and UDP.
4. **IP Conflict Discovery:** Checked the kernel routing table via `ip route` on Debian. Discovered an IP collision:
   `192.168.100.0/24 dev enp0s8 proto kernel scope link src 192.168.100.10`
   The Debian agent had assigned itself `192.168.100.10`, which was the exact static IP of the Ubuntu Wazuh manager. Traffic was routing back into itself.

### Solution
1. **Reassigned IP:** Edited `/etc/network/interfaces` on Debian to permanently change the static IP from `.10` to `.20`.
2. **Flushed Interface:** Cleared the old IP and restarted the network service:
   ```bash
   sudo ip addr flush dev enp0s8
   sudo systemctl restart networking
   ```
3. **Reset Crypto Keys:** Wiped the out-of-sync agent keys on the manager via `manage_agents`. Manually imported the clean key on the Debian machine using `manage_agents -i "KEY"`.
4. **Result:** Netcat tests passed and the Debian agent successfully changed to `Active` on the dashboard.
