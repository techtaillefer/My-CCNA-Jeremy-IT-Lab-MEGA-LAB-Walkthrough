# Part 9 — Wireless

Entirely GUI-driven — the WLC has no CLI in Packet Tracer. Everything the CLI parts of this lab built (VLAN 40, the DHCP pool, option 43, the trunk to ASW-A1) now gets used.

---

## Prerequisites — check these before you start

Almost every "the WLC page won't load" problem traces back to one of these:

| Requirement | Configured in | Verify with |
|---|---|---|
| VLAN 40 exists and is allowed on trunks | Part 2.3 / 2.5 | `show interfaces trunk` |
| ASW-A1 F0/2 is a trunk, native VLAN 99, allowing 40 & 99 | Part 2.8 | `show interfaces f0/2 switchport` |
| HSRP VIP 10.6.0.1 is up for VLAN 40 | Part 3.15 | `show standby brief` |
| Wi-Fi DHCP pool exists on R1 | Part 6.1 | `show ip dhcp pool` |
| Option 43 → 10.0.0.7 in both mgmt pools | Part 6.1 | `show run \| section dhcp` |
| PortFast trunk + BPDU Guard on F0/2 | Part 4.3 | `show spanning-tree interface f0/2 detail` |

**WLC1 itself** needs a static management address of **10.0.0.7 /28** with gateway **10.0.0.1**, on the untagged (native) VLAN 99 — that's why Part 2.8 set the native VLAN to 99 rather than the usual 1000.

---

## 9.1 — Reach the GUI

From **PC1** (or any PC in a subnet that can route to 10.0.0.0/28) → **Desktop → Web Browser**:

```
https://10.0.0.7
```

- Username: `admin`
- Password: `adminPW12`

**Must be `https://`, not `http://`.** The WLC only listens on HTTPS by default.

If the page doesn't load, go back through the prerequisites table above. Ping 10.0.0.7 from PC1 first to separate a routing problem from a web-service problem.

---

## 9.2 — Dynamic interface for the Wi-Fi WLAN

**CONTROLLER → Interfaces → New**

| Field | Value |
|---|---|
| Interface Name | `Wi-Fi` |
| VLAN Id | `40` |
| Port Number | `1` |
| IP Address | `10.6.0.2` |
| Netmask | `255.255.255.0` |
| Gateway | `10.6.0.1` |
| Primary DHCP Server | `10.0.0.76` |

### What a dynamic interface actually is

A dynamic interface is the WLC's presence in a client VLAN. Wireless traffic arrives at the controller inside a CAPWAP tunnel from the AP; the controller strips the tunnel and drops the frames onto the wired network **tagged with the dynamic interface's VLAN**. It's the bridge between the wireless side and the wired side.

Interface types on a WLC:

| Type | Purpose |
|---|---|
| **Management** | Controller reachability, AP CAPWAP tunnel termination |
| **Dynamic** | One per client WLAN/VLAN — the wired-side gateway for wireless clients |
| Virtual | Internal use — DHCP relay, web auth redirect (usually 192.0.2.1) |
| Service port | Out-of-band management |

### The addressing

- **`10.6.0.2`** — the dynamic interface's own address in the Wi-Fi subnet. Note this collides conceptually with DSW-A1's VLAN 40 SVI (also 10.6.0.2 in Part 3.15). In Packet Tracer's simplified model this is what the lab asks for and it works; on real gear you'd give the dynamic interface a distinct address, since duplicate IPs on the same subnet are a genuine problem.
- **`10.6.0.1`** — the HSRP VIP, so the WLC's default gateway survives losing a distribution switch.
- **`10.0.0.76`** — R1's loopback. The WLC **relays** wireless clients' DHCP requests to R1, which is why the Wi-Fi pool lives on R1 rather than on the controller.

---

## 9.3 — Create the WLAN

**WLANs → Create New → Go**

| Field | Value |
|---|---|
| Profile Name | `Wi-Fi` |
| SSID | `Wi-Fi` |
| ID | `1` |

Then on the WLAN's configuration page:

**General tab:**
- Status: **Enabled** ✔
- Interface/Interface Group: **Wi-Fi** (the dynamic interface from 9.2)

**Security → Layer 2 tab:**
- Layer 2 Security: **WPA+WPA2**
- WPA2 Policy: ✔ enabled
- WPA2 Encryption: **AES** ✔ (make sure TKIP is unchecked)
- Auth Key Mgmt: **PSK**
- PSK Format: ASCII → `cisco123`

**Apply.**

### Two things to double-check

1. **Status → Enabled.** The WLAN is created disabled by default and a disabled WLAN doesn't broadcast. This is the most common miss in this part.
2. **Interface = Wi-Fi.** It defaults to `management`. Leaving it there puts wireless clients in VLAN 99 instead of VLAN 40, which quietly defeats the entire design.

### Wireless security recap

| Standard | Encryption | Auth | Status |
|---|---|---|---|
| WEP | RC4 | Shared key | Broken — never use |
| WPA | TKIP | PSK / 802.1X | Deprecated |
| **WPA2** | **AES-CCMP** | PSK / 802.1X | Current baseline |
| WPA3 | AES-GCMP + SAE | SAE / 802.1X | Best available |

**PSK** (personal) uses a shared passphrase. **802.1X** (enterprise) authenticates each user individually against a RADIUS server — the right answer for a real corporate network, but out of scope here.

---

## 9.4 — Verify LWAP association

**WIRELESS → Access Points → All APs**

Both LWAP1 (Office A) and LWAP2 (Office B) should be listed and associated.

### How the APs found the controller

Worth tracing, because it's the payoff for several earlier parts:

1. The AP boots on an access port in **VLAN 99** (Part 2.7).
2. It DHCPs an address from the **A-Mgmt** or **B-Mgmt** pool (Part 6.1), relayed to R1 by the distribution switch (Part 6.2).
3. The DHCP offer includes **option 43 = 10.0.0.7** (Part 6.1).
4. The AP sends a CAPWAP Join Request to 10.0.0.7.
5. The WLC accepts, and the AP downloads its config and becomes operational.

If an AP doesn't appear, walk that chain in order. Nine times out of ten it's step 2 or 3 — check `show ip dhcp binding` on R1 to see whether the AP ever leased an address.

**Give it time.** CAPWAP association isn't instant, and Packet Tracer is slower than real hardware. Wait a minute or two before assuming something's wrong.

### AP modes worth knowing

| Mode | Client traffic | Use case |
|---|---|---|
| **Local** (default, used here) | Tunnelled to the WLC via CAPWAP | Campus, controller on-site |
| FlexConnect | Switched locally at the AP | Branch offices over a WAN |
| Monitor | None — sensing only | Rogue detection, location services |
| Sniffer | None — captures frames | Troubleshooting |

Local mode is why the AP switchports are plain access ports in VLAN 99 rather than trunks (see Part 2.7).

---

## Known Packet Tracer limitation

> Wireless clients **will not** successfully lease an IP address from the Wi-Fi DHCP pool. This is a Packet Tracer limitation, not a configuration error.

The lab documents this explicitly. If both APs show as associated and the WLAN is enabled and correctly configured, Part 9 is complete — don't spend an hour chasing a client that was never going to get an address.

---

## Part 9 verification checklist

On the WLC GUI:
- CONTROLLER → Interfaces → `Wi-Fi` present with the correct VLAN, IP, gateway, and DHCP server
- WLANs → `Wi-Fi` listed, **Enabled**, mapped to the Wi-Fi interface
- WLANs → Security → WPA2 + AES + PSK confirmed
- WIRELESS → All APs → both LWAPs associated

On the CLI:

```
R1#show ip dhcp binding             ! APs should have leases
ASW-A1#show interfaces f0/2 switchport
ASW-A1#show interfaces trunk
DSW-A1#show standby brief           ! VLAN 40 group 4 present
```

---

## That's the whole lab!!

Fourteen devices, nine parts, and essentially the entire CCNA 200-301 blueprint in one topology.  Truly shout out to Jeremy again for creating this.


---

**Back to:** [README](../README.md) · [Verification cheat sheet](verification-cheatsheet.md) · [Pitfalls](pitfalls.md)
