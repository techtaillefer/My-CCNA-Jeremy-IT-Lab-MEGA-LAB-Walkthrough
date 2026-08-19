# Part 6 — Network Services: DHCP, DNS, NTP, SNMP, Syslog, FTP, SSH, NAT

The longest part of the lab. Everything that makes the network *usable* rather than just routable.

---

## 6.1 — DHCP server on R1

Seven pools. The first ten usable addresses of every subnet are excluded (reserved for infrastructure — gateways, HSRP addresses, the WLC, switch SVIs).

### Exclusions first

```
ip dhcp excluded-address 10.0.0.1 10.0.0.10
ip dhcp excluded-address 10.1.0.1 10.1.0.10
ip dhcp excluded-address 10.2.0.1 10.2.0.10
ip dhcp excluded-address 10.0.0.17 10.0.0.26
ip dhcp excluded-address 10.3.0.1 10.3.0.10
ip dhcp excluded-address 10.4.0.1 10.4.0.10
ip dhcp excluded-address 10.6.0.1 10.6.0.10
```

Note Office B management: `10.0.0.16/28` — the network address is .16, so the **first usable is .17** and the ten excluded run **.17 to .26**. Every other subnet is a /24 or a /28 starting on .0, so it's .1–.10.

Configure exclusions **before** the pools. If a client leases an address you later exclude, the exclusion doesn't revoke the existing lease.

### The pools

```
ip dhcp pool A-Mgmt
 network 10.0.0.0 255.255.255.240
 default-router 10.0.0.1
 dns-server 10.5.0.4
 domain-name jeremysitlab.com
 option 43 ip 10.0.0.7
!
ip dhcp pool A-PC
 network 10.1.0.0 255.255.255.0
 default-router 10.1.0.1
 dns-server 10.5.0.4
 domain-name jeremysitlab.com
!
ip dhcp pool A-Phone
 network 10.2.0.0 255.255.255.0
 default-router 10.2.0.1
 dns-server 10.5.0.4
 domain-name jeremysitlab.com
!
ip dhcp pool B-Mgmt
 network 10.0.0.16 255.255.255.240
 default-router 10.0.0.17
 dns-server 10.5.0.4
 domain-name jeremysitlab.com
 option 43 ip 10.0.0.7
!
ip dhcp pool B-PC
 network 10.3.0.0 255.255.255.0
 default-router 10.3.0.1
 dns-server 10.5.0.4
 domain-name jeremysitlab.com
!
ip dhcp pool B-Phone
 network 10.4.0.0 255.255.255.0
 default-router 10.4.0.1
 dns-server 10.5.0.4
 domain-name jeremysitlab.com
!
ip dhcp pool Wi-Fi
 network 10.6.0.0 255.255.255.0
 default-router 10.6.0.1
 dns-server 10.5.0.4
 domain-name jeremysitlab.com
```

> **Watch the spelling of `default-router`.** The original notes I worked from had `deafult-router` in the A-Mgmt pool. IOS rejects it, and it's easy to miss the error scrolling past when you're pasting a large block. If a pool isn't handing out gateways, check this first.

### Option 43 — WLC discovery

```
option 43 ip 10.0.0.7
```

This is how a lightweight AP finds its controller. The AP DHCPs an address in the management VLAN, reads option 43 from the offer, and starts a CAPWAP join to that address.

Only the two **management** pools carry it, because the APs live in VLAN 99 (see Part 2.7 — local mode, not FlexConnect). The user pools don't need it.

> **PT vs real IOS:** Packet Tracer accepts the friendly `option 43 ip 10.0.0.7`. Real IOS wants a hex TLV: `option 43 hex f104_0a00_0007` — type `f1`, length `04`, then the WLC address in hex. Worth recognising if you see it in the wild.

### Verify

```
show ip dhcp pool
show ip dhcp binding
show running-config | section dhcp
```

`show ip dhcp pool` gives leased-address counts per pool. `show ip dhcp binding` lists actual client leases — it stays empty until relay is configured in 6.2.

To clear leases while testing:

```
clear ip dhcp binding *
```

---

## 6.2 — DHCP relay

R1 is the DHCP server, but DHCP DISCOVER messages are broadcasts and routers don't forward broadcasts. The distribution switches — the default gateways for these VLANs — need to relay them.

**DSW-A1 and DSW-A2:**

```
interface vlan 10
 ip helper-address 10.0.0.76
!
interface vlan 20
 ip helper-address 10.0.0.76
!
interface vlan 40
 ip helper-address 10.0.0.76
!
interface vlan 99
 ip helper-address 10.0.0.76
```

**DSW-B1 and DSW-B2:**

```
interface vlan 10
 ip helper-address 10.0.0.76
!
interface vlan 20
 ip helper-address 10.0.0.76
!
interface vlan 30
 ip helper-address 10.0.0.76
!
interface vlan 99
 ip helper-address 10.0.0.76
```

### How relay picks the right pool

`ip helper-address` converts the broadcast into a **unicast** aimed at R1's loopback, and — crucially — stamps the relay agent's own interface IP into the DHCP packet's **giaddr** field. R1 reads giaddr, matches it against its pool `network` statements, and picks the right pool. Without giaddr, R1 would have no way to know which subnet the request came from.

**Configure it on both DSWs in each office**, not just the HSRP Active one. If the Active router fails, the Standby has to keep relaying.

The target is the **loopback** (10.0.0.76), not a physical interface — the loopback stays up as long as R1 is up and is reachable via OSPF over either core path.

`ip helper-address` also relays several other UDP broadcast services by default (TFTP, DNS, TIME, NetBIOS, TACACS). You can trim that with `no ip forward-protocol udp <port>`.

**Verify:** open PC1 → Desktop → IP Configuration → DHCP. It should lease an address in 10.1.0.11 or above (the first ten are excluded), with gateway 10.1.0.1 and DNS 10.5.0.4. Then on R1:

```
show ip dhcp binding
```

---

## 6.3 — DNS records on SRV1

GUI, not CLI. **SRV1 → Services → DNS → On**, then add:

| Name | Type | Detail |
|---|---|---|
| google.com | A Record | 172.253.62.100 |
| youtube.com | A Record | 152.250.31.93 |
| jeremysitlab.com | A Record | 66.235.200.145 |
| www.jeremysitlab.com | **CNAME** | jeremysitlab.com |

The fourth one is a **CNAME** (canonical name), not an A record — it's an alias pointing at another *name*, which then resolves to an address. Selecting "A Record" and typing a hostname into the address field won't work.

Don't forget to flip the DNS **Service** to **On** at the top of the panel. Records without the service enabled do nothing.

---

## 6.4 — Domain name and DNS client on all devices

**All 13 routers and switches:**

```
ip domain name jeremysitlab.com
ip name-server 10.5.0.4
```

Two reasons this matters:

1. It lets devices resolve names, so `ping jeremysitlab.com` works from the CLI.
2. **`ip domain name` is a prerequisite for generating RSA keys** in 6.10. `crypto key generate rsa` fails without a hostname and a domain name, because the key is labelled `hostname.domain`. Do this before attempting SSH.

> Older IOS uses `ip domain-name` (with a hyphen). Newer uses `ip domain name` (space). Packet Tracer generally accepts both; use `?` if one is rejected.

**Also set SRV1's own DNS server** to 10.5.0.4 — SRV1 → Config → Global Settings → DNS Server. Without it, SRV1 can't resolve names for its own tests.

---

## 6.5 — NTP server on R1

```
ntp master 5
ntp server 216.239.35.0
```

- **`ntp master 5`** — R1 acts as an NTP server at **stratum 5**. Stratum is distance from the authoritative clock: stratum 0 is the reference clock itself (atomic, GPS), stratum 1 is directly attached to it, and so on. Lower is more trusted.
- **`ntp server 216.239.35.0`** — R1 is simultaneously a client of an upstream server (that address is one of Google's public NTP servers).

> **Patience required.** NTP synchronisation is deliberately slow, and Packet Tracer makes it slower. `show ntp status` may read "unsynchronised" for a long while. Configure it, verify the syntax, and move on — the lab explicitly tells you not to wait.

---

## 6.6 — Authenticated NTP clients

**R1** (must define the same key it will be authenticated against):

```
ntp authentication-key 1 md5 ccna
ntp trusted-key 1
```

**All Core, Distribution, and Access switches:**

```
ntp authentication-key 1 md5 ccna
ntp trusted-key 1
ntp authenticate
ntp server 10.0.0.76 key 1
```

### How NTP authentication works

- `ntp authentication-key 1 md5 ccna` — defines key **number** 1 with password `ccna`. The number and the password must match on both ends.
- `ntp trusted-key 1` — marks key 1 as trusted for authentication.
- `ntp authenticate` — turns authentication on globally. Without it, the key is defined but unused.
- `ntp server 10.0.0.76 key 1` — points at R1's loopback and authenticates using key 1.

The client authenticates the **server**, not the other way round. The point is to stop an attacker from impersonating your time source — which matters more than it sounds, because bogus time breaks certificate validation, invalidates log timestamps, and can lock out time-based access rules.

> `ntp authenticate` is required on real IOS but is often accepted-and-ignored in Packet Tracer. Include it; it's correct and costs nothing.

**Verify:**

```
show ntp status
show ntp associations
show clock
```

In `show ntp associations`, a `*` next to an address is the currently synced peer.

---

## 6.7 — SNMP

**All 13 devices:**

```
snmp-server community SNMPSTRING ro
```

`ro` = read-only, so the community string permits **GET** but not **SET**. `rw` would allow SET (writing config via SNMP) — explicitly not wanted here.

The community string is SNMPv2c's entire security model: a shared plaintext password sent in every packet. This is why SNMPv3 exists — it adds real authentication and encryption. For CCNA purposes, know that v1 and v2c are insecure and v3 is not.

**Verify:**

```
show snmp community
show snmp
```

---

## 6.8 — Syslog

**All 13 devices:**

```
logging host 10.5.0.4
logging trap debugging
logging buffered 8192
```

- **`logging host 10.5.0.4`** — send messages to SRV1. Older syntax `logging 10.5.0.4` works identically.
- **`logging trap debugging`** — severity **7**, the highest-numbered level, which means *log everything*. You can type `logging trap 7` instead.
- **`logging buffered 8192`** — keep a local 8192-byte ring buffer in RAM.

### Syslog severity levels

Mnemonic: **E**very **A**wesome **C**isco **E**ngineer **W**ill **N**eed **I**ce-cream **D**aily.

| # | Level | Meaning |
|---|---|---|
| 0 | Emergency | System unusable |
| 1 | Alert | Immediate action needed |
| 2 | Critical | Critical condition |
| 3 | Error | Error condition |
| 4 | Warning | Warning condition |
| 5 | Notification | Normal but significant |
| 6 | Informational | Informational (default for most) |
| 7 | Debugging | Debug output |

Setting a level logs that level **and everything more severe** (lower numbered).

Also make sure SRV1's **Syslog service is On** (SRV1 → Services → SYSLOG).

**Verify:**

```
show logging
```

Confirm the trap logging level, the buffer size, and the host. Then shut and unshut an interface and watch the message appear in SRV1's syslog window.

---

## 6.9 — IOS upgrade over FTP

**R1:**

```
ip ftp username cisco
ip ftp password cisco
```

Then from privileged EXEC:

```
R1#copy ftp: flash:
Address or name of remote host []? 10.5.0.4
Source filename []? c2900-universalk9-mz.SPA.155-3.M4a.bin
Destination filename [c2900-universalk9-mz.SPA.155-3.M4a.bin]? <Enter>
```

**This takes several minutes in Packet Tracer.** Genuinely — go make a coffee. Don't click around, and don't assume it's hung.

Once it completes:

```
R1#show flash:
```

Confirm both the old and the new image are present, and check free space.

Set the new image as the boot file:

```
R1(config)#boot system flash:c2900-universalk9-mz.SPA.155-3.M4a.bin
```

```
R1#write memory
R1#reload
```

After reload, confirm the new version loaded, then remove the old image:

```
R1#show version
R1#delete flash:c2900-universalk9-mz.SPA.151-4.M4.bin
```

### Notes

- **`write memory` before `reload`** or the boot statement is lost and you'll boot the old image again.
- Check `show run | include boot` first — if there's an existing `boot system` line pointing at the old image, remove it with `no boot system ...`. IOS tries boot statements **in order**, so a stale first entry wins.
- With no `boot system` statement at all, IOS boots the first valid image it finds in flash — which may or may not be the one you want.
- The exact old filename varies; read it off `show flash:` rather than assuming.

---

## 6.10 — SSH

**All 13 devices:**

```
crypto key generate rsa
```

When prompted for modulus size:

```
How many bits in the modulus [512]: 4096
```

**4096 is the largest supported** and the requirement asks for the largest. It takes noticeably longer to generate — that's normal.

Then:

```
ip ssh version 2
!
access-list 1 permit 10.1.0.0 0.0.0.255
!
line vty 0 15
 access-class 1 in
 transport input ssh
 login local
 logging synchronous
```

### Line by line

- **`crypto key generate rsa`** — creates the asymmetric key pair SSH uses. Requires hostname + domain name (done in 6.4). Generating keys is also what implicitly enables the SSH server.
- **`ip ssh version 2`** — forces SSHv2 only. SSHv1 has known vulnerabilities; without this the device runs in compatibility mode accepting both.
- **`access-list 1 permit 10.1.0.0 0.0.0.255`** — standard ACL matching only Office A's PC subnet. Standard ACLs filter on **source only**, which is exactly what's wanted here. Remember the implicit `deny any` at the end does the actual restricting.
- **`access-class 1 in`** — applies the ACL to VTY lines. Note the keyword: it's `access-class` on VTY lines, **not** `ip access-group` (that's for physical interfaces). This is a favourite exam distinction.
- **`transport input ssh`** — SSH only; Telnet refused. The default on many platforms is `transport input all` or `none`.
- **`login local`** — authenticate against the local user database (the `cisco`/`ccna` account from Part 1).
- **`logging synchronous`** — same quality-of-life fix as the console line.

### Test it

From **PC1** (in 10.1.0.0/24 — permitted):

```
C:\> ping 10.0.0.76
C:\> ssh -l cisco 10.0.0.76
Password: ccna
```

Should connect.

From **PC3** (in 10.3.0.0/24 — Office B, denied):

```
C:\> ping 10.0.0.76
C:\> ssh -l cisco 10.0.0.76
```

Ping succeeds (the ACL only guards VTY access, not IP reachability) but SSH is refused. That contrast is the proof the `access-class` is working.

**Verify:**

```
show ip ssh
show ssh
show access-lists
show running-config | section line vty
```

---

## 6.11 — Static NAT

```
ip nat inside source static 10.5.0.4 203.0.113.113
!
interface range g0/0/0,g0/1/0
 ip nat outside
!
interface range g0/0-1
 ip nat inside
```

Static NAT is a permanent one-to-one mapping. It exists so hosts **on the Internet can initiate connections inbound** to SRV1 — dynamic NAT/PAT only supports outbound-initiated flows.

### The inside/outside boundary

NAT does nothing until interfaces are marked. Getting this backwards is the number one NAT failure:

- **`ip nat inside`** — G0/0 and G0/1 (facing the LAN / private addresses)
- **`ip nat outside`** — G0/0/0 and G0/1/0 (facing the Internet / public addresses)

### NAT terminology

| Term | Meaning | Here |
|---|---|---|
| Inside local | Private address, as seen inside | 10.5.0.4 |
| Inside global | Public address, as seen outside | 203.0.113.113 |
| Outside local | Remote host as seen from inside | 172.253.62.100 |
| Outside global | Remote host's real address | 172.253.62.100 |

"Inside/outside" = whose network. "Local/global" = which side of the router you're observing from.

**Verify:**

```
show ip nat translations
show ip nat statistics
show ip nat translations verbose
```

The static entry appears in the translation table immediately, before any traffic — that's what distinguishes static from dynamic.

---

## 6.12 — Dynamic PAT

```
access-list 2 permit 10.1.0.0 0.0.0.255
access-list 2 permit 10.2.0.0 0.0.0.255
access-list 2 permit 10.3.0.0 0.0.0.255
access-list 2 permit 10.4.0.0 0.0.0.255
access-list 2 permit 10.6.0.0 0.0.0.255
!
ip nat pool POOL1 203.0.113.200 203.0.113.207 netmask 255.255.255.248
!
ip nat inside source list 2 pool POOL1 overload
```

### Detail worth noting

**Order matters** — the requirement specifies the order of the ACL entries explicitly. ACLs are evaluated top-down, and while the result is identical here (no overlapping ranges), the grading and general good practice both care.

**Management VLANs are excluded on purpose.** 10.0.0.0/28 and 10.0.0.16/28 are infrastructure; they have no business browsing the Internet. Only user subnets get translated.

**`overload` is the keyword that makes it PAT.** Without it, this is plain dynamic NAT: 8 public addresses, 8 simultaneous hosts, and the ninth gets dropped. With `overload`, the router adds a unique source **port number** to each translation, so all eight addresses together support tens of thousands of concurrent sessions.

**`netmask 255.255.255.248`** = /29, matching the 203.0.113.200–207 range. You can also write `prefix-length 29`.

### Test

From PC1:

```
C:\> ping jeremysitlab.com
```

This exercises the whole stack at once: DHCP gave PC1 its address and DNS server, DNS resolved the name via SRV1, OSPF routed it to R1, the default static route pointed at the ISP, and PAT translated it.

On R1:

```
R1#show ip nat translations
```

You should see entries mapping `10.1.0.x:port` → `203.0.113.200:port`.

### Failover test

```
R1(config)#interface g0/0/0
R1(config-if)#shutdown
!
R1(config)#router ospf 1
R1(config-router)#no default-information originate
R1(config-router)#default-information originate
```

```
R1#show ip route
```

The `S*` default should now be via 203.0.113.5 (the floating static took over). Ping `jeremysitlab.com` from PC1 again — it should still work.

Restore:

```
R1(config)#interface g0/0/0
R1(config-if)#no shutdown
!
R1(config)#router ospf 1
R1(config-router)#no default-information originate
R1(config-router)#default-information originate
```

The re-issuing of `default-information originate` is the Packet Tracer workaround explained in [Part 5](part-05-routing.md#the-packet-tracer-failover-quirk) — real IOS would use `default-information originate always` and need none of this.

---

## 6.13 — Replace CDP with LLDP

**R1, Core, and Distribution switches:**

```
no cdp run
lldp run
```

**All Access switches:**

```
no cdp run
lldp run
!
interface f0/1
 no lldp transmit
```

### Why

**CDP** is Cisco-proprietary and enabled by default. It's genuinely useful for mapping a network, which is exactly why it's a security concern — it broadcasts your device model, IOS version, IP addresses, and port IDs to anything on the wire.

**LLDP** (802.1AB) is the vendor-neutral equivalent and is **disabled by default**, so `lldp run` is required.

`no lldp transmit` on the access ports stops the switch from advertising itself to end hosts, while `receive` stays on so the switch still learns what's attached. Transmit and receive are separately controllable — a useful distinction CDP doesn't offer.

> Note for VoIP: `switchport voice vlan` relies on CDP or LLDP-MED to tell the phone its VLAN. Disabling CDP *and* LLDP Tx on F0/1 means the phones can't be told their voice VLAN dynamically any more. In this lab that's accepted; in production you'd leave LLDP-MED transmitting on phone ports.

**Verify:**

```
show cdp
show lldp
show lldp neighbors
show lldp interface f0/1
```

`show cdp` should report that CDP is not enabled. `show lldp neighbors` takes up to 30 seconds to populate after enabling.

---

## Part 6 verification checklist

```
show ip dhcp pool
show ip dhcp binding
show ip nat translations
show ip nat statistics
show ip ssh
show access-lists
show logging
show ntp status
show ntp associations
show snmp community
show lldp neighbors
show cdp                            ! should be disabled
show flash:
show version
```

From PC1: `ping jeremysitlab.com`, `ssh -l cisco 10.0.0.76`.

---

**Next:** [Part 7 — Security: ACLs and Layer 2 Hardening →](part-07-security.md)
