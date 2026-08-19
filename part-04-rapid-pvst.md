# Part 4 — Rapid Spanning Tree Protocol

Short part, big concept: make the Layer 2 forwarding path agree with the Layer 3 forwarding path.

---

## 4.1 — Enable Rapid PVST+

**All Access and Distribution switches** (not the core — it has no VLANs):

```
spanning-tree mode rapid-pvst
```

### STP modes on the exam

| Mode | Standard | Instances |
|---|---|---|
| PVST+ | Cisco, 802.1D-based | One per VLAN |
| **Rapid PVST+** | Cisco, 802.1w-based | One per VLAN |
| MST | 802.1s | One per group of VLANs |

Rapid PVST+ converges in a few seconds instead of ~50, because RSTP replaces the passive listening/learning timers with an active proposal/agreement handshake between neighbors.

Per-VLAN instances are what make this design possible at all — without them you couldn't have a different root bridge for VLAN 10 than for VLAN 20.

---

## 4.2 — Align root bridges with HSRP Active routers

Requirement: root bridge for each VLAN = HSRP Active router for that VLAN, using the **lowest possible priority**; the HSRP Standby router gets **one increment above** the lowest.

Bridge priority must be a multiple of **4096** (only the top 4 bits of the 16-bit priority field are configurable). So the lowest possible is `0` and one increment above is `4096`.

**DSW-A1** (Active for VLANs 10 and 99):

```
spanning-tree vlan 10,99 priority 0
spanning-tree vlan 20,40 priority 4096
```

**DSW-A2** (Active for VLANs 20 and 40):

```
spanning-tree vlan 20,40 priority 0
spanning-tree vlan 10,99 priority 4096
```

**DSW-B1** (Active for VLANs 10 and 99):

```
spanning-tree vlan 10,99 priority 0
spanning-tree vlan 20,30 priority 4096
```

**DSW-B2** (Active for VLANs 20 and 30):

```
spanning-tree vlan 20,30 priority 0
spanning-tree vlan 10,99 priority 4096
```

### Why alignment matters

If the STP root for VLAN 10 is DSW-A2 but the HSRP Active router for VLAN 10 is DSW-A1, then a frame from PC1 to its default gateway takes the STP path *up to DSW-A2*, gets bridged *across Po1 to DSW-A1*, and only then gets routed. Every packet takes an unnecessary hop across the inter-switch link. Nothing breaks — it's just quietly inefficient and it saturates Po1. Aligning them means traffic goes straight to the router that's actually forwarding it.

### The `priority` vs `root primary` distinction

You could use `spanning-tree vlan 10 root primary`, which tells IOS to *calculate* a winning priority (usually 24576, or 4096 below the current root). But it doesn't set 0, so it doesn't satisfy "lowest possible priority." Set the value explicitly.

### Bridge ID recap

A Bridge ID is `Priority + Extended System ID (VLAN) + MAC address`. With `priority 0` on VLAN 10, DSW-A1's actual advertised priority is `0 + 10 = 10`. That's why `show spanning-tree` displays "priority 10" and not "0" — the VLAN number is baked into the field. It's not a mistake.

**Verify:**

```
show spanning-tree vlan 10
```

Look for **`This bridge is the root`** on the intended switch. Also:

```
show spanning-tree summary
show spanning-tree root
```

`show spanning-tree root` gives a one-line-per-VLAN table — the fastest way to confirm all four VLANs at once.

On an access switch, `show spanning-tree vlan 10` should show one root port toward the root DSW and the other uplink in **Alternate/Blocking** state. That blocked port is spanning tree doing its job.

---

## 4.3 — PortFast and BPDU Guard on host ports

Requirement: on all ports connected to end hosts, **including WLC1**, configured in interface config mode.

**ASW-A1** (F0/1 → LWAP1, F0/2 → WLC1):

```
interface f0/1
 spanning-tree portfast
 spanning-tree bpduguard enable
!
interface f0/2
 spanning-tree portfast trunk
 spanning-tree bpduguard enable
```

**ASW-A2, ASW-A3, ASW-B1, ASW-B2, ASW-B3:**

```
interface f0/1
 spanning-tree portfast
 spanning-tree bpduguard enable
```

### `portfast` vs `portfast trunk`

This is the detail most people miss. F0/2 is a **trunk** (it carries VLANs 40 and 99 to the WLC), and plain `spanning-tree portfast` is rejected — or silently ineffective — on a trunk port. You need:

```
spanning-tree portfast trunk
```

The `trunk` keyword tells IOS "yes, I know this is a trunk, and yes, I really do mean it — there's an end device on the other end, not a switch."

### What these actually do

**PortFast** skips listening and learning, moving the port straight to forwarding. Without it, a PC waits ~30 seconds after link-up before it can send anything — long enough for a DHCP request to time out. That's the classic "PC gets an APIPA address on boot" symptom.

**BPDU Guard** is PortFast's safety net. A PortFast port that receives a BPDU is immediately put in `err-disabled`, because a BPDU means someone plugged a switch into a port that was declared host-only. Without it, a rogue switch could become root bridge and reshape the entire topology.

Recovering an err-disabled port:

```
interface f0/1
 shutdown
 no shutdown
```

Or configure automatic recovery:

```
errdisable recovery cause bpduguard
errdisable recovery interval 300
```

### Global vs interface config

There are global equivalents:

```
spanning-tree portfast default
spanning-tree portfast bpduguard default
```

The requirement explicitly says *interface config mode*, so use the per-interface commands. In production the global versions are usually preferred on access switches — set once, applies to every access port, no way to forget one.

**Verify:**

```
show spanning-tree interface f0/1 detail
show spanning-tree summary
show interfaces status err-disabled
```

`show spanning-tree summary` reports how many ports are in PortFast and BPDU Guard mode.

---

## Part 4 verification checklist

```
show spanning-tree summary          ! mode = rapid-pvst
show spanning-tree root             ! root per VLAN matches HSRP Active
show spanning-tree vlan 10          ! "This bridge is the root" on DSW-x1
show spanning-tree vlan 20          ! "This bridge is the root" on DSW-x2
show spanning-tree interface f0/1 detail
show interfaces status err-disabled ! should be empty
```

Cross-check against Part 3: for every VLAN, the switch showing `Active` in `show standby brief` should be the same switch showing `This bridge is the root` in `show spanning-tree`.

---

**Next:** [Part 5 — Static and Dynamic Routing →](part-05-routing.md)
