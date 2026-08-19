# Pitfalls & Gotchas

The things that actually cost me time, plus the ones that reliably cost everyone time. Ordered by how much pain they cause relative to how small the fix is.

---

## The big ones (imo)

### `ip routing` missing on a multilayer switch

**Symptom:** SVIs are up, addresses are right, OSPF forms no adjacencies, nothing routes between VLANs.

**Cause:** multilayer switches ship with IPv4 routing **disabled**.

**Fix:** `ip routing` on all six Core and Distribution switches. See [Part 3.2](part-03-ip-l3-etherchannel-hsrp.md#32--enable-ipv4-routing).

The IPv6 twin of this is `ipv6 unicast-routing`, which is needed on **routers too** — and it's easy to configure on CSW1 and forget on CSW2.

---

### `channel-group` before `no switchport` on the L3 EtherChannel

**Symptom:** `ip address` on Port-channel1 is rejected.

**Cause:** the port-channel inherits Layer 2 or Layer 3 from its first member. Bundle switchports and you get a Layer 2 channel.

**Fix:** delete and rebuild in the right order.

```
interface po1
 no interface port-channel 1
!
interface range g1/0/2-3
 no switchport
 channel-group 1 mode desirable
```

Confirm with `show etherchannel summary` — you want `RU`, not `SU`.

---

### `switchport trunk allowed vlan` overwrites the list

**Symptom:** you add a VLAN to a trunk and three others vanish.

**Cause:** the bare command **replaces** the entire allowed list.

**Fix:** use `add` / `remove`:

```
switchport trunk allowed vlan add 50
switchport trunk allowed vlan remove 20
```

---

### DAI validation checks overwrite each other

**Symptom:** `show ip arp inspection` lists only one validation check enabled.

**Cause:** each `ip arp inspection validate <x>` command **replaces** the previous one.

**Fix:** one command, all three keywords:

```
ip arp inspection validate src-mac dst-mac ip
```

Always confirm with `show ip arp inspection`. See [Part 7.4](part-07-security.md#the-single-most-important-gotcha-here).

---

### DHCP breaks the moment you enable snooping

Two independent causes, both in [Part 7.3](part-07-security.md#option-82):

1. **Uplinks not trusted.** Every port is untrusted by default once snooping is on, so the switch drops the DHCP OFFER coming down from the server.
2. **Option 82 still being inserted.** Cisco IOS DHCP servers discard relayed packets carrying option 82 with `giaddr` 0.0.0.0. Fix with `no ip dhcp snooping information option`.

---

### PortFast rejected on the WLC trunk

**Symptom:** `spanning-tree portfast` doesn't apply to ASW-A1 F0/2.

**Cause:** F0/2 is a trunk. Plain `portfast` is for access ports.

**Fix:** `spanning-tree portfast trunk`.

---

### OSPF neighbor won't come up

Check in this order — it's almost always the first one in this lab:

1. **Network type mismatch** — `ip ospf network point-to-point` must be on **both** ends
2. Area mismatch
3. Subnet / mask mismatch on the link
4. Hello / dead timer mismatch
5. `passive-interface` accidentally covering the link
6. MTU mismatch (sticks in `EXSTART`/`EXCHANGE`)
7. An ACL blocking OSPF (protocol 89) or the interface being down

`show ip ospf interface <int>` shows the network type, timers, and area all in one place.

---

## The small ones that waste an hour

### `deafult-router`

A typo in the source notes I started from, in the A-Mgmt DHCP pool. IOS rejects it, but the error scrolls past in a large paste and the pool silently hands out no gateway. If one pool's clients have no default gateway, check the spelling first.

### `enable algorithm-type scrypt secret <pw>`

The word order is not what you'd guess. It is **not** `enable secret algorithm-type scrypt <pw>`.

### `exec-timeout 30` means 30 minutes

Syntax is `exec-timeout <minutes> <seconds>`. `exec-timeout 0 0` disables the timeout entirely.

### `access-class`, not `ip access-group`, on VTY lines

`ip access-group` is for physical interfaces. VTY lines use `access-class <n> in`. Getting this wrong means your SSH restriction does nothing.

### `ip domain name` must exist before `crypto key generate rsa`

The RSA key is labelled `hostname.domain`. Without a domain name, key generation fails. [Part 6.4](part-06-network-services.md#64--domain-name-and-dns-client-on-all-devices) has to happen before [6.10](part-06-network-services.md#610--ssh).

### `write memory` before `reload` during the IOS upgrade

Otherwise the `boot system` statement is lost and you boot straight back into the old image.

Also check for a pre-existing `boot system` line — IOS tries them **in order**, so a stale first entry wins.

### Sticky MACs don't survive a reload without `write memory`

Sticky writes to running-config, not startup-config.

### `www.jeremysitlab.com` is a CNAME, not an A record

Selecting the wrong record type in SRV1's DNS panel is easy and the failure is silent.

### Root bridge priority displays as VLAN-ID-plus-priority

`priority 0` on VLAN 10 shows as `10` in `show spanning-tree`. That's the extended system ID, not a mistake.

### HSRP version mismatch

`standby version 2` must be set on **both** peers, ideally before the rest of the group config. Mismatched versions produce two Active routers and no useful error message.

### Port security needs a statically configured port mode

A port left in dynamic mode rejects `switchport port-security`. Part 2.7's `switchport mode access` is a prerequisite, not decoration.

---

## Packet Tracer–specific behavior

| Thing | PT behavior                                                                | Real IOS |
|---|---|---|
| `default-information originate always` | **Doesn't exist** — must remove/re-add the plain command after a link flip | Works; handles failover automatically |
| `ip ospf network point-to-point` on an L3 Port-channel | Breaks the adjacency — leave Po1 default                                   | Supported |
| DHCP option 43 | Friendly `option 43 ip 10.0.0.7`                                           | Hex TLV: `option 43 hex f1040a000007` |
| NTP sync | Extremely slow; may never show synced                                      | Minutes |
| FTP IOS copy | 4–5 minutes, looks frozen                                                  | Fast |
| Wireless client DHCP | **Never works** — documented limitation                                    | Works |
| `ntp authenticate` | Often accepted and ignored                                                 | Required |
| DAI with static hosts | Loose enough that SRV1 keeps working                                       | Needs an ARP ACL |

---

## Ordering dependencies

These aren't optional sequencing preferences — get them wrong and you'll be undoing work:

```
no switchport            →  channel-group           (L3 EtherChannel)
EtherChannel             →  trunk config on Po1
VTP domain + mode        →  create VLANs
VLANs exist              →  SVIs come up
switchport mode access   →  port security
PortFast                 →  BPDU Guard
ip domain name           →  crypto key generate rsa
DHCP snooping            →  DAI
ip helper-address        →  clients can lease
standby version 2        →  rest of HSRP group config
DHCP pools + option 43   →  LWAPs join the WLC
```

---

## General troubleshooting method

When something doesn't work, resist the urge to re-paste config. Work bottom-up:

1. **Physical** — `show interfaces status`. Is it up/up? Did you shut it in Part 2.9 by mistake?
2. **Data link** — `show interfaces trunk`, `show vlan brief`, `show etherchannel summary`. Is the VLAN allowed? Is the trunk formed?
3. **Network** — `show ip interface brief`, `show ip route`. Is there an address? Is there a route?
4. **Protocol** — `show ip ospf neighbor`, `show standby brief`. Are the adjacencies up?
5. **Policy** — `show access-lists`, `show port-security`, `show ip dhcp snooping`. Is something you configured deliberately blocking it?

Step 5 is the one people skip, and after Part 7 it's frequently the answer. `show access-lists` match counters tell you immediately whether an ACL is the culprit.

---

**Back to:** [README](../README.md)
