# Section 3 – System & Interface Configuration

**Status:** ✅ Complete

Initial setup of **FW1**: management access, system settings (DNS, time zone, NTP), and interface configuration with roles, aliases and administrative access.

Config files:
- [`fw1-system.conf`](fw1-system.conf) – hostname, admin timeout, DNS, NTP
- [`fw1-interfaces.conf`](fw1-interfaces.conf) – interfaces

## Labs

- [Lab 1 – Initial Configuration & Management Access](#lab-1--initial-configuration--management-access)
- [Lab 2 – System Settings: DNS, Time & NTP](#lab-2--system-settings-dns-time--ntp)
- [Lab 3 – Interface Configuration](#lab-3--interface-configuration)

---

### Lab 1 – Initial Configuration & Management Access

**Configured on FW1 (console)**

- New admin password (first login with the default `admin` account forces a change)
- Hostname `FW1`
- port5: `192.168.100.200/24`, administrative access HTTPS, HTTP, SSH, PING

With port5 reachable, FW1 can be managed from the management network over HTTPS (GUI) and SSH. I use SSH from Remmina on my Ubuntu host, which makes copying CLI output much easier than the web console.

**Takeaways**

- Out of the box, a FortiGate VM is managed from the console. The first step is giving one interface an IP and admin access so the GUI and SSH become available.
- Management access is kept to a dedicated management interface (port5). Other interfaces only allow PING (see Lab 3).

---

### Lab 2 – System Settings: DNS, Time & NTP

**Configured**

- **DNS** (GUI: Network > DNS): primary `8.8.8.8`, secondary `1.1.1.1`
- **Time zone** (GUI: System > Settings): GMT+10:00 Canberra, Melbourne, Sydney
- **Idle timeout:** 480 minutes (lab convenience)
- **NTP** (CLI): sync time from the AD server (`192.168.100.230`, Windows Server 2019 acting as NTP server), and act as a local NTP server for the management network on port5

```
config system ntp
    set ntpsync enable
    set type custom
    config ntpserver
        edit 1
            set server "192.168.100.230"
        next
    end
    set server-mode enable
    set interface "port5"
end
```

After the CLI change, the GUI (System > Settings) showed NTP set to **Custom** with `192.168.100.230`.

**Verification**

```
FW1 # get system dns
primary             : 8.8.8.8
secondary           : 1.1.1.1
protocol            : cleartext
```

```
FW1 # diagnose sys ntp status
synchronized: yes, ntpsync: enabled, server-mode: enabled

ipv4 server(192.168.100.230) 192.168.100.230 -- reachable(0x1) S:1 T:689 selected
        server-version=3, stratum=1
        clock offset is -0.575193 sec, root delay is 0.000000 sec
```

```
FW1 # get system status
Version: FortiGate-VM64-KVM v7.0.9,build0444,221121 (GA.M)
License Status: Valid
Hostname: FW1
Operation Mode: NAT
System time: Sun Sep 20 17:30:43 2026
```
*(Trimmed.)* System time was also confirmed on the dashboard's System Information widget.

**Takeaways**

- FortiGate needs DNS for its own services (FortiGuard, resolving NTP/server names). `config system dns` does the same as the GUI page, and `get system dns` verifies it.
- Accurate time matters for logs, certificates and authentication. Syncing from the AD server keeps FW1 on the same clock as the domain, which matters later for FSSO/LDAP.
- FW1 is both an NTP **client** (to AD) and an NTP **server** (`server-mode enable`) listening on port5.
- `diagnose sys ntp status` is the quickest check: look for `synchronized: yes`, `reachable` and `selected`.
- The GUI warns that the HTTPS admin port (443) conflicts with the default SSL-VPN port. It doesn't matter yet, but one of them would need to move before using SSL-VPN.
- The 480-minute idle timeout is for the lab only. The default is 5 minutes, and production should stay short.

---

### Lab 3 – Interface Configuration

**Configured**

| Interface | Alias | Role | IP Address | Admin Access |
|---|---|---|---|---|
| port1 | WAN-1 | WAN | 192.168.1.1/24 | PING |
| port2 | WAN-2 | WAN | 192.168.2.1/24 | PING |
| port3 | LAN | LAN | 10.0.1.254/24 | PING |
| port4 | DMZ | DMZ | none yet (VLAN subinterfaces in Section 5) | PING |
| port5 | MGMT | Undefined | 192.168.100.200/24 | PING, HTTPS, HTTP, SSH |

Full config: [`fw1-interfaces.conf`](fw1-interfaces.conf)

**Verification**

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
*(Trimmed to relevant fields.)*

**Takeaways**

- **Aliases are labels, not names.** The alias (e.g. `WAN-1`) appears in the GUI, but the CLI, routes and policies still reference the real interface name (`port1`).
- **Interface roles shape the GUI.** The role (WAN, LAN, DMZ) changes which options the GUI shows. Device identification is enabled on the LAN interface so FW1 can detect connected devices.
- **Admin access is limited to the management port.** WAN, LAN and DMZ only allow PING.
- **port4 has no IP** on purpose: the DMZ servers sit on VLANs, so the addressing goes on VLAN subinterfaces later.
- **Lab vs production:** HTTP admin access is enabled for convenience. In production I would disable HTTP and restrict admin accounts to trusted hosts.

---

## Troubleshooting Log

### `show system interface physical` → "entry is not found in table"

- **Symptom:** Trying to check interface status, the command returned `entry is not found in table`.
- **Cause:** `show system interface` works on the configuration table, so it treated `physical` as the name of an interface to look up. No interface is called "physical".
- **Fix:** Physical interface status is runtime information, which comes from `get`: `get system interface physical`.
- **Lesson:** `show` displays configuration (only values changed from default); `get` displays settings and runtime status.

### NTP not synchronizing with the AD server

- **Symptom:** Right after configuring NTP, `diagnose sys ntp status` showed the server as unreachable:
  ```
  synchronized: no, ntpsync: enabled, server-mode: enabled
  ipv4 server(192.168.100.230) 192.168.100.230 -- unreachable(0x0) S:7 T:8
          no data
  ```
- **Cause:** The AD server (the NTP source) was powered off in the lab.
- **Fix:** Started the AD node. On the next check, the server showed `reachable(0x1) ... selected` and `synchronized: yes`.
- **Lesson:** When a client can't sync, check that the server side is actually up before changing the client's config.
