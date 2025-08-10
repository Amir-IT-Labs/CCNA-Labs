# Floating Static Route Lab — Full Configurations

**File:** `floating-static-lab.md`

**Purpose:** Demonstrate primary dynamic routing (OSPF) between R1 and R2 with floating static routes (AD=111) as backups through provider routers (SPR1/SPR2). When the primary 10.0.0.0/30 link is shut down, traffic should fail over to the static path.

---

## Topology (written)

- **R1** — Core router for `10.0.1.0/24` LAN
- **R2** — Core router for `10.0.2.0/24` LAN
- **R1 <-> R2** primary link: `10.0.0.0/30` (OSPF)
- **Alternate path**: R1 -> SPR1 -> SPR2 -> R2 using `203.0.113.0/30` blocks and `192.168.1.0/30` between SPRs
- **ISPBR1 / ISPBR2** simulate upstream ISPs (basic configs included)

## IP Addressing Table

| Device | Interface             | IP Address   | Subnet Mask     | Notes                      |
| ------ | --------------------- | ------------ | --------------- | -------------------------- |
| R1     | G0/2/0 (to R2)        | 10.0.0.1     | 255.255.255.252 | primary OSPF link          |
| R1     | G0/1 (LAN)            | 10.0.1.254   | 255.255.255.0   | R1 LAN gateway             |
| R1     | G0/0/0 (to ISP/SPR1)  | 203.0.113.2  | 255.255.255.252 | toward SPR1 / ISP segment  |
| R1     | G0/1/0 (spare/ISP)    | 203.0.113.10 | 255.255.255.252 | optional provider link     |
| R2     | G0/2/0 (to R1)        | 10.0.0.2     | 255.255.255.252 | primary OSPF link          |
| R2     | G0/1 (LAN)            | 10.0.2.254   | 255.255.255.0   | R2 LAN gateway             |
| R2     | G0/0/0 (to ISP/SPR2)  | 203.0.113.6  | 255.255.255.252 | toward SPR2 / ISP segment  |
| R2     | G0/1/0 (spare/ISP)    | 203.0.113.14 | 255.255.255.252 | optional provider link     |
| SPR1   | G0/0/0 (to R1 region) | 203.0.113.1  | 255.255.255.252 | connects toward R1         |
| SPR1   | G0/1/0 (to SPR2)      | 192.168.1.1  | 255.255.255.252 | SPR backbone               |
| SPR2   | G0/0/0 (to R2 region) | 203.0.113.5  | 255.255.255.252 | connects toward R2         |
| SPR2   | G0/1/0 (to SPR1)      | 192.168.1.2  | 255.255.255.252 | SPR backbone               |
| ISPBR1 | G0/0/0                | 203.0.113.9  | 255.255.255.252 | upstream/loopbacks present |
| ISPBR2 | G0/0/0                | 203.0.113.13 | 255.255.255.252 | upstream/loopbacks present |

---

## Design Notes

- OSPF process `1` runs on the **primary** 10.0.0.0/30 link and on the local LAN segments on R1 and R2 so they learn each other's LANs dynamically.
- Floating static routes are added on R1 and R2 with **administrative distance 111** (higher than OSPF's 110) so they activate only when the OSPF route is removed.
- SPR1 and SPR2 are simple static-route-based transit routers that forward traffic between the `203.0.113.x` networks and the `192.168.1.x` backbone.

---

## Full Device Configurations

> **Note:** These configurations are written as complete examples for each device. Adjust `enable secret` and passwords to your lab policy. Replace any `domain` or `username` lines with your own if required.

### R1 (full)

```text
! R1 — core router for 10.0.1.0/24
hostname R1
no ip domain-lookup
!
enable secret cisco
service password-encryption
!
interface GigabitEthernet0/2/0
 description Link to R2 (primary OSPF)
 ip address 10.0.0.1 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/1
 description LAN 10.0.1.0/24
 ip address 10.0.1.254 255.255.255.0
 no shutdown
!
interface GigabitEthernet0/0/0
 description Link to SPR1 / Provider
 ip address 203.0.113.2 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/1/0
 description Optional provider link
 ip address 203.0.113.10 255.255.255.252
 no shutdown
!
router ospf 1
 log-adjacency-changes
 network 10.0.0.0 0.0.0.3 area 0
 network 10.0.1.0 0.0.0.255 area 0
!
! Floating static route (backup) via SPR1
ip route 10.0.2.0 255.255.255.0 203.0.113.1 111
!
! Basic management lines
line con 0
 logging synchronous
line vty 0 4
 password vtypass
 login
 transport input ssh
!
end
```

### R2 (full)

```text
! R2 — core router for 10.0.2.0/24
hostname R2
no ip domain-lookup
!
enable secret cisco
service password-encryption
!
interface GigabitEthernet0/2/0
 description Link to R1 (primary OSPF)
 ip address 10.0.0.2 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/1
 description LAN 10.0.2.0/24
 ip address 10.0.2.254 255.255.255.0
 no shutdown
!
interface GigabitEthernet0/0/0
 description Link to SPR2 / Provider
 ip address 203.0.113.6 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/1/0
 description Optional provider link
 ip address 203.0.113.14 255.255.255.252
 no shutdown
!
router ospf 1
 log-adjacency-changes
 network 10.0.0.0 0.0.0.3 area 0
 network 10.0.2.0 0.0.0.255 area 0
!
! Floating static route (backup) via SPR2
ip route 10.0.1.0 255.255.255.0 203.0.113.5 111
!
line con 0
 logging synchronous
line vty 0 4
 password vtypass
 login
 transport input ssh
!
end
```

### SPR1 (full)

```text
! SPR1 — service provider router 1
hostname SPR1
no ip domain-lookup
!
enable secret cisco
service password-encryption
!
interface GigabitEthernet0/0/0
 description Link toward R1/provider
 ip address 203.0.113.1 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/1/0
 description SPR backbone to SPR2
 ip address 192.168.1.1 255.255.255.252
 no shutdown
!
! Static routes toward R1 and R2 networks
ip route 10.0.1.0 255.255.255.0 203.0.113.2
ip route 10.0.2.0 255.255.255.0 192.168.1.2
! Default to SPR2 if needed (optional)
ip route 0.0.0.0 0.0.0.0 192.168.1.2
!
line con 0
 logging synchronous
line vty 0 4
 password vtypass
 login
!
end
```

### SPR2 (full)

```text
! SPR2 — service provider router 2
hostname SPR2
no ip domain-lookup
!
enable secret cisco
service password-encryption
!
interface GigabitEthernet0/0/0
 description Link toward R2/provider
 ip address 203.0.113.5 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/1/0
 description SPR backbone to SPR1
 ip address 192.168.1.2 255.255.255.252
 no shutdown
!
! Static routes toward R2 and R1 networks
ip route 10.0.2.0 255.255.255.0 203.0.113.6
ip route 10.0.1.0 255.255.255.0 192.168.1.1
! Default to SPR1 if needed (optional)
ip route 0.0.0.0 0.0.0.0 192.168.1.1
!
line con 0
 logging synchronous
line vty 0 4
 password vtypass
 login
!
end
```

### ISPBR1 (sample)

```text
hostname ISPBR1
no ip domain-lookup
!
interface GigabitEthernet0/0/0
 ip address 203.0.113.9 255.255.255.252
 no shutdown
!
interface Loopback0
 ip address 1.1.1.1 255.255.255.255
!
ip route 0.0.0.0 0.0.0.0 203.0.113.10
!
end
```

### ISPBR2 (sample)

```text
hostname ISPBR2
no ip domain-lookup
!
interface GigabitEthernet0/0/0
 ip address 203.0.113.13 255.255.255.252
 no shutdown
!
interface Loopback0
 ip address 2.2.2.2 255.255.255.255
!
ip route 0.0.0.0 0.0.0.0 203.0.113.14
!
end
```

---

## Verification & Tests (Commands)

1. **Show OSPF and interfaces**

```bash
show ip ospf neighbor
show ip ospf interface brief
show ip interface brief
```

2. **Routing table check (before failure)**

```bash
show ip route
! Expect O entries for remote LANs, e.g. O 10.0.2.0/24 via 10.0.0.2
```

3. **Bring down primary link (simulate failure)**

On R1 (or R2):

```bash
conf t
interface gigabitEthernet0/2/0
 shutdown
end
```

4. **Routing table check (after failure)**

```bash
show ip route
! Expect S 10.0.2.0/24 [111/0] via 203.0.113.1 on R1
! and S 10.0.1.0/24 [111/0] via 203.0.113.5 on R2
```

5. **Ping tests**

- From a host behind R1 (e.g. 10.0.1.10) ping 10.0.2.10 — should succeed before and after failover (using different paths).
- Use `traceroute` to confirm path changes (direct via 10.0.0.0/30 before, via SPRs after).

---

## Troubleshooting & Tips

- Make sure `ip route` next-hops exist and are reachable from the router that points to them (e.g., R1 must be able to reach 203.0.113.1). If the next-hop is not reachable, the static route will not install.
- If you prefer next-hop interface instead of next-hop IP, ensure the interface is up and the IP reachability rules are satisfied.
- If using redistribution or more complex topologies, consider route-maps or tracking objects with `ip sla` and `track` to create more granular failover conditions.

---

## Appendix: Why AD=111?

- OSPF default administrative distance = 110.
- Static default administrative distance = 1 (most preferred).
- By configuring the static backup with AD=111, it becomes less preferred than OSPF and will only activate when OSPF withdraws the route.

---

*End of file.*

