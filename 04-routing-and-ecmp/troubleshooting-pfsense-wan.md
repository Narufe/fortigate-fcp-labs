# Troubleshooting: No Internet Access from FW1

**Section:** 4 – Routing & ECMP (Route Priority lab)
**Status:** ✅ Resolved

## Summary

While testing route priority, FW1 could not reach `8.8.8.8`. FW1's routing was correct. The fault was upstream: the pfSense router acting as the WAN had no IPv4 address on its internet-facing interface, because its node instance in EVE-NG was in a broken state. Wiping the node back to its base image fixed it.

## Environment

- EVE-NG running in VMware Workstation (Linux host)
- **Internet** node = EVE-NG Cloud2 → `pnet2` → VMware `vmnet8` (NAT, `172.29.129.0/24`, gateway `.2`, DHCP enabled)
- **WAN** node = pfSense 2.6.0 (image provided with the course), EVE-NG node ID 18

## Symptom

Ping to `8.8.8.8` failed, and the traceroute stopped at pfSense with `!H` (host unreachable):

```
FW1 # execute traceroute 8.8.8.8
 1  192.168.2.254  0.487 ms  1.635 ms  1.471 ms
 2  192.168.2.254  2.555 ms !H  0.569 ms !H  0.549 ms !H
```

`!H` means pfSense itself was replying "I can't reach that", so FW1 had already handed the packet off successfully.

![FW1 traceroute failing](images/fw1-traceroute-fail.png)

## Troubleshooting Steps

### 1. Verify FW1's routing

Both default routes were installed, with the port2 route preferred (same distance, lower priority). FW1 was configured correctly.

![FW1 routing table](images/fw1-routing-table.png)

### 2. Test the next hop

`execute ping 192.168.2.254` succeeded, so FW1 → pfSense was working. The problem was beyond FW1.

### 3. Test beyond pfSense

`execute ping 172.29.129.2` (the VMware NAT gateway) failed. pfSense couldn't reach anything on its WAN side.

### 4. Verify the EVE-NG cloud mapping

On the EVE-NG host:

- `bridge link` showed pfSense's e0 (`vunl0_18_0`) attached to `pnet2`, i.e. Cloud2.
- `ip -br addr` showed `pnet0` = bridged home network, `pnet1` = host-only management, `pnet2` = no IP (VMware NAT).

The lab wiring was correct.

### 5. Packet capture on the cloud – a red herring

`tcpdump -ni pnet2` showed a constant stream of:

```
ARP, Request who-has 192.168.0.96 tell 192.168.0.194, length 28
```

I first assumed this was pfSense looking for a gateway that doesn't exist in my environment. Before acting on it, I checked the other node on the same cloud, **EXT-PC**. It had a static IP of `192.168.0.194` and MAC `50:00:00:0f:00:00`. EVE-NG builds MACs from the node ID (`0x0f` = node 15 = EXT-PC), so the ARPs came from EXT-PC, not pfSense.

![EXT-PC interface config](images/ext-pc-ifconfig.png)

**Lesson:** identify the source of captured traffic before drawing conclusions.

### 6. Capture only pfSense's traffic

pfSense is node 18 (`0x12`), so its e0 MAC is `50:00:00:12:00:00`:

```
tcpdump -eni pnet2 ether host 50:00:00:12:00:00
```

It showed only IPv6 link-local traffic (multicast listener reports, neighbor solicitation) and **no IPv4 at all**: no DHCP requests, no ARP. The WAN interface was up but had no IPv4 address.

The pfSense boot log also showed errors, and the console asked for a login instead of showing the usual menu:

```
Starting webConfigurator...failed!
ERROR: It was not possible to identify which pfSense kernel is installed
fcgicli: Could not connect to server(/var/run/php-fpm.socket).
```

![pfSense boot errors](images/pfsense-boot-failed.png)

The node's RAM (2048 MB) was checked and ruled out as a cause.

### 7. Fix: Wipe the node

In EVE-NG: **Stop** the WAN node → right-click → **Wipe** → **Start**.

Wipe discards the node's local changes and resets it to the base image, while keeping its ID and cable connections.

## Verification

pfSense booted cleanly, webConfigurator started, the console menu appeared, and the WAN received a DHCP lease from VMware NAT:

```
INTERNET (wan)  -> vtnet0  -> v4/DHCP4: 172.29.129.128/24
LAN1 (lan)      -> vtnet1  -> v4: 192.168.1.254/24
LAN2 (opt1)     -> vtnet2  -> v4: 192.168.2.254/24
```

![pfSense healthy console](images/pfsense-console-fixed.png)

The capture now showed pfSense exchanging traffic with the VMware NAT gateway (MAC prefix `00:50:56` = VMware), including DNS queries to internet servers. From FW1, ping and traceroute to `8.8.8.8` succeeded:

```
FW1 # execute traceroute 8.8.8.8
 1  192.168.2.254   0.721 ms   ← pfSense via port2 (priority 1 route)
 2  172.29.129.2    0.989 ms   ← VMware NAT gateway
 3  192.168.0.1     1.628 ms   ← home router
 ...
12  8.8.8.8 <dns.google>  5.644 ms
```

Hop 1 also confirms the route priority lab: with equal distance, FW1 uses the port2 default route (priority 1) over port1 (priority 10).

![FW1 traceroute working](images/fw1-traceroute-success.png)

## Root Cause

The pfSense node's local instance had drifted into a broken state (boot errors, WAN not requesting a DHCP address). The base image itself was fine: its WAN is set to DHCP, so once reset, it picked up an address from VMware NAT automatically.

## Lessons Learned

- Work outward one hop at a time: local routing → next hop → beyond the next hop.
- `!H` in a traceroute comes from the device that returned it, which pinpoints where traffic stops.
- In packet captures, confirm the source (MAC/IP) before blaming a device.
- In EVE-NG, **Wipe** is a quick fix for a misbehaving node, but it discards any changes made to that node.

## Commands Used

| Where | Command | Purpose |
|---|---|---|
| FW1 | `execute ping` / `execute traceroute` | Test reachability and find where traffic stops |
| FW1 | `get router info routing-table all` | Check installed routes |
| EVE-NG host | `bridge link` | Map lab node interfaces to clouds (pnets) |
| EVE-NG host | `ip -br addr` | Identify which pnet is on which network |
| EVE-NG host | `tcpdump -ni pnet2` | Capture traffic on the cloud |
| EVE-NG host | `tcpdump -eni pnet2 ether host <MAC>` | Capture one node's traffic only |
