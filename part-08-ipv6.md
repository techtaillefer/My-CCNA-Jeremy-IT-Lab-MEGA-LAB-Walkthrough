# Part 8 — IPv6

A dual-stack pilot across the core: R1, CSW1, and CSW2 only. The rest of the network stays IPv4-only, which is exactly how real migrations start.

---

## 8.1 — Enable IPv6 routing and address the interfaces

### R1

```
ipv6 unicast-routing
!
interface g0/0/0
 ipv6 address 2001:db8:a::2/64
 no shutdown
!
interface g0/1/0
 ipv6 address 2001:db8:b::2/64
 no shutdown
!
interface g0/0
 ipv6 address 2001:db8:a1::/64 eui-64
!
interface g0/1
 ipv6 address 2001:db8:a2::/64 eui-64
```

### CSW1

```
ipv6 unicast-routing
!
interface g1/0/1
 ipv6 address 2001:db8:a1::/64 eui-64
!
interface po1
 ipv6 enable
```

### CSW2

```
ipv6 unicast-routing
!
interface g1/0/1
 ipv6 address 2001:db8:a2::/64 eui-64
!
interface po1
 ipv6 enable
```

> **Don't skip `ipv6 unicast-routing` on CSW2.** It's easy to configure CSW1 properly, then paste only the interface commands onto CSW2 because the addressing looks so similar. Without it, CSW2 will hold IPv6 addresses and respond to pings but won't forward a single IPv6 packet.

---

## Concepts

### `ipv6 unicast-routing`

IPv6 routing is **disabled by default on every device**, including routers. Without it, a device is an IPv6 *host*: it has addresses, it answers pings, and it drops everything it's asked to forward. It also stops the device sending Router Advertisements, so nothing downstream can SLAAC.

This is the IPv6 equivalent of `ip routing` in Part 3.2 — with the important difference that it's needed on **routers** too, not just multilayer switches.

### EUI-64

```
ipv6 address 2001:db8:a1::/64 eui-64
```

EUI-64 builds the 64-bit interface ID automatically from the interface's 48-bit MAC address:

1. Split the MAC in half: `AABB.CC` | `DD.EEFF`
2. Insert `FFFE` in the middle: `AABB.CCFF.FEDD.EEFF`
3. **Flip the 7th bit** of the first byte (the Universal/Local bit)

That bit flip is the part everyone forgets. Worked example:

```
MAC:         00 1A 2B 3C 4D 5E
Insert FFFE: 001A:2BFF:FE3C:4D5E
Flip bit 7:  00 = 0000 0000 → 0000 0010 = 02
Result:      021A:2BFF:FE3C:4D5E

Full address: 2001:DB8:A1:0:21A:2BFF:FE3C:4D5E
```

Note that you type the **prefix with `::`** and let the router fill in the host portion. You do not type a host part.

Verify what the router actually generated:

```
show ipv6 interface g0/0
```

Look for `FF:FE` in the middle of the address — that's EUI-64's signature. The privacy downside of this scheme (your MAC address is broadcast in every packet's source address) is precisely why RFC 4941 privacy extensions exist.

### `ipv6 enable`

```
interface po1
 ipv6 enable
```

The requirement is "enable IPv6 **without using the `ipv6 address` command**." `ipv6 enable` turns on IPv6 processing on the interface and generates a **link-local address** (`FE80::/10`, using EUI-64 from the MAC) — but no global unicast address.

That's genuinely useful: link-local addressing is all that's needed for routing protocol adjacencies (OSPFv3, EIGRPv6, and all IPv6 next-hops use link-local), so a transit link between two routers often doesn't need a global address at all. Fewer addresses to manage, smaller attack surface.

**Every** IPv6-enabled interface gets a link-local address automatically, including the ones with global addresses. `show ipv6 interface brief` shows both.

### Address types recap

| Type | Prefix | Notes |
|---|---|---|
| Global unicast | 2000::/3 | Internet-routable |
| Unique local | FC00::/7 | The IPv6 "private" range |
| Link-local | FE80::/10 | Auto-generated, not routable off-link |
| Multicast | FF00::/8 | Replaces broadcast entirely |
| **Documentation** | **2001:DB8::/32** | Reserved for docs/labs — which is why every address here starts with it |

---

## 8.2 — IPv6 static default routes

**R1:**

```
ipv6 route ::/0 2001:db8:a::1
ipv6 route ::/0 g0/1/0 2001:db8:b::1 2
```

### Route types

**Recursive** (first line): next-hop address only. The router does a second lookup to determine the exit interface.

**Fully specified** (second line): exit interface **and** next-hop address. Note the argument order — `ipv6 route <prefix> <interface> <next-hop> [AD]`.

Fully specified routes matter more in IPv6 than IPv4 because IPv6 next-hops are often **link-local** addresses, and a link-local address is only meaningful in the context of a specific interface (`FE80::1` could exist on every link simultaneously). If you use a link-local next-hop, the interface is **mandatory**.

### Floating route

The trailing `2` is the administrative distance, one above the IPv6 static default of 1. Same mechanism as the IPv4 floating static in Part 5 — the backup sits out of the routing table until the primary is withdrawn.

**`::/0`** is the IPv6 default route, equivalent to `0.0.0.0/0`.

**Verify:**

```
show ipv6 route
show ipv6 route static
```

Only the AD 1 route should be installed.

---

## Part 8 verification checklist

```
show ipv6 interface brief
show ipv6 interface g0/0
show ipv6 route
show ipv6 route static
show running-config | include ipv6 unicast-routing
```

Connectivity tests:

```
R1#ping ipv6 2001:db8:a1::<CSW1's EUI-64 suffix>
CSW1#show ipv6 neighbors
```

`show ipv6 neighbors` is the IPv6 equivalent of `show arp` — it displays the Neighbor Discovery cache. NDP replaces ARP entirely in IPv6, using ICMPv6 Neighbor Solicitation and Neighbor Advertisement messages over multicast rather than broadcast.

Easiest way to grab an EUI-64 address for testing: run `show ipv6 interface brief` on the far end and copy it, rather than calculating it by hand.

---

**Next:** [Part 9 — Wireless →](part-09-wireless.md)
