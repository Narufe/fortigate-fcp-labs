# Section 4 – Routing & ECMP

**Status:** 🚧 In progress

Static routing on FortiGate: administrative distance, route priority and metric, primary/backup routes, ECMP and load-balancing methods, link health monitoring, policy routing, named addresses in static routes, Internet Services routing, and the RIB vs FIB.

## Labs

- [Lab 1 – Static Routing](#lab-1--static-routing)
- [Lab 2 – Primary & Backup Routes](#lab-2--primary--backup-routes)

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

## Troubleshooting Log

| Issue | Status |
|---|---|
| [No internet access from FW1 – broken pfSense WAN node](troubleshooting-pfsense-wan.md) | ✅ Resolved |
