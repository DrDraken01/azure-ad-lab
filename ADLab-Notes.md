# AD / DC + Network Lab — Job-Relevant Notes

*A living doc. Not a step guide — this captures the stuff that maps to real tickets, real design decisions, and real interview questions. Updated as the build progresses.*

---

## The two spines (memorize these — everything else hangs off them)

1. **DNS is the backbone of Active Directory.** Domain members must point their DNS at a domain controller. Clients *find* the DC by querying DNS for **SRV records** (`_ldap`, `_kerberos`, `_gc`) under the `_msdcs` zone. No correct DNS → the client literally cannot locate the domain. This is the root cause of most "domain join failed" and "GPO not applying" tickets.
   - **Troubleshooting reflex:** when domain stuff breaks, check DNS *first*. `nslookup lab.local`, then `nslookup -type=SRV _ldap._tcp.dc._msdcs.lab.local`.

2. **Kerberos depends on time.** Default Kerberos clock-skew tolerance is ~5 minutes. If a client's clock drifts past that, auth fails — and the error message won't mention time. The DC (specifically the **PDC Emulator** FSMO role holder) is the authoritative time source for the domain; it should sync to an external NTP source, and everything else syncs down from it. `w32tm /query /status` is your friend.

---

## Part B — Domain Controller build

- **DC gets a static IP, always.** A DC that pulls DHCP is a broken DC — its own DNS/AD records get unstable. In real environments, all servers are static or DHCP-reserved.
- **DC points DNS at itself** (here, `192.168.10.10`). This is expected and correct. In multi-DC environments the convention is: point at *another* DC as primary and self as secondary (avoids a subtle startup dependency), but self-pointing is fine for a single-DC lab.
- **`.local` is a legacy/lab-only choice — know why.** Real orgs no longer use `.local` because it collides with mDNS/Bonjour and you can't get publicly-trusted TLS certs for it. Modern practice is a subdomain of a domain you own, e.g. `ad.company.com` or `corp.company.com`. **Good interview line:** "I used `.local` for the lab, but in production I'd use a routable subdomain of an owned domain for cert and naming hygiene."
- **DSRM password = Directory Services Restore Mode.** It's the break-glass local account used to boot into AD recovery mode when the directory itself is broken. Not the same as the domain admin password. Real ops teams vault this separately.
- **Rename the server *before* promoting to DC.** Renaming a live DC is painful; get the name right first.
- After promotion, verifying in **DNS Manager** that `lab.local` forward zone and `_msdcs` exist = confirming the SRV plumbing that makes step 11 (domain join) even possible.

## Part C — Domain join

- The whole success/failure of the join hinges on **client DNS = DC IP**. If join fails, this is the first thing to check, every time.
- `LAB\Administrator` vs `Administrator`: the `LAB\` prefix specifies the *domain* identity vs the client's local account. Knowing the difference between local and domain accounts is basic but interviewers probe it.

## Part D — DHCP

- **The three options that matter and why:**
  - `003` Router → default gateway
  - `006` DNS → **must be the DC**, not a public resolver. Handing out `8.8.8.8` here silently breaks all domain functionality. This is the "DHCP hands out wrong DNS" gotcha.
  - `015` Domain → DNS suffix
- **Exclusions vs reservations:** exclude the static server range from the scope so DHCP never hands out an in-use address. Reservations (MAC → fixed IP) are how you give a device a stable address while still managing it via DHCP.

## Part E — Users, groups, OUs, GPO

- **OUs are for two things: delegation and GPO targeting.** They are *not* security groups. You link policy to OUs and delegate admin rights at the OU level. Common beginner confusion — don't mix up OUs and groups in an interview.
- **GPO precedence = LSDOU:** Local → Site → Domain → OU, with the *last* applied winning (closest to the object). Add security filtering and (rarely) WMI filters on top.
- **Computer Config vs User Config:** the Control Panel restriction in this lab is *User* config, so it follows the user, not the machine.
- **Troubleshooting reflex:** `gpupdate /force` to reapply, `gpresult /r` (or `/h report.html`) to see what actually applied and *why* — this is how you prove a policy is or isn't hitting a user.

## Part F — Network side (Packet Tracer)

- **Separating a SERVERS VLAN from a CLIENTS VLAN is a security practice**, not just a lab exercise — it's network segmentation, which limits lateral movement. Good thing to name explicitly when discussing the design.
- **Router-on-a-stick:** one physical link, dot1q subinterfaces (`.10`, `.20`), each subinterface is the gateway for its VLAN. The trunk must *allow* both VLANs or inter-VLAN routing dies.
- **`ip helper-address` / DHCP relay — the concept that ties the two halves together.** DHCP uses broadcasts, and broadcasts don't cross routers. So when clients live in a different subnet than the DHCP server, the router's VLAN interface needs a helper-address pointing at the DHCP server to *forward* the request as unicast. This is one of the most common real-world designs (centralized DHCP) and a frequent interview question.

## Part G — Lab limitations (be honest about these)

- Packet Tracer **cannot host real AD** and can't bridge to your VM traffic. It's network *simulation*; the VMs are your real identity plane. For actually routing VM client traffic across simulated VLANs you'd need **GNS3 or EVE-NG**. Knowing this distinction shows you understand what each tool is actually for.

---

## Hybrid / cloud tie-in (leverage your Entra + Intune background)

This on-prem lab is the *foundation* under everything you've already touched in the cloud. Worth connecting explicitly:

- On-prem AD extends to **Entra ID** via **Entra Connect (formerly Azure AD Connect)**, which syncs identities up to the cloud — this is **hybrid identity**.
- Devices can be **hybrid-joined** (on-prem AD + Entra registered), which is exactly the world Intune/M365 management lives in.
- **Strong interview narrative:** "I built the on-prem AD/DNS/DHCP foundation from scratch, and I understand how it extends to hybrid via Entra Connect — so I can reason about identity from the domain controller all the way up to Intune policy."

---

## Interview soundbites / troubleshooting reflexes (running list)

- "Domain issues? Check DNS first." — SRV records under `_msdcs`.
- "Auth failing intermittently? Check time skew." — Kerberos 5-min window, PDC emulator as time source.
- "Prove GPO application with `gpresult`, not assumptions."
- "DHCP across subnets means `ip helper-address`."
- "OUs are for GPO and delegation, not security."
