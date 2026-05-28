## Troubleshooting: Debian Agent "Never Connected" Error

### Issue
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
