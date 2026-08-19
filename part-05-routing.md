# Part 5 — Static and Dynamic Routing

OSPFv2 across the whole Layer 3 core, plus redundant default routes out to the Internet. This is the part where the network stops being a collection of subnets and starts being a network.

---

## 5.1 — OSPF requirements breakdown

| Requirement | Command |
|---|---|
| Process ID 1, Area 0 | `router ospf 1` / `... area 0` |
| RID = loopback IP | `router-id x.x.x.x` |
| Switches: match exact interface IP | `network x.x.x.x 0.0.0.0 area 0` |
| R1: interface-mode OSPF | `ip ospf 1 area 0` |
| Loopbacks in OSPF, passive | `passive-interface loopback0` |
| DSW SVIs passive (except VLAN 99) | `passive-interface vlan X` |
| No DR/BDR on physical links | `ip ospf network point-to-point` |

---

## 5.2 — R1

```
router ospf 1
 router-id 10.0.0.76
 passive-interface loopback0
!
interface loopback0
 ip ospf 1 area 0
!
interface range g0/0-1
 ip ospf 1 area 0
 ip ospf network point-to-point
```

**Only the LAN-facing interfaces** (G0/0 and G0/1) run OSPF. G0/0/0 and G0/1/0 face the Internet — you don't peer OSPF with an ISP, and advertising your internal routes to them would be a bad day.

---

## 5.3 — CSW1

```
router ospf 1
 router-id 10.0.0.77
 passive-interface loopback0
 network 10.0.0.34 0.0.0.0 area 0
 network 10.0.0.41 0.0.0.0 area 0
 network 10.0.0.45 0.0.0.0 area 0
 network 10.0.0.49 0.0.0.0 area 0
 network 10.0.0.53 0.0.0.0 area 0
 network 10.0.0.57 0.0.0.0 area 0
 network 10.0.0.77 0.0.0.0 area 0
!
interface range g1/0/1,g1/1/1-4
 ip ospf network point-to-point
```

## 5.4 — CSW2

```
router ospf 1
 router-id 10.0.0.78
 passive-interface loopback0
 network 10.0.0.38 0.0.0.0 area 0
 network 10.0.0.42 0.0.0.0 area 0
 network 10.0.0.61 0.0.0.0 area 0
 network 10.0.0.65 0.0.0.0 area 0
 network 10.0.0.69 0.0.0.0 area 0
 network 10.0.0.73 0.0.0.0 area 0
 network 10.0.0.78 0.0.0.0 area 0
!
interface range g1/0/1,g1/1/1-4
 ip ospf network point-to-point
```

**Note the exclusion:** `Port-channel1` is deliberately *not* in the `ip ospf network point-to-point` range. The requirement calls that out explicitly — leave Po1 at the default broadcast network type. Trying to change it on the L3 port-channel causes the adjacency to break in Packet Tracer.

---

## 5.5 — DSW-A1

```
router ospf 1
 router-id 10.0.0.79
 passive-interface loopback0
 passive-interface vlan 10
 passive-interface vlan 20
 passive-interface vlan 40
 network 10.0.0.46 0.0.0.0 area 0
 network 10.0.0.62 0.0.0.0 area 0
 network 10.0.0.79 0.0.0.0 area 0
 network 10.0.0.2 0.0.0.0 area 0
 network 10.1.0.2 0.0.0.0 area 0
 network 10.2.0.2 0.0.0.0 area 0
 network 10.6.0.2 0.0.0.0 area 0
!
interface range g1/1/1-2
 ip ospf network point-to-point
```

## 5.6 — DSW-A2

```
router ospf 1
 router-id 10.0.0.80
 passive-interface loopback0
 passive-interface vlan 10
 passive-interface vlan 20
 passive-interface vlan 40
 network 10.0.0.50 0.0.0.0 area 0
 network 10.0.0.66 0.0.0.0 area 0
 network 10.0.0.80 0.0.0.0 area 0
 network 10.0.0.3 0.0.0.0 area 0
 network 10.1.0.3 0.0.0.0 area 0
 network 10.2.0.3 0.0.0.0 area 0
 network 10.6.0.3 0.0.0.0 area 0
!
interface range g1/1/1-2
 ip ospf network point-to-point
```

## 5.7 — DSW-B1

```
router ospf 1
 router-id 10.0.0.81
 passive-interface loopback0
 passive-interface vlan 10
 passive-interface vlan 20
 passive-interface vlan 30
 network 10.0.0.54 0.0.0.0 area 0
 network 10.0.0.70 0.0.0.0 area 0
 network 10.0.0.81 0.0.0.0 area 0
 network 10.0.0.18 0.0.0.0 area 0
 network 10.3.0.2 0.0.0.0 area 0
 network 10.4.0.2 0.0.0.0 area 0
 network 10.5.0.2 0.0.0.0 area 0
!
interface range g1/1/1-2
 ip ospf network point-to-point
```

## 5.8 — DSW-B2

```
router ospf 1
 router-id 10.0.0.82
 passive-interface loopback0
 passive-interface vlan 10
 passive-interface vlan 20
 passive-interface vlan 30
 network 10.0.0.58 0.0.0.0 area 0
 network 10.0.0.74 0.0.0.0 area 0
 network 10.0.0.82 0.0.0.0 area 0
 network 10.0.0.19 0.0.0.0 area 0
 network 10.3.0.3 0.0.0.0 area 0
 network 10.4.0.3 0.0.0.0 area 0
 network 10.5.0.3 0.0.0.0 area 0
!
interface range g1/1/1-2
 ip ospf network point-to-point
```

---

## Concepts behind the config

### `network x.x.x.x 0.0.0.0 area 0`

The wildcard mask `0.0.0.0` means "match this one exact address." It's not the subnet — it's the **interface's own IP**. This style is verbose but unambiguous: you can read the OSPF config and know exactly which interfaces are enabled, with no risk of accidentally enabling OSPF on something you didn't intend.

Tip for building these lists quickly:

```
show ip interface brief | exclude unassigned
```

Copy the addresses straight out of that output.

Remember: the `network` statement selects **which interfaces to enable OSPF on**. It does not decide what gets advertised — OSPF advertises the *actual subnet configured on the matched interface*. That's why `network 10.1.0.2 0.0.0.0` correctly advertises `10.1.0.0/24`.

### Passive interfaces

A passive interface still gets **advertised** into OSPF, but stops sending hellos — so no adjacency can form over it.

Use it wherever there's no legitimate OSPF neighbor:

- **Loopbacks** — nobody's on the other end.
- **User VLAN SVIs (10, 20, 30, 40)** — only PCs and phones live there. Sending hellos to them wastes bandwidth and, worse, would let anyone with a laptop and a Linux box form an adjacency and inject routes.

**VLAN 99 is deliberately not passive.** DSW-x1 and DSW-x2 are OSPF neighbors across the Layer 2 Po1 link via their management SVIs, giving the pair a backup adjacency if a distribution switch loses both routed uplinks to the core.

Alternative style: `passive-interface default` then `no passive-interface <x>` for the few that need adjacencies. Cleaner in production, but the lab asks for explicit statements.

### Point-to-point network type

On a broadcast segment, OSPF elects a DR and BDR to cut down on LSA flooding. Every routed link here is a /30 with exactly two routers on it — a DR election is pure overhead and adds ~40 seconds of wait time on startup.

```
interface g1/1/1
 ip ospf network point-to-point
```

**This must match on both ends.** Mismatched network types = no adjacency, and the symptom (stuck in `EXSTART`/`INIT`, or no neighbor at all) doesn't obviously point at the cause. If a neighbor is missing, check the network type on both sides first.

Loopbacks default to network type `LOOPBACK` and are always advertised as /32 regardless of configured mask — normal, expected, not a bug.

### Router-ID

```
router-id 10.0.0.76
```

Manually setting the RID to the loopback IP makes the topology readable — `show ip ospf neighbor` output maps directly to your device list. Without it, OSPF picks the highest loopback IP, then the highest active interface IP.

RID changes don't take effect on a running process until you either `clear ip ospf process` or reload. If you set the RID *after* adjacencies form, run:

```
clear ip ospf process
```

---

## Verification

```
show ip ospf neighbor
```

Expected neighbor counts:

| Device   | Neighbors                                   |
| -------- | ------------------------------------------- |
| R1       | 2 (CSW1, CSW2)                              |
| CSW1     | 6 (R1, CSW2, all four DSWs)                 |
| CSW2     | 6 (R1, CSW1, all four DSWs)                 |
| Each DSW | 3 (CSW1, CSW2, its partner DSW via VLAN 99) |

State should be **FULL**. With point-to-point network types you'll see `FULL/  -` (no DR/BDR role) rather than `FULL/DR` or `FULL/BDR` — that's the confirmation the network type took effect.

```
show ip ospf interface brief
show ip ospf interface g1/1/1
show ip route ospf
show ip protocols
```

`show ip protocols` conveniently lists the RID, the networks, and every passive interface in one place — the fastest way to audit this part.

**Full reachability test:** from PC1, ping SRV1 at `10.5.0.4`. That single ping traverses ASW → DSW → CSW → DSW → ASW across both offices. If it works, OSPF is solid.

---

## 5.9 — Static default routes

Requirements:
- One default route per Internet connection, both **recursive**
- The G0/1/0 route is a **floating static** with AD = default + 1
- R1 becomes an **ASBR** advertising the default into OSPF

**R1:**

```
ip route 0.0.0.0 0.0.0.0 203.0.113.1
ip route 0.0.0.0 0.0.0.0 203.0.113.5 2
!
router ospf 1
 default-information originate
```

### Recursive vs directly-attached vs fully-specified

| Type              | Syntax                                        | Behavior                                                                       |
| ----------------- | --------------------------------------------- | ------------------------------------------------------------------------------ |
| **Recursive**     | `ip route 0.0.0.0 0.0.0.0 203.0.113.1`        | Next-hop IP only — router must do a second lookup to find the exit interface   |
| Directly attached | `ip route 0.0.0.0 0.0.0.0 g0/0/0`             | Exit interface only — risky on multi-access links (ARPs for every destination) |
| Fully specified   | `ip route 0.0.0.0 0.0.0.0 g0/0/0 203.0.113.1` | Both — most explicit                                                           |

The requirement says recursive, so **next-hop IP only**. (Part 8 asks for a fully-specified route for IPv6, so the lab makes you demonstrate both.)

### Floating static

Default AD for a static route is **1**. Adding `2` at the end raises the AD on the backup route, so it doesn't appear in the routing table at all while the primary is up. If G0/0/0 fails and 203.0.113.1 becomes unreachable, the primary is withdrawn and the AD 2 route installs itself.

Check the table:

```
R1#show ip route static
S*   0.0.0.0/0 [1/0] via 203.0.113.1
```

Only one default route should show. The AD 2 route is in the config but not the RIB — that's correct behavior, not a missing config.

### `default-information originate`

This is what makes R1 an **ASBR** (Autonomous System Boundary Router). It injects the default route into OSPF as an **external Type 5 LSA**, so every other router learns `0.0.0.0/0` pointing back at R1.

On other routers:

```
CSW1#show ip route | include 0.0.0.0
O*E2 0.0.0.0/0 [110/1] via 10.0.0.33, 00:05:12, GigabitEthernet1/0/1
```

`O*E2` — OSPF external type 2. The `*` marks it as a candidate default.

### The Packet Tracer failover quirk

`default-information originate` only injects the default **while R1 actually has one in its own routing table**. When G0/0/0 goes down, the primary static is withdrawn and the floating route installs — but PT doesn't always re-trigger the LSA generation, so the rest of the network loses its default route.

Real IOS solves this with:

```
router ospf 1
 default-information originate always
```

`always` injects the default regardless of whether R1 has one locally. **This command doesn't exist in Packet Tracer.** The workaround, needed each time you flip the interface in Part 6.12:

```
router ospf 1
 no default-information originate
 default-information originate
```

Annoying, but understanding *why* it's needed is the actually useful takeaway.

---

## Part 5 verification checklist

```
show ip ospf neighbor              ! all FULL, correct counts, "/  -" state
show ip protocols                  ! RID, networks, passive interfaces
show ip ospf interface brief
show ip route ospf                 ! all subnets learned
show ip route static               ! one default route only
show ip route | include 0.0.0.0    ! O*E2 on non-R1 devices
```

From PC1: `ping 10.5.0.4` (SRV1, across both offices).

---

**Next:** [Part 6 — Network Services →](part-06-network-services.md)
