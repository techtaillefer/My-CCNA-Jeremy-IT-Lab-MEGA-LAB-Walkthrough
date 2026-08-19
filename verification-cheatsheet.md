# Verification Cheat Sheet

Every `show` command from this lab, grouped by topic, with what "good" looks like.

---

## Layer 2

```
show vlan brief                          ! VLANs exist with correct names
show interfaces trunk                    ! native VLAN, allowed list, trunking status
show interfaces g0/1 switchport          ! mode, negotiation, native/access VLAN
show interfaces status                   ! up/down, VLAN, speed, duplex
show mac address-table
```

**Look for:** trunk ports listed under "Port / Mode / Encapsulation / Status / Native vlan" as `on / 802.1q / trunking / 1000`.

---

## EtherChannel

```
show etherchannel summary
show etherchannel port-channel
show interfaces po1
```

**Flag reference:**

| Flag | Meaning |
|---|---|
| `S` | Layer 2 channel |
| `R` | Layer 3 channel |
| `U` | In use |
| `D` | Down |
| `(P)` | Member bundled in the port-channel |
| `(I)` | Member independent — not bundled |
| `(s)` | Member suspended — config mismatch |

Office A/B DSW pairs → `Po1(SU)`. CSW1/CSW2 → `Po1(RU)`.

---

## VTP

```
show vtp status
show vtp counters
```

**Look for:** matching domain name, VTP version 2 running, correct operating mode, and identical configuration revision numbers across the office.

---

## Spanning Tree

```
show spanning-tree summary               ! mode, PortFast/BPDU Guard counts
show spanning-tree root                  ! one line per VLAN — fastest overall check
show spanning-tree vlan 10
show spanning-tree interface f0/1 detail
show interfaces status err-disabled
```

**Look for:** `This bridge is the root` on the DSW that is HSRP Active for that VLAN. Displayed priority = configured priority + VLAN ID (so VLAN 10 at priority 0 shows as 10).

---

## HSRP

```
show standby brief
show standby
show standby vlan 10
```

**Look for:** exactly one Active and one Standby per group; `P` for preempt on the intended Active; priority 105 on the intended Active.

Two Actives in the same group = the peers can't hear each other. Check the VLAN is allowed on the trunk, both SVIs are up, and both sides run `standby version 2`.

---

## IP / Interfaces

```
show ip interface brief | exclude unassigned
show ip interface vlan 10
show ip route
show ip route connected
show ip route ospf
show ip route static
show ip protocols
```

---

## OSPF

```
show ip ospf neighbor
show ip ospf interface brief
show ip ospf interface g1/1/1
show ip ospf database
show ip protocols
```

**Expected neighbor counts:** R1 = 2, CSW1 = 6, CSW2 = 6, each DSW = 3.

**Look for:** state `FULL`. With point-to-point network types the role column shows `-` rather than `DR`/`BDR` — that confirms the network type applied.

**No neighbor?** Check, in order: network type match, area match, subnet mask match, hello/dead timer match, passive-interface, MTU, ACLs.

---

## DHCP

```
show ip dhcp pool
show ip dhcp binding
show ip dhcp conflict
show ip dhcp server statistics
clear ip dhcp binding *
```

On the relay switch: `show running-config interface vlan 10 | include helper`

---

## NAT

```
show ip nat translations
show ip nat translations verbose
show ip nat statistics
clear ip nat translation *
```

**Look for:** the static entry for 10.5.0.4 ↔ 203.0.113.113 present even with zero traffic; dynamic entries showing port numbers (proving `overload`); inside and outside interface counts non-zero in `show ip nat statistics`.

---

## ACLs

```
show access-lists
show ip access-lists OfficeA_to_OfficeB
show ip interface vlan 10 | include access list
```

**Look for:** per-ACE match counters incrementing on the line you expect. If the wrong line is matching, your ACE order is wrong.

---

## Port Security

```
show port-security
show port-security interface f0/1
show port-security address
show running-config interface f0/1
```

**Look for:** violation mode `Restrict`, correct maximum (1 or 2), sticky MACs present in the running-config.

---

## DHCP Snooping & DAI

```
show ip dhcp snooping
show ip dhcp snooping binding
show ip dhcp snooping statistics
show ip arp inspection
show ip arp inspection interfaces
show ip arp inspection statistics
```

**Look for:** Option 82 insertion disabled; uplinks trusted, host ports untrusted; **all three** DAI validation checks (src-mac, dst-mac, ip) listed as enabled.

---

## Services

```
show ntp status
show ntp associations
show clock
show snmp community
show snmp
show logging
show hosts
show ip ssh
show ssh
show users
```

---

## Discovery protocols

```
show cdp                                 ! should report CDP not enabled
show lldp
show lldp neighbors
show lldp neighbors detail
show lldp interface f0/1
```

---

## IPv6

```
show ipv6 interface brief
show ipv6 interface g0/0
show ipv6 route
show ipv6 route static
show ipv6 neighbors
ping ipv6 <address>
```

---

## Files & system

```
show flash:
show version
show running-config
show startup-config
dir flash:
```

---

## Filtering tricks that save time

```
show running-config | section router ospf
show running-config | include ntp
show running-config | begin interface Vlan
show ip interface brief | exclude unassigned
show interfaces status | include connected
show running-config interface g1/1/1
```

- `section` — the whole config block
- `include` — matching lines only
- `exclude` — everything except matching lines
- `begin` — from the first match to the end

---

## End-to-end tests

From **PC1** (10.1.0.0/24):

```
ping 10.1.0.1                ! HSRP VIP — local gateway
ping 10.0.0.76               ! R1 loopback — OSPF working
ping 10.5.0.4                ! SRV1 — cross-office routing
ping 10.3.0.x                ! Office B PC — ACL permits ICMP
ping jeremysitlab.com        ! DNS + NAT + default route
ssh -l cisco 10.0.0.76       ! SSH permitted from this subnet
```

From **PC3** (10.3.0.0/24):

```
ping 10.0.0.76               ! should succeed
ssh -l cisco 10.0.0.76       ! should be REFUSED by ACL 1
```

If all of those behave as described, the whole lab is working.
