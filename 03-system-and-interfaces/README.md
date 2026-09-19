# Section 3 – System & Interface Configuration

Initial setup of **FW1**: hostname and global settings, interface IP addressing, roles and aliases, and administrative access.

Full config: [`fw1-interfaces.conf`](fw1-interfaces.conf)

## What I Configured

| Interface | Alias | Role | IP Address | Admin Access |
|---|---|---|---|---|
| port1 | WAN-1 | WAN | 192.168.1.1/24 | ping |
| port2 | WAN-2 | WAN | 192.168.2.1/24 | ping |
| port3 | LAN | LAN | 10.0.1.254/24 | ping |
| port4 | DMZ | DMZ | none yet (VLAN subinterfaces added in Section 5) | ping |
| port5 | MGMT | – | 192.168.100.200/24 | ping, https, ssh, http |

Global settings: hostname `FW1`, admin idle timeout set to 480 minutes (lab convenience).

## Verification

`get system interface physical` confirms all interfaces are up with the expected addressing:

```
FW1 # get system interface physical
== [onboard]
        ==[port1]
                mode: static
                ip: 192.168.1.1 255.255.255.0
                status: up
        ==[port2]
                mode: static
                ip: 192.168.2.1 255.255.255.0
                status: up
        ==[port3]
                mode: static
                ip: 10.0.1.254 255.255.255.0
                status: up
        ==[port4]
                mode: static
                ip: 0.0.0.0 0.0.0.0
                status: up
        ==[port5]
                mode: static
                ip: 192.168.100.200 255.255.255.0
                status: up
```
*(Output trimmed to relevant fields.)*

## Key Takeaways

- **Aliases are labels, not names.** The alias (e.g. `WAN-1`) appears in the GUI, but the CLI, routes and policies still reference the real interface name (`port1`).
- **Interface roles shape the GUI.** Setting a role (WAN, LAN, DMZ) changes which options the GUI shows for that interface. Device identification is enabled on the LAN interface so FW1 can detect and inventory connected devices.
- **Admin access is limited to the management port.** WAN and LAN interfaces only allow ping; HTTPS/SSH admin access is only on the dedicated MGMT interface (port5).
- **Lab vs production.** HTTP admin access and a 480-minute admin timeout are for lab convenience. In production I would disable HTTP, keep a short timeout (the FortiOS default is 5 minutes), and restrict admin accounts to trusted hosts.
- **`show` vs `get`.** `show` displays configuration (only values changed from default); `get` displays settings and runtime status such as link state.

## Troubleshooting Log

### `show system interface physical` → "entry is not found in table"

- **Symptom:** Trying to check interface status, the command returned `entry is not found in table`.
- **Cause:** `show system interface` works on the configuration table, so it treated `physical` as the name of an interface to look up. No interface is called "physical", so it wasn't found.
- **Fix:** Physical interface status is runtime information, so it comes from `get`: `get system interface physical`.
