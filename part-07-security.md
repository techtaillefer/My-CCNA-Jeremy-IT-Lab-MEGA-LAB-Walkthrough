# Part 7 — Security: ACLs and Layer 2 Hardening

Filtering at Layer 3, then three Layer 2 features that defend the access edge.

---

## 7.1 — Extended ACL: OfficeA_to_OfficeB

Requirements:
- Allow ICMP from Office A PCs → Office B PCs
- Block **all other** traffic from Office A PCs → Office B PCs
- Allow everything else
- Apply per general best practice for extended ACLs

**DSW-A1 and DSW-A2:**

```
ip access-list extended OfficeA_to_OfficeB
 permit icmp 10.1.0.0 0.0.0.255 10.3.0.0 0.0.0.255
 deny ip 10.1.0.0 0.0.0.255 10.3.0.0 0.0.0.255
 permit ip any any
!
interface vlan 10
 ip access-group OfficeA_to_OfficeB in
```

### Why this ACE order is the only order that works

ACLs are evaluated **top-down, first match wins**, then stop.

1. `permit icmp 10.1.0.0/24 → 10.3.0.0/24` — pings between offices are allowed.
2. `deny ip 10.1.0.0/24 → 10.3.0.0/24` — everything else between those two subnets is dropped. `ip` covers all protocols, so this catches TCP, UDP, and anything else.
3. `permit ip any any` — required. Without it the implicit `deny any any` at the end of every ACL would black-hole *all* traffic from VLAN 10, including its Internet access.

Swap lines 1 and 2 and ICMP gets denied by the broader rule before it ever reaches the permit. That's the whole lesson: **specific rules first, general rules last.**

### Why it's applied where it is

The general best practice for **extended** ACLs is to place them **as close to the source as possible** — drop unwanted traffic before it consumes bandwidth crossing the network. (Standard ACLs match only on source address, so they go close to the *destination* to avoid over-blocking; that contrast is a reliable exam question.)

Here the closest Layer 3 point to Office A's PCs is the VLAN 10 SVI on the distribution switches — the PCs' default gateway. Applied **inbound**, so traffic is filtered as it arrives from the hosts.

**Apply it on both DSW-A1 and DSW-A2.** If you only do the HSRP Active router, the filter silently disappears the moment failover happens. Security controls that vanish during a failure aren't security controls.

**Verify:**

```
show access-lists
show ip access-lists OfficeA_to_OfficeB
show ip interface vlan 10 | include access list
```

Test from PC1 (10.1.0.x):

```
C:\> ping 10.3.0.x        ! should succeed
```

Then try any non-ICMP service to the same host — HTTP to a web server in 10.3.0.0/24, for example — and it should fail. `show access-lists` shows per-line match counters, which is the quickest way to confirm which ACE is actually being hit.

---

## 7.2 — Port Security

Requirements:
- Minimum necessary number of MACs per port
- Violation mode that blocks bad traffic without killing the port, and sends notifications
- Learned MACs saved to running-config automatically

### Working out "minimum necessary"

| Switch | F0/1 connects to | MACs needed |
|---|---|---|
| ASW-A1 | LWAP1 | **1** |
| ASW-A2 | PC1 + Phone1 | **2** |
| ASW-A3 | PC2 + Phone2 | **2** |
| ASW-B1 | LWAP2 | **1** |
| ASW-B2 | PC3 + Phone3 | **2** |
| ASW-B3 | SRV1 (no virtualisation) | **1** |

The default maximum is **1**, so the single-MAC ports need no `maximum` command at all.

**ASW-A1, ASW-B1, ASW-B3:**

```
interface f0/1
 switchport port-security
 switchport port-security violation restrict
 switchport port-security mac-address sticky
```

**ASW-A2, ASW-A3, ASW-B2:**

```
interface f0/1
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky
```

### Violation modes

| Mode | Drops bad frames | Port stays up | Logs / SNMP trap | Increments counter |
|---|---|---|---|---|
| **Protect** | Yes | Yes | **No** | No |
| **Restrict** | Yes | Yes | **Yes** | Yes |
| Shutdown (default) | Yes | **No** — err-disabled | Yes | Yes |

The requirement — block invalid traffic, don't affect valid traffic, send notifications — matches **restrict** exactly. Protect fails the notification requirement; shutdown fails the "don't affect valid traffic" requirement, since err-disabling the port kills the legitimate host too.

### Sticky MAC

```
switchport port-security mac-address sticky
```

The switch dynamically learns MACs and writes them into the running-config as static secure entries. Benefit: you don't have to type MAC addresses by hand, and the addresses survive a link flap.

> They only survive a **reload** if you `write memory` after they're learned. Sticky writes to running-config, not startup-config — a distinction worth remembering.

### Port security prerequisite

Port security only works on ports that are **statically** access or trunk. A port left in dynamic (DTP-negotiating) mode rejects the command. Part 2.7 already set `switchport mode access` on every F0/1, so this works — that ordering wasn't accidental.

**Verify:**

```
show port-security
show port-security interface f0/1
show port-security address
show running-config interface f0/1
```

The last one is where you'll see the sticky MACs written in.

---

## 7.3 — DHCP Snooping

Requirements:
- Enable for all active VLANs in each LAN
- Trust the appropriate ports
- Disable Option 82 insertion
- 15 pps rate limit on active untrusted ports
- 100 pps on ASW-A1's WLC connection

**ASW-A1:**

```
ip dhcp snooping
ip dhcp snooping vlan 10,20,40,99
no ip dhcp snooping information option
!
interface range g0/1-2
 ip dhcp snooping trust
!
interface f0/1
 ip dhcp snooping limit rate 15
!
interface f0/2
 ip dhcp snooping limit rate 100
```

**ASW-A2, ASW-A3:**

```
ip dhcp snooping
ip dhcp snooping vlan 10,20,40,99
no ip dhcp snooping information option
!
interface range g0/1-2
 ip dhcp snooping trust
!
interface f0/1
 ip dhcp snooping limit rate 15
```

**ASW-B1, ASW-B2, ASW-B3:**

```
ip dhcp snooping
ip dhcp snooping vlan 10,20,30,99
no ip dhcp snooping information option
!
interface range g0/1-2
 ip dhcp snooping trust
!
interface f0/1
 ip dhcp snooping limit rate 15
```

### What it defends against

A rogue DHCP server on the access edge can hand clients a bogus default gateway and become a man-in-the-middle for everything they send. DHCP Snooping stops this by classifying every port as trusted or untrusted:

- **Trusted** ports may forward DHCP **server** messages (OFFER, ACK, NAK).
- **Untrusted** ports may only send DHCP **client** messages (DISCOVER, REQUEST). A server message arriving on an untrusted port is dropped and the port may be err-disabled.

**All ports are untrusted by default** once snooping is enabled — which is why forgetting to trust the uplinks breaks DHCP for the entire switch. The uplinks G0/1 and G0/2 face the distribution switches, which is the direction the real DHCP server lives, so they get trusted.

**Note F0/2 on ASW-A1 stays untrusted.** WLC1 is not a DHCP server and shouldn't be sending server messages; it just gets a higher rate limit (100 pps) because it relays DHCP for wireless clients and generates more traffic than a single PC.

### The binding table

DHCP Snooping builds a **binding table** — MAC, IP, lease time, VLAN, and interface for every client that leased through the switch. This table is the foundation that **DAI** (7.4) and IP Source Guard depend on. It's the real reason snooping has to be configured before DAI.

```
show ip dhcp snooping binding
```

### Option 82

```
no ip dhcp snooping information option
```

Option 82 is the DHCP relay agent information field. By default, a snooping switch inserts it — but Cisco IOS DHCP servers **discard relayed packets with option 82 when giaddr is 0.0.0.0**, which is exactly the case for a Layer 2 switch inserting it. Result: DHCP silently breaks. Disabling insertion is standard practice on Layer 2 access switches and here it's a hard requirement.

If DHCP stops working immediately after you enable snooping, this is the first thing to check.

### Rate limiting

```
ip dhcp snooping limit rate 15
```

Caps DHCP packets at 15 per second on that port. A legitimate client sends a handful of DHCP packets on boot and then nothing for hours; anything sustaining 15 pps is trying to exhaust the DHCP pool. Exceeding the limit err-disables the port.

Only set rate limits on **untrusted** ports. Putting a low limit on a trusted uplink carrying DHCP for the whole switch will err-disable it under normal load.

**Verify:**

```
show ip dhcp snooping
show ip dhcp snooping binding
show ip dhcp snooping statistics
```

---

## 7.4 — Dynamic ARP Inspection

Requirements:
- Enable for all active VLANs in each LAN
- Trust the appropriate ports
- Enable **all** optional validation checks

**ASW-A1, ASW-A2, ASW-A3:**

```
ip arp inspection vlan 10,20,40,99
ip arp inspection validate src-mac dst-mac ip
!
interface range g0/1-2
 ip arp inspection trust
```

**ASW-B1, ASW-B2, ASW-B3:**

```
ip arp inspection vlan 10,20,30,99
ip arp inspection validate src-mac dst-mac ip
!
interface range g0/1-2
 ip arp inspection trust
```

### What it defends against

ARP has no authentication whatsoever. Any host can send a gratuitous ARP claiming to own the default gateway's IP, and every device on the segment will believe it — **ARP poisoning**, the standard on-path attack.

DAI intercepts every ARP message on untrusted ports and checks it against the **DHCP Snooping binding table**. If the sender's MAC/IP pairing doesn't match a legitimate lease, the ARP is dropped.

Same trust model as snooping: uplinks trusted, host ports untrusted.

### The single most important gotcha here

```
ip arp inspection validate src-mac dst-mac ip
```

**This must be one command with all three keywords.** Entering them separately:

```
ip arp inspection validate src-mac
ip arp inspection validate dst-mac      ← this REPLACES the previous line
ip arp inspection validate ip           ← and this replaces that one
```

…leaves you with only `ip` validation enabled. The command overwrites rather than appends. Always verify with `show ip arp inspection` that all three appear.

What each checks:

| Check | Compares |
|---|---|
| `src-mac` | Ethernet header source MAC vs ARP body sender MAC |
| `dst-mac` | Ethernet header destination MAC vs ARP body target MAC |
| `ip` | ARP body for invalid/unexpected IPs (0.0.0.0, 255.255.255.255, multicast) |

### Statically addressed hosts

DAI relies on the DHCP binding table — so a host with a **static** IP has no binding entry, and its ARP gets dropped.

**SRV1 is statically addressed** (10.5.0.4, configured in Part 3.10) and sits on ASW-B3's untrusted F0/1. In production you'd handle this with an ARP ACL:

```
arp access-list STATIC-HOSTS
 permit ip host 10.5.0.4 mac host <SRV1-MAC>
!
ip arp inspection filter STATIC-HOSTS vlan 30
```

The lab doesn't require it, and Packet Tracer's DAI implementation is loose enough that SRV1 keeps working — but this is the kind of thing that causes a real outage on a real network, so it's worth knowing the fix exists.

**Verify:**

```
show ip arp inspection
show ip arp inspection vlan 10
show ip arp inspection interfaces
show ip arp inspection statistics
```

Confirm all three validation checks are listed as enabled.

---

## Layer 2 security summary

| Feature | Attack it stops | Depends on |
|---|---|---|
| **Port Security** | MAC flooding / CAM table overflow, unauthorised devices | Static access/trunk mode |
| **DHCP Snooping** | Rogue DHCP server, pool exhaustion | — |
| **DAI** | ARP poisoning / on-path attacks | DHCP Snooping binding table |
| **BPDU Guard** (Part 4) | Rogue switch becoming root bridge | PortFast |

Order of configuration is not arbitrary: Port Security needs static port mode from Part 2, DAI needs snooping from 7.3, BPDU Guard needs PortFast from Part 4.

---

## Part 7 verification checklist

```
show access-lists
show ip interface vlan 10 | include access list
show port-security
show port-security interface f0/1
show ip dhcp snooping
show ip dhcp snooping binding
show ip arp inspection
show ip arp inspection interfaces
show interfaces status err-disabled      ! should be empty
```

---

**Next:** [Part 8 — IPv6 →](part-08-ipv6.md)
