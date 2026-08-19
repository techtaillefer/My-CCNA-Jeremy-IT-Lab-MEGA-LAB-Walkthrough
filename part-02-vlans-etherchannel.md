# Part 2 — VLANs, Trunking, VTP, Layer 2 EtherChannel

This is where the switched infrastructure comes together. **Order matters here more than anywhere else in the lab** — build EtherChannels before trunks, and VTP before you create VLANs.

---

## 2.1 — Office A: PAgP EtherChannel (DSW-A1 ↔ DSW-A2)

Requirement: Cisco-proprietary protocol, both sides actively negotiating.

**DSW-A1 and DSW-A2:**

```
interface range g1/0/4-5
 channel-group 1 mode desirable
```

`channel-group 1` automatically creates `Port-channel1` — you never create the Po interface manually.

### Choosing the mode

| Protocol | Active mode | Passive mode | Static |
|---|---|---|---|
| **PAgP** (Cisco) | `desirable` | `auto` | `on` |
| **LACP** (802.3ad, open standard) | `active` | `passive` | `on` |

"Both switches should actively try to form an EtherChannel" → both sides `desirable`. (`auto`/`auto` never forms a channel — neither side ever speaks first.)

`mode on` forms a channel with no negotiation protocol at all. It works, but it will happily bundle mismatched ports and black-hole traffic, so it's the wrong answer whenever a protocol is specified.

---

## 2.2 — Office B: LACP EtherChannel (DSW-B1 ↔ DSW-B2)

Requirement: open standard, both sides active.

**DSW-B1 and DSW-B2:**

```
interface range g1/0/4-5
 channel-group 1 mode active
```

---

## 2.3 — Trunking

All Access ↔ Distribution links, plus both EtherChannels.

Requirements:
- DTP explicitly disabled
- Native VLAN 1000 (unused)
- Office A allows VLANs 10, 20, 40, 99
- Office B allows VLANs 10, 20, 30, 99

**DSW-A1 and DSW-A2:**

```
interface range g1/0/1-3
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 1000
 switchport trunk allowed vlan 10,20,40,99
!
interface po1
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 1000
 switchport trunk allowed vlan 10,20,40,99
```

**ASW-A1, ASW-A2, ASW-A3:**

```
interface range g0/1-2
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 1000
 switchport trunk allowed vlan 10,20,40,99
```

**DSW-B1 and DSW-B2:**

```
interface range g1/0/1-3
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 1000
 switchport trunk allowed vlan 10,20,30,99
!
interface po1
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 1000
 switchport trunk allowed vlan 10,20,30,99
```

**ASW-B1, ASW-B2, ASW-B3:**

```
interface range g0/1-2
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 1000
 switchport trunk allowed vlan 10,20,30,99
```

### Things worth understanding here

**Configure the Po interface, not the members.** Applying trunk settings to `Port-channel1` pushes them down to G1/0/4 and G1/0/5 automatically. If you configure the physical members individually and they end up mismatched, the channel goes down — check with `show etherchannel summary` for an `(s)` (suspended) flag.

**`switchport nonegotiate` is the DTP kill switch.** `switchport mode trunk` alone still sends DTP frames; it just doesn't need them. The requirement says *explicitly* disable, so `nonegotiate` is mandatory. Note it's invalid on a port in dynamic mode — the port must already be hard-set to trunk or access.

**Native VLAN 1000 is a security move.** Untagged frames on the trunk land in an unused VLAN that goes nowhere, which defeats double-tagging (VLAN hopping) attacks. Setting the same native VLAN on both ends is essential — mismatches generate CDP/STP errors, and here CDP gets disabled in Part 6 anyway, so the errors would go silent.

**`switchport trunk allowed vlan` replaces the list; it doesn't add to it.** Use `switchport trunk allowed vlan add 50` to append. Typing the bare command twice means the second one wins and the first list is gone.

> **Older platform note:** on 3560-class switches you must run `switchport trunk encapsulation dot1q` *before* `switchport mode trunk`, because those switches support ISL too. The 3650s used here are dot1q-only and reject the command. If you see `Command rejected: An interface whose trunk encapsulation is "Auto" cannot be configured to "trunk" mode`, that's what you're hitting.

**Verify:**

```
show interfaces trunk
show etherchannel summary
show interfaces g1/0/1 switchport
```

In `show etherchannel summary`, you want `Po1(SU)` and members flagged `(P)` — bundled in the port-channel. `(I)` means independent (not bundled), `(D)` means down.

---

## 2.4 — VTPv2

Requirement: one distribution switch per office is a VTPv2 server on domain `JeremysITLab`; all access switches are clients.

**DSW-A1 and DSW-B1** (the servers):

```
vtp version 2
vtp domain JeremysITLab
```

`vtp mode server` is the default, so it doesn't need typing — but typing it doesn't hurt and makes the config self-documenting.

**All six Access switches:**

```
vtp mode client
```

### Why this works without configuring the domain on the clients

A switch with a **null VTP domain** adopts the first domain name it hears in a VTP advertisement. Since the access switches are trunked to DSW-A1/DSW-B1, they learn `JeremysITLab` automatically. That's also exactly why VTP is dangerous in production — a switch joins a domain just by being plugged in.

**DSW-A2 and DSW-B2** are left as servers with a null domain; they'll learn the domain over the Po1 trunk too. That's fine and intentional — it means either distribution switch can create VLANs.

### The classic VTP trap

VTP updates are accepted based on the **configuration revision number**, not on device role. A client with a *higher* revision number than the server will overwrite the server's VLAN database. Wiping VLANs off a whole campus by plugging in an old lab switch is a real outage that has happened to real people.

To reset a switch's revision number to 0: change it to transparent mode and back, or change the domain name to something else and back.

**Verify:**

```
show vtp status
```

Check: Domain Name, VTP Version 2 running, Operating Mode, and Configuration Revision. All switches in an office should show the same revision number once they've synced.

---

## 2.5 / 2.6 — Create the VLANs

Create these on the VTP **server** only; VTP propagates them.

**DSW-A1 (Office A):**

```
vlan 10
 name PCs
vlan 20
 name Phones
vlan 40
 name Wi-Fi
vlan 99
 name Management
```

**DSW-B1 (Office B):**

```
vlan 10
 name PCs
vlan 20
 name Phones
vlan 30
 name Servers
vlan 99
 name Management
```

VLAN names are case-sensitive and there's no partial credit — `Wi-Fi` is not `WiFi`.

> If a switch rejects `vlan 10` with `VTP VLAN configuration not allowed when device is in CLIENT mode`, you're on the wrong switch. That's VTP working correctly.

**Verify — on a client switch, not the server:**

```
show vlan brief
```

Seeing all four VLANs appear on ASW-A3 without ever typing them there is the actual proof VTP is doing its job.

---

## 2.7 — Access ports

Requirements:
- LWAPs don't use FlexConnect → they sit in the **Management VLAN (99)**, untagged, as ordinary access ports
- PCs in VLAN 10, phones in VLAN 20 (voice VLAN)
- SRV1 in VLAN 30
- Manually set access mode, explicitly disable DTP

**ASW-A1 and ASW-B1** (LWAP1 / LWAP2):

```
interface f0/1
 switchport mode access
 switchport nonegotiate
 switchport access vlan 99
```

**ASW-A2, ASW-A3, ASW-B2** (PC + IP phone):

```
interface f0/1
 switchport mode access
 switchport nonegotiate
 switchport access vlan 10
 switchport voice vlan 20
```

**ASW-B3** (SRV1):

```
interface f0/1
 switchport mode access
 switchport nonegotiate
 switchport access vlan 30
```

### Why the LWAPs go in VLAN 99

In **local mode** (the default, no FlexConnect), an LWAP tunnels *all* client traffic back to the WLC inside CAPWAP. The AP itself only ever needs to reach the controller — so its switchport is a plain access port in the management VLAN. Client VLAN 40 traffic never appears on that link untagged; it's encapsulated.

With FlexConnect, the AP switches client traffic locally and the port would need to be a trunk carrying the client VLANs. That's the distinction the requirement is testing.

### Voice VLAN

`switchport voice vlan 20` makes the port a hybrid: VLAN 10 untagged for the PC daisy-chained through the phone, VLAN 20 tagged for the phone itself. The switch uses CDP/LLDP to tell the phone which VLAN to tag with.

> **Heads-up for Part 6:** you'll disable CDP globally later. LLDP-MED takes over the job, which is why Part 6 has you enable `lldp run` rather than just turning CDP off.

---

## 2.8 — ASW-A1 → WLC1

Requirements: carry Wi-Fi (40) and Management (99); Management untagged; DTP off.

```
interface f0/2
 switchport mode trunk
 switchport nonegotiate
 switchport trunk allowed vlan 40,99
 switchport trunk native vlan 99
```

"Management untagged" = make VLAN 99 the **native VLAN** on this trunk. This is the one place in the lab where the native VLAN is deliberately a real, in-use VLAN rather than the black-hole VLAN 1000 — the WLC expects its management interface untagged.

---

## 2.9 — Shut unused ports

```
ASW-A1(config)#interface range f0/3-24
ASW-A1(config-if-range)#shutdown
```

Adjust the range per device. On the access switches, F0/1 (and F0/2 on ASW-A1) plus G0/1-2 are in use; everything else gets shut. On the distribution switches, G1/0/1-5 and G1/1/1-2 are in use.

Do this **last** in Part 2 — shutting ports before the trunks are up makes troubleshooting the trunks confusing.

**Verify:**

```
show ip interface brief | include down
show interfaces status
```

Unused ports should read `administratively down`, not just `down`.

---

## Part 2 verification checklist

```
show etherchannel summary          ! Po1 = SU, members = P
show interfaces trunk              ! correct native + allowed VLANs
show vtp status                    ! domain, v2, mode, revision
show vlan brief                    ! VLANs present on clients via VTP
show interfaces status             ! access ports in right VLANs
show interfaces f0/1 switchport    ! mode access, negotiation off
```

---

**Next:** [Part 3 — IP Addressing, Layer 3 EtherChannel, HSRPv2 →](part-03-ip-l3-etherchannel-hsrp.md)
