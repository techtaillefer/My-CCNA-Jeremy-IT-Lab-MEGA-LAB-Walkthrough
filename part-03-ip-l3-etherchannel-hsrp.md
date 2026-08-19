# Part 3 — IP Addressing, Layer 3 EtherChannel, HSRPv2

The Layer 3 underlay. Nothing routes yet (that's Part 5), but every address the routing protocol will advertise gets configured here.

---

## 3.1 — R1 interfaces

```
interface range g0/0/0,g0/1/0
 ip address dhcp
 no shutdown
!
interface g0/0
 ip address 10.0.0.33 255.255.255.252
 no shutdown
!
interface g0/1
 ip address 10.0.0.37 255.255.255.252
 no shutdown
!
interface loopback0
 ip address 10.0.0.76 255.255.255.255
```

**Why the WAN links are DHCP clients:** they face two ISPs. Real edge routers frequently get their public address from the provider, and it lets the lab avoid hard-coding public addressing. You'll see them pick up addresses in the 203.0.113.0/24 range.

**Why a /32 loopback:** loopbacks are software interfaces that never go down as long as the device is up. They're used here as the OSPF Router-ID (Part 5), the DHCP relay target (Part 6.2), and the NTP server address (Part 6.6) — all things that shouldn't depend on any single physical link staying alive. A /32 wastes no addresses.

Loopbacks come up automatically; `no shutdown` is harmless but unnecessary.

**Verify:**

```
show ip interface brief
show ip route connected
```

Confirm G0/0/0 and G0/1/0 actually leased addresses — if they show `unassigned`, the DHCP exchange failed and the Internet routes in Part 5 won't work.

---

## 3.2 — Enable IPv4 routing

**CSW1, CSW2, DSW-A1, DSW-A2, DSW-B1, DSW-B2:**

```
ip routing
```

Multilayer switches ship with routing **disabled**. Without this, SVIs and routed ports exist but the switch drops anything not destined for itself — no inter-VLAN routing, no OSPF adjacency, nothing. It's a single command and it's the single most commonly forgotten one in the whole lab.

Access switches do **not** get `ip routing` — they're pure Layer 2 and use `ip default-gateway` instead (3.11).

---

## 3.3 — Layer 3 EtherChannel (CSW1 ↔ CSW2)

**CSW1:**

```
interface range g1/0/2-3
 no switchport
 channel-group 1 mode desirable
!
interface po1
 ip address 10.0.0.41 255.255.255.252
 no shutdown
```

**CSW2:**

```
interface range g1/0/2-3
 no switchport
 channel-group 1 mode desirable
!
interface po1
 ip address 10.0.0.42 255.255.255.252
 no shutdown
```

### The critical ordering rule

`no switchport` must come **before** `channel-group`. Here's why:

The Port-channel interface inherits its Layer 2 / Layer 3 nature from its **first member port**. If you bundle the ports while they're still switchports, IOS creates a *Layer 2* Port-channel1, and `ip address` on it gets rejected. Fixing that means deleting the port-channel and starting over.

`no switchport` converts a switchport into a **routed port** — it stops participating in VLANs/STP and behaves like a router interface.

**Verify:**

```
show etherchannel summary
show ip interface brief | include Po
ping 10.0.0.42
```

In `show etherchannel summary`, look for the `RU` flag on Po1 — **R** = Layer 3, **U** = in use. `SU` here would mean you got a Layer 2 channel by mistake.

---

## 3.4 / 3.5 — Core switch routed interfaces

**CSW1:**

```
interface g1/0/1
 no switchport
 ip address 10.0.0.34 255.255.255.252
 no shutdown
!
interface g1/1/1
 no switchport
 ip address 10.0.0.45 255.255.255.252
 no shutdown
!
interface g1/1/2
 no switchport
 ip address 10.0.0.49 255.255.255.252
 no shutdown
!
interface g1/1/3
 no switchport
 ip address 10.0.0.53 255.255.255.252
 no shutdown
!
interface g1/1/4
 no switchport
 ip address 10.0.0.57 255.255.255.252
 no shutdown
!
interface loopback0
 ip address 10.0.0.77 255.255.255.255
!
interface range g1/0/4-24
 shutdown
```

**CSW2:**

```
interface g1/0/1
 no switchport
 ip address 10.0.0.38 255.255.255.252
 no shutdown
!
interface g1/1/1
 no switchport
 ip address 10.0.0.61 255.255.255.252
 no shutdown
!
interface g1/1/2
 no switchport
 ip address 10.0.0.65 255.255.255.252
 no shutdown
!
interface g1/1/3
 no switchport
 ip address 10.0.0.69 255.255.255.252
 no shutdown
!
interface g1/1/4
 no switchport
 ip address 10.0.0.73 255.255.255.252
 no shutdown
!
interface loopback0
 ip address 10.0.0.78 255.255.255.255
!
interface range g1/0/4-24
 shutdown
```

The core switches carry no VLANs and no SVIs — every port is either routed or shut down. That's textbook collapsed-core design: the core does fast Layer 3 forwarding and nothing else.

**Verify:**

```
show ip interface brief | exclude unassigned
ping 10.0.0.33
```

---

## 3.6–3.9 — Distribution switch routed uplinks

Each DSW has two routed uplinks (one to each core switch) plus a loopback. The SVIs come later, in the HSRP steps.

**DSW-A1:**

```
interface g1/1/1
 no switchport
 ip address 10.0.0.46 255.255.255.252
 no shutdown
!
interface g1/1/2
 no switchport
 ip address 10.0.0.62 255.255.255.252
 no shutdown
!
interface loopback0
 ip address 10.0.0.79 255.255.255.255
```

**DSW-A2:**

```
interface g1/1/1
 no switchport
 ip address 10.0.0.50 255.255.255.252
 no shutdown
!
interface g1/1/2
 no switchport
 ip address 10.0.0.66 255.255.255.252
 no shutdown
!
interface loopback0
 ip address 10.0.0.80 255.255.255.255
```

**DSW-B1:**

```
interface g1/1/1
 no switchport
 ip address 10.0.0.54 255.255.255.252
 no shutdown
!
interface g1/1/2
 no switchport
 ip address 10.0.0.70 255.255.255.252
 no shutdown
!
interface loopback0
 ip address 10.0.0.81 255.255.255.255
```

**DSW-B2:**

```
interface g1/1/1
 no switchport
 ip address 10.0.0.58 255.255.255.252
 no shutdown
!
interface g1/1/2
 no switchport
 ip address 10.0.0.74 255.255.255.252
 no shutdown
!
interface loopback0
 ip address 10.0.0.82 255.255.255.255
```

**Sanity check before moving on** — from each DSW, ping both core switches:

```
DSW-A1#ping 10.0.0.45
DSW-A1#ping 10.0.0.61
```

Both must succeed. If they don't, the routed uplinks are miswired or one side is still a switchport. Fixing this now is far easier than debugging it through OSPF later.

---

## 3.10 — SRV1

GUI configuration, not CLI. **SRV1 → Config → FastEthernet0 → Static:**

- IPv4 Address: `10.5.0.4`
- Subnet Mask: `255.255.255.0`
- Default Gateway: `10.5.0.1` (the VLAN 30 HSRP VIP)

SRV1 uses the **VIP**, not either physical DSW address — that's the entire point of HSRP.

> SRV1's own DNS setting gets configured in Part 6.

---

## 3.11 — Access switch management SVIs

Access switches are Layer 2 only, so they need one SVI for management plus a default gateway.

**ASW-A1:**

```
interface vlan 99
 ip address 10.0.0.4 255.255.255.240
 no shutdown
!
ip default-gateway 10.0.0.1
```

**ASW-A2:**

```
interface vlan 99
 ip address 10.0.0.5 255.255.255.240
 no shutdown
!
ip default-gateway 10.0.0.1
```

**ASW-A3:**

```
interface vlan 99
 ip address 10.0.0.6 255.255.255.240
 no shutdown
!
ip default-gateway 10.0.0.1
```

**ASW-B1:**

```
interface vlan 99
 ip address 10.0.0.20 255.255.255.240
 no shutdown
!
ip default-gateway 10.0.0.17
```

**ASW-B2:**

```
interface vlan 99
 ip address 10.0.0.21 255.255.255.240
 no shutdown
!
ip default-gateway 10.0.0.17
```

**ASW-B3:**

```
interface vlan 99
 ip address 10.0.0.22 255.255.255.240
 no shutdown
!
ip default-gateway 10.0.0.17
```

### Notes

**"First usable address of the appropriate subnet"** — Office A management is 10.0.0.0/28 → first usable is **10.0.0.1**. Office B is 10.0.0.16/28 → first usable is **10.0.0.17**. Both of these are HSRP VIPs, which is exactly right: the switches' management traffic survives losing a distribution switch.

**`ip default-gateway` only works when `ip routing` is disabled.** On a Layer 2 switch it's the only option. If you ever enable `ip routing` on one of these, the command is silently ignored and you need a static default route instead.

**/28 = 255.255.255.240**, 16 addresses, 14 usable. Worth being fluent in: /28 → .240, /29 → .248, /30 → .252.

**The SVI stays down** until VLAN 99 exists *and* at least one port in VLAN 99 is up (or a trunk carrying VLAN 99 is up). Since Part 2 built those trunks, it should come up right away.

---

## 3.12–3.19 — HSRPv2

Eight HSRP groups: four per office, one per VLAN. Active router alternates to load-share.

### Design summary

| Office | VLAN | Group | VIP | DSW-x1 | DSW-x2 | Active |
|---|---|---|---|---|---|---|
| A | 99 | 1 | 10.0.0.1 | 10.0.0.2 | 10.0.0.3 | **A1** |
| A | 10 | 2 | 10.1.0.1 | 10.1.0.2 | 10.1.0.3 | **A1** |
| A | 20 | 3 | 10.2.0.1 | 10.2.0.2 | 10.2.0.3 | **A2** |
| A | 40 | 4 | 10.6.0.1 | 10.6.0.2 | 10.6.0.3 | **A2** |
| B | 99 | 1 | 10.0.0.17 | 10.0.0.18 | 10.0.0.19 | **B1** |
| B | 10 | 2 | 10.3.0.1 | 10.3.0.2 | 10.3.0.3 | **B1** |
| B | 20 | 3 | 10.4.0.1 | 10.4.0.2 | 10.4.0.3 | **B2** |
| B | 30 | 4 | 10.5.0.1 | 10.5.0.2 | 10.5.0.3 | **B2** |

Default priority is 100, so "5 above the default" = **105**.

---

### Office A — DSW-A1

```
interface vlan 99
 ip address 10.0.0.2 255.255.255.240
 standby version 2
 standby 1 ip 10.0.0.1
 standby 1 priority 105
 standby 1 preempt
!
interface vlan 10
 ip address 10.1.0.2 255.255.255.0
 standby version 2
 standby 2 ip 10.1.0.1
 standby 2 priority 105
 standby 2 preempt
!
interface vlan 20
 ip address 10.2.0.2 255.255.255.0
 standby version 2
 standby 3 ip 10.2.0.1
!
interface vlan 40
 ip address 10.6.0.2 255.255.255.0
 standby version 2
 standby 4 ip 10.6.0.1
```

### Office A — DSW-A2

```
interface vlan 99
 ip address 10.0.0.3 255.255.255.240
 standby version 2
 standby 1 ip 10.0.0.1
!
interface vlan 10
 ip address 10.1.0.3 255.255.255.0
 standby version 2
 standby 2 ip 10.1.0.1
!
interface vlan 20
 ip address 10.2.0.3 255.255.255.0
 standby version 2
 standby 3 ip 10.2.0.1
 standby 3 priority 105
 standby 3 preempt
!
interface vlan 40
 ip address 10.6.0.3 255.255.255.0
 standby version 2
 standby 4 ip 10.6.0.1
 standby 4 priority 105
 standby 4 preempt
```

### Office B — DSW-B1

```
interface vlan 99
 ip address 10.0.0.18 255.255.255.240
 standby version 2
 standby 1 ip 10.0.0.17
 standby 1 priority 105
 standby 1 preempt
!
interface vlan 10
 ip address 10.3.0.2 255.255.255.0
 standby version 2
 standby 2 ip 10.3.0.1
 standby 2 priority 105
 standby 2 preempt
!
interface vlan 20
 ip address 10.4.0.2 255.255.255.0
 standby version 2
 standby 3 ip 10.4.0.1
!
interface vlan 30
 ip address 10.5.0.2 255.255.255.0
 standby version 2
 standby 4 ip 10.5.0.1
```

### Office B — DSW-B2

```
interface vlan 99
 ip address 10.0.0.19 255.255.255.240
 standby version 2
 standby 1 ip 10.0.0.17
!
interface vlan 10
 ip address 10.3.0.3 255.255.255.0
 standby version 2
 standby 2 ip 10.3.0.1
!
interface vlan 20
 ip address 10.4.0.3 255.255.255.0
 standby version 2
 standby 3 ip 10.4.0.1
 standby 3 priority 105
 standby 3 preempt
!
interface vlan 30
 ip address 10.5.0.3 255.255.255.0
 standby version 2
 standby 4 ip 10.5.0.1
 standby 4 priority 105
 standby 4 preempt
```

---

### HSRP concepts that matter

**`standby version 2` first.** Changing versions resets the group, so set it before the rest of the group config. HSRPv2 is required here because v1 only supports group numbers 0–255 and, more importantly, v1 can't do millisecond timers or IPv6. v2 also uses a different multicast address (224.0.0.102 vs v1's 224.0.0.2) and a different virtual MAC format (`0000.0C9F.Fxxx` vs `0000.0C07.ACxx`). **A v1 and v2 router will never form a working pair** — mismatched versions is a classic "why is everything Active" symptom.

**Preemption is not the default.** Without `standby x preempt`, a router that comes back after a failure stays Standby even though it has the better priority. The lab only asks for preempt on the intended-Active router, which is enough to guarantee the design holds after a reboot.

**Higher priority wins.** On a tie, the highest interface IP wins. Since DSW-x2 always has the higher address (.3 > .2), leaving both at default 100 would make DSW-x2 Active everywhere — which is why groups 1 and 2 need the explicit priority bump on DSW-x1.

**Group numbers are per-VLAN, not global.** Office A group 1 and Office B group 1 are entirely independent because they're on different VLANs on different switches. Reusing 1–4 in both offices is deliberate and correct.

**Load sharing:** A1 is Active for VLANs 99 and 10, A2 for 20 and 40. Both distribution switches carry traffic instead of one sitting idle. Part 4 aligns the STP root bridge with these choices so the Layer 2 and Layer 3 paths agree.

---

### Verify

```
show standby brief
```

Expected on DSW-A1:

```
Interface  Grp  Pri P State    Active          Standby         Virtual IP
Vl10       2    105 P Active   local           10.1.0.3        10.1.0.1
Vl20       3    100   Standby  10.2.0.3        local           10.2.0.1
Vl40       4    100   Standby  10.6.0.3        local           10.6.0.1
Vl99       1    105 P Active   local           10.0.0.3        10.0.0.1
```

The `P` in the third column is preemption enabled. Every group should show exactly one Active and one Standby — two Actives means the routers can't hear each other (version mismatch, VLAN not allowed on the trunk, or SVI down).

More detail:

```
show standby
show standby vlan 10
```

**Failover test:**

```
DSW-A1(config)#interface vlan 10
DSW-A1(config-if)#shutdown
```

DSW-A2 should take over within ~10 seconds (3× the 3-second hello). `no shutdown` and, thanks to preempt, DSW-A1 reclaims Active. Ping the VIP from PC1 continuously while doing this — you should lose one or two packets at most.

---

## Part 3 verification checklist

```
show ip interface brief | exclude unassigned
show ip route connected
show etherchannel summary          ! Po1 = RU on the core switches
show standby brief                 ! all 8 groups, one Active each
show vlan brief                    ! SVIs need their VLANs to exist
ping 10.0.0.1                      ! VIP reachable from an access switch
```

---

**Next:** [Part 4 — Rapid Spanning Tree Protocol →](part-04-rapid-pvst.md)
