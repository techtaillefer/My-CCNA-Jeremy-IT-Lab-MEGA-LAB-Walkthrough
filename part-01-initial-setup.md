# Part 1 — Initial Setup

Baseline device hardening. Boring, but every later part assumes it's done — SSH in Part 6 in particular depends on the local user account created here.

**Devices:** all 13 (R1, CSW1–2, DSW-A1/A2/B1/B2, ASW-A1/A2/A3/B1/B2/B3)

---

## 1.1 — Hostnames

```
Switch(config)#hostname R1
```

Repeat per device with the correct name. In Packet Tracer, do this **first** — the CLI prompt is the only reliable way to tell which of six identical-looking access switches you're currently inside.

Naming scheme used throughout:

- `R1` — edge router
- `CSW1`, `CSW2` — core switches
- `DSW-A1`, `DSW-A2`, `DSW-B1`, `DSW-B2` — distribution switches
- `ASW-A1`…`ASW-B3` — access switches

---

## 1.2 — Enable secret with the strongest available hash

The requirement is *type 9 (scrypt) if available, otherwise type 5 (MD5)*. Not every platform in this topology supports scrypt, so check first:

```
R1(config)#enable secret ?
```

If you see `algorithm-type` in the output, the device supports selecting a hash. If you don't, you only get type 5.

**R1 and all Access switches** (type 5 only):

```
enable secret jeremysitlab
```

**Core and Distribution switches** (type 9 available):

```
enable algorithm-type scrypt secret jeremysitlab
```

> Note the word order: it's `enable algorithm-type scrypt secret <pw>`, **not** `enable secret algorithm-type scrypt <pw>`. This trips people up constantly.

**Verify:**

```
show running-config | include enable
```

Type 5 hashes start with `$1$`, type 9 hashes start with `$9$`. That prefix is the fastest way to confirm you got what you asked for.

### Hash types worth knowing for the exam

| Type | Algorithm | Verdict |
|---|---|---|
| 0 | Plaintext | Never |
| 7 | Cisco proprietary, reversible | Never — trivially decoded |
| 5 | MD5 | Legacy, acceptable fallback |
| 8 | PBKDF2-SHA256 | Good |
| 9 | scrypt | Best available on IOS |

---

## 1.3 — Local user account

Same hash logic. Check support the same way:

```
R1(config)#username cisco ?
```

**R1 and all Access switches:**

```
username cisco secret ccna
```

**Core and Distribution switches:**

```
username cisco algorithm-type scrypt secret ccna
```

> Use `secret`, never `password`. `username cisco password ccna` stores type 0 plaintext (or type 7 if service password-encryption is on), which is not a hash at all.

**Verify:**

```
show running-config | include username
```

---

## 1.4 — Console line

```
line console 0
 login local
 exec-timeout 30
 logging synchronous
```

**All 13 devices.**

What each line does:

- **`login local`** — authenticate against the local user database (the `cisco` account from 1.3) instead of a shared line password. Without a `username` configured, this locks you out of the console.
- **`exec-timeout 30`** — disconnect after 30 minutes idle. The syntax is `exec-timeout <minutes> <seconds>`; omitting seconds means 0. `exec-timeout 0 0` disables the timeout entirely (convenient in a lab, terrible in production).
- **`logging synchronous`** — stops console log messages from interleaving with whatever you're mid-way through typing. Purely a quality-of-life fix, and one you'll appreciate immediately in Part 3 when HSRP starts flapping messages at you.

**Verify:**

```
show running-config | section line con
```

---

## Part 1 verification checklist

```
show running-config | include hostname
show running-config | include enable
show running-config | include username
show running-config | section line con 0
```

---

**Next:** [Part 2 — VLANs, Trunking, VTP, Layer 2 EtherChannel →](part-02-vlans-etherchannel.md)
