# Section 4 – Routing & ECMP

**Status:** 🚧 In progress

Static routing on FortiGate: administrative distance, route priority and metric, primary/backup routes, ECMP and load-balancing methods, link health monitoring, policy routing, named addresses in static routes, Internet Services routing, and the RIB vs FIB.

## Labs

- [Lab 1 – Static Routing](#lab-1--static-routing)
- [Lab 2 – Primary & Backup Routes](#lab-2--primary--backup-routes)
- [Lab 3 – ECMP (Equal-Cost Multi-Path)](#lab-3--ecmp-equal-cost-multi-path)

---

### Lab 1 – Static Routing

**Configured on FW1**

- Two default routes: via `192.168.1.254` (port1) and `192.168.2.254` (port2), distance 10
- Static route to the management network `192.168.100.0/24` via port5, gateway `0.0.0.0`

**Verification**

```
FW1 # get router info routing-table all
S*      0.0.0.0/0 [10/0] via 192.168.1.254, port1, [1/0]
                  [10/0] via 192.168.2.254, port2, [1/0]
C       10.0.1.0/24 is directly connected, port3
C       192.168.1.0/24 is directly connected, port1
C       192.168.2.0/24 is directly connected, port2
C       192.168.100.0/24 is directly connected, port5
```

The static route to `192.168.100.0/24` doesn't appear in the routing table. The full database shows why:

```
FW1 # get router info routing-table database
S       192.168.100.0/24 [10/0] is directly connected, port5, [1/0]
C    *> 192.168.100.0/24 is directly connected, port5
```

**Takeaways**

- `[10/0]` = distance/metric, `[1/0]` = priority/weight.
- The static route exists in the RIB but isn't selected (no `*>`): the connected route (distance 0) beats the static route (distance 10).
- A gateway of `0.0.0.0` means the destination is directly connected on that interface, so there's no next hop.
- A route alone doesn't give the LAN access to the AD server: it also needs a firewall policy (port3 → port5) and the AD server's gateway pointing back to FW1 (`192.168.100.200`).

---

### Lab 2 – Primary & Backup Routes

Two ways to make port1 the primary path and port2 the backup, each tested with a failover (port1 set to down).

#### Method 1 – Different distance

**Configured:** port1 route distance 10, port2 route distance 20. Firewall policies LAN → WAN-1 and LAN → WAN-2 (NAT enabled).

```
FW1 # get router info routing-table database
S       0.0.0.0/0 [20/0] via 192.168.2.254, port2, [1/0]
S    *> 0.0.0.0/0 [10/0] via 192.168.1.254, port1, [1/0]
```

Only the primary is active. Traceroutes from PC1–PC3 go via `192.168.1.254`.

**Failover (port1 down):**

```
FW1 # get router info routing-table database
S    *> 0.0.0.0/0 [20/0] via 192.168.2.254, port2, [1/0]
S       0.0.0.0/0 [10/0] via 192.168.1.254, port1 inactive, [1/0]
```

```
PC2$ traceroute -n 8.8.8.8
 1  10.0.1.254      (FW1)
 2  192.168.2.254   (pfSense via port2 = backup)
 3  172.29.129.2
 ...
12  8.8.8.8
```

Forward Traffic logs and FortiView Sessions confirmed new sessions leaving via WAN-2 (port2), with source NAT changing from `192.168.1.1` to `192.168.2.1`. When port1 came back up, traffic returned to port1 automatically.

#### Method 2 – Same distance, different priority

**Configured:** both routes distance 10, port1 priority 1, port2 priority 2.

```
FW1 # get router info routing-table static
S*      0.0.0.0/0 [10/0] via 192.168.1.254, port1, [1/0]
                  [10/0] via 192.168.2.254, port2, [2/0]
```

Both routes are in the routing table, and traffic uses port1 (lower priority). Swapping the priorities moved traffic to port2, with PC1's `tracert` hop 2 changing accordingly. A continuous ping from PC1 lost 1 packet out of 1285 across the changes. The same failover test (port1 down) moved all traffic to port2.

**Takeaways**

- **Distance** decides which routes enter the routing table. **Priority** only breaks ties between routes with the same distance.
- With different distances, the backup isn't in the FIB. It waits in the RIB until the primary goes away.
- With priority, both routes stay in the FIB and new traffic uses the lower priority. Keeping the backup active means traffic arriving on the backup interface passes the reverse path check.
- The LAN → WAN-2 policy must exist before failover, or failover traffic hits the implicit deny.
- Source NAT follows the outgoing interface.
- Failover only happened because the interface went **down**. If port1 stayed up but the path behind it failed, the route would stay active and traffic would be black-holed. Link health monitoring (later in this section) addresses that.
- Lab note: Linux `traceroute` uses UDP probes (seen as UDP/334xx in FortiView), Windows `tracert` uses ICMP.
- Lab note: on FortiOS 7.0.9 (eval) the Application Name column in Forward Traffic stays empty. The Service column shows `PING` instead.

---

### Lab 3 – ECMP (Equal-Cost Multi-Path)

**Configured:** changed the port2 default route's priority from 2 to 1, so both default routes have the same distance (10) **and** the same priority (1).

```
FW1 # get router info routing-table database
S    *> 0.0.0.0/0 [10/0] via 192.168.1.254, port1, [1/0]
     *>           [10/0] via 192.168.2.254, port2, [1/0]
```

Both routes are selected and in the FIB, so FW1 load-balances between them.

#### Default mode: source-IP based

Traceroutes to `8.8.8.8` and `1.1.1.1` from each PC (hop 2 shows the link used):

| Host | To 8.8.8.8 | To 1.1.1.1 |
|---|---|---|
| PC1 (10.0.1.1) | WAN-2 (port2) | WAN-2 (port2) |
| PC2 (10.0.1.2) | WAN-1 (port1) | WAN-1 (port1) |
| PC3 (10.0.1.3) | WAN-2 (port2) | WAN-2 (port2) |

Each host sticks to one link regardless of destination. Confirmed in FortiView Sessions and Forward Traffic logs.

#### Changing the mode: source + destination IP

```
FW1 # config system settings
FW1 (settings) # set v4-ecmp-mode ?
source-ip-based         Select next hop based on source IP.
weight-based            Select next hop based on weight.
usage-based             Select next hop based on usage.
source-dest-ip-based    Select next hop based on both source and destination IPs.
FW1 (settings) # set v4-ecmp-mode source-dest-ip-based
FW1 (settings) # end
```

Now the same host uses different links for different destinations. From PC1:

```
C:\> tracert -d 1.1.1.1
  1  10.0.1.254
  2  192.168.2.254     <- WAN-2

C:\> tracert -d 8.8.8.8
  1  10.0.1.254
  2  192.168.1.254     <- WAN-1
```

Forward Traffic filtered on source `10.0.1.1` showed PC1's sessions split across WAN-1 (SNAT `192.168.1.1`) and WAN-2 (SNAT `192.168.2.1`) depending on destination. PC2 and PC3 showed the same pattern.

#### Failover within ECMP

- Started a continuous ping from PC1 to `1.1.1.1`. Logs showed it leaving via WAN-2.
- Set port2 to down. New log entries switched to WAN-1 (SNAT `192.168.1.1`), and `tracert` hop 2 changed to `192.168.1.254`.
- The ping lost **0 of 3629** packets.
- **Log & Report > Events > System Events** recorded `Link monitor: Interface port2 was turned down`, plus the admin's `Edit system.interface port2`, giving an audit trail of what changed and who changed it.
- The routing database marked the port2 route as `inactive` while the link was down.

**Takeaways**

- ECMP needs routes with the **same distance and the same priority**. Both are then active (`*>`) in the FIB.
- The default mode, **source-IP based**, picks the link from the source IP, so each host always uses the same link. That balances load across hosts, not within one host.
- **Source-destination IP based** uses both addresses, so one host can use both links for different destinations.
- The other modes are **weight-based** (uses route weights) and **usage-based** (fills one link up to a spillover threshold before using the next).
- Load balancing is **per session**, not per packet: the continuous ping stayed on WAN-2 until that link went down.
- If one ECMP member fails, its route goes inactive and traffic moves to the remaining link automatically.

---

## Troubleshooting Log

| Issue | Status |
|---|---|
| [No internet access from FW1 – broken pfSense WAN node](troubleshooting-pfsense-wan.md) | ✅ Resolved |
| [No Forward Traffic logs after rebuilding FW1](#no-forward-traffic-logs-after-rebuilding-fw1) | ✅ Resolved |

### No Forward Traffic logs after rebuilding FW1

- **Symptom:** After FW1 was reset and rebuilt, traffic from the LAN was working and visible in FortiView Sessions, but **Log & Report > Forward Traffic** showed "No results".
- **Cause:** The recreated firewall policies were only logging security events, so plain allowed traffic (pings, traceroutes) wasn't logged.
- **Fix:** Set each policy's **Log Allowed Traffic** to **All Sessions** (CLI: `set logtraffic all`).
- **Lesson:** FortiView Sessions shows the live session table and doesn't depend on logging. Forward Traffic only shows what the policy is configured to log. When sessions exist but logs don't, check the policy's logging setting.
