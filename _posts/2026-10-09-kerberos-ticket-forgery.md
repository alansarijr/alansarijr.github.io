---
layout: post
title: "Mining the Domain: Digging Up Golden, Silver, Diamond & Sapphire Tickets"
categories: [security]
tags: [active directory, kerberos, red team, blue team, detection]
toc: true
mermaid: true
---

*How attackers forge the keys to the kingdom — and how defenders tell a real ticket from a counterfeit.*

---

## TL;DR

If you only remember one table from this post, make it this one:

| Attack | What's forged | Secret needed | Where the DC sees it | How it's caught |
|---|---|---|---|---|
| **Golden** | A TGT, from scratch | `krbtgt` key | TGS requests with no login before them | Missing `4768` before `4769` |
| **Silver** | A service ticket (TGS) | One service/machine key | **Nothing at all** | Login on the host with no matching DC traffic |
| **Diamond** | A *real* TGT, then tampered | `krbtgt` key | Looks completely normal | Ticket's claimed privileges ≠ what AD actually records |
| **Sapphire** | A real TGT carrying a *stolen* real PAC | `krbtgt` key | Normal, plus one odd delegation request | Privilege mismatch + a suspicious S4U2self request |

Four attacks, each built to dodge the detection that caught the one before it. The arc below walks through that cat-and-mouse story — and why the last two are the hardest to see.

---

## Introduction

If you've spent any time around Active Directory red teaming, you've heard "pass the ticket" and *Golden Ticket* tossed around like magic spells. In a way, they are: a Golden Ticket is a master key that impersonates anyone, anywhere in a domain, for as long as the attacker wants.

But the forged-ticket family is bigger than Golden Tickets — and the differences between its members are exactly what makes detection engineering interesting. Each variant was designed, consciously or not, to evade the detection logic of the last:

- **Golden Ticket** — forge a Ticket Granting Ticket (TGT) from scratch.
- **Silver Ticket** — forge a service ticket (TGS) and skip the KDC entirely.
- **Diamond Ticket** — don't forge at all; *get a real TGT from the KDC and tamper with it.*
- **Sapphire Ticket** — don't even tamper; make the KDC hand you a *real privileged PAC* and reuse it.

We'll start with a quick Kerberos refresher, break down each attack (how it works, what it needs, why it's stealthy), and finish with a practical detection playbook for the blue teamers in the room.

> New to this? Read the refresher first — every attack below is just "which step did the attacker cheat on, and whose key let them do it?"
{: .prompt-tip }

---

## A 60-Second Kerberos Refresher

Everything hinges on how Kerberos normally works. In an AD domain the **Key Distribution Center (KDC)** — running on every Domain Controller — is the trusted broker of identity.

The simplified flow:

```text
Client                                  KDC                     Service
  |                                      |                         |
  |--- 1. AS-REQ (prove I'm alice) ----->|                         |
  |<-- 2. AS-REP: TGT (enc: krbtgt key) -|                         |
  |                                      |                         |
  |--- 3. TGS-REQ (TGT + "I want CIFS -->|                         |
  |            on fileserver01")         |                         |
  |<-- 4. TGS-REP: service ticket ------ |                         |
  |        (enc: service's key)          |                         |
  |                                      |                         |
  |----------- 5. AP-REQ (service ticket) ----------------------->|
  |<------------------------- 6. Access granted -------------------|
```

Two facts to hold onto:

1. **The TGT is encrypted with the `krbtgt` account's key.** Only the KDC can read or validate it. Inside the TGT lives the **PAC** (Privilege Attribute Certificate) — the structure that says who you are and which groups you belong to.
2. **A service ticket is encrypted with the target service's own key** (e.g., a machine account's key). The service decrypts it locally and trusts what it says.

Each forged-ticket attack exploits the theft of one of these keys. **Whoever holds the key can mint tickets the recipient will trust.** That's the whole game.

---

## Golden Ticket: Forging the Master Key

### The idea

The `krbtgt` account's password hash — or better, its AES key — is the single most sensitive secret in a domain, because it encrypts *every* TGT. Steal it (via DCSync, a dumped `ntds.dit`, or a compromised DC) and you no longer need the KDC. **You become the KDC.**

With the krbtgt key, an attacker forges a TGT for *anyone*: Domain Admins, nonexistent users, accounts with 10-year lifetimes. The domain's own KDC accepts it and issues service tickets against it, because as far as it can tell, it issued the thing itself.

```text
Attacker (holds krbtgt key)
   |
   | [forges TGT for "Administrator" entirely offline]
   |
   |--- TGS-REQ (forged TGT) ---> KDC: "looks valid to me"
   |<-- TGS-REP: service ticket --
   |
   |--- AP-REQ ---> Service: owned
```

> **Seen in the wild as:** `mimikatz "kerberos::golden /user:Administrator /domain:... /sid:... /krbtgt:<hash> /ptt"` or Impacket's `ticketer.py`. The `/ptt` ("pass the ticket") injects it straight into memory.
{: .prompt-info }

### Why it gets caught

The Golden Ticket's original sin is the **missing AS exchange**. The attacker never did steps 1–2 — they jumped straight to presenting a TGT. On the DC you'll see:

- Event **`4769`** (service ticket requested) **with no preceding `4768`** (TGT requested) for that user.

A SOC that correlates these events can spot it. Beyond that, lazy forgeries leak signatures:

- **Lifetime anomalies** — Mimikatz historically forged 10-year tickets; domain policy is usually 10 hours. A ticket that violates `MaxTicketAge` is a smoking gun.
- **Encryption downgrade** — forging with RC4 when the domain speaks AES256 shows up in the `Ticket Encryption Type` field of Kerberos events (`0x17` = RC4, `0x12` = AES256).
- **Impossible identities** — tickets for users that don't exist, or for `Administrator` accounts that never authenticate that way.

The red team's answer was simple: *make the forgery prettier* — match lifetimes, use AES, impersonate real accounts. But the missing-`4768` problem is **structural**, not cosmetic. That problem is what eventually produced the Diamond ticket.

---

## Silver Ticket: The KDC Never Knows

### The idea

If the Golden Ticket forges the *first* artifact of the flow, the Silver Ticket forges the *second*: the **service ticket (TGS)** itself.

A service ticket is encrypted with the target service's key — the machine account hash of `fileserver01`, say, or the password hash of a service account running an SPN. Steal that key (dump it from the host's memory, crack a weak service password, grab it from a DC) and you can forge a service ticket offline:

```text
Attacker (holds fileserver01$ machine account key)
   |
   | [forges CIFS service ticket for "Administrator" offline]
   |
   |--- AP-REQ (forged service ticket) ---> fileserver01
   |<--------------- Access granted ------
   |
   KDC: (never contacted)
```

Notice what's missing: **no AS-REQ, no AS-REP, no TGS-REQ, no TGS-REP.** The attacker goes straight to the service with an AP-REQ. The Domain Controller — where almost all Kerberos monitoring lives — sees *nothing at all*.

Silver tickets are scoped to one service on one box, but that scope is usually enough: CIFS for a file server, MSSQL for a database, HTTP for an internal app. And since machine account passwords rotate only every ~30 days (and service account passwords sometimes *never*), the key has a long shelf life.

> **Seen in the wild as:** `mimikatz "kerberos::golden ... /target:fileserver01 /service:cifs /rc4:<machine-hash>"` (same module, scoped to a service) or Impacket `ticketer.py -spn cifs/...`.
{: .prompt-info }

### Why it gets caught

You cannot detect Silver tickets at the DC. Full stop. Detection moves **to the target host and into the correlation gap between host and DC**:

- **DC–host correlation** — the target server logs a **`4624`** (successful logon) for a user, but the DCs show *no* Kerberos activity for that user anywhere near that time. Authentication that appears with no corresponding KDC traffic is the signature.
- **PAC validation** — a service *can* ask the KDC to verify a ticket's PAC. The forged ticket's **server checksum** is fine (the attacker has the service key), but the **KDC checksum needs the krbtgt key**, which the Silver-ticket attacker does *not* have. Services configured to validate PACs against the KDC can catch this.
- **Identity absurdities** — the same tells as Golden tickets: nonexistent users, impossible privilege levels, odd logon times.

Silver tickets are the quiet ones of the family: they trade breadth for near-total invisibility at the network's center.

---

## Diamond Ticket: The Real One, Tampered

### The idea

Diamond tickets exist because red teamers looked at the Golden Ticket's "missing `4768`" detection and asked: *why forge a TGT from scratch when the KDC will just... give me one?*

```text
Attacker (holds krbtgt key + any valid low-priv account)
   |
   |--- 1. AS-REQ (as lowpriv_user) ----> KDC
   |<-- 2. AS-REP: a REAL, genuine TGT --      <- 4768 logged. Looks normal.
   |
   | [3. Decrypt the TGT with the krbtgt key]
   | [4. Modify the PAC: inject Domain Admins (RID 512),
   |     Enterprise Admins (519), or swap identity]
   | [5. Recompute PAC checksums, re-encrypt with krbtgt key]
   |
   |--- 6. TGS-REQ (modified TGT) -------> KDC
   |<-- 7. TGS-REP: service ticket with escalated PAC
   |
   |--- 8. AP-REQ -----------------------> Service: owned
```

The attacker performs a **completely legitimate AS-REQ** — often as their own unprivileged account — and receives a genuine TGT. Because it's encrypted with the stolen krbtgt key, they decrypt it, rewrite the PAC to grant themselves Domain Admin membership, recompute the signatures, and re-encrypt. They then present this "upgraded" TGT in a TGS-REQ, and the KDC issues service tickets carrying the inflated privileges.

> **Seen in the wild as:** Rubeus `diamond /krbtgt:<hash> /user:lowpriv /ticketuser:Administrator /groups:512 ...`. The tell is that a *real* TGT request precedes everything.
{: .prompt-info }

### Why it's so stealthy

This attack defeats the two detections that work against its cousins:

1. **Sequence analysis dies.** There *is* a real `4768` before every `4769`. The timeline looks textbook-normal.
2. **Cryptographic PAC validation dies.** The crucial detail: in a TGT, *both* PAC signatures (server checksum and KDC checksum) are computed with the **krbtgt key** — because for a TGT, the "server" *is* the KDC. The Diamond attacker holds that key, so their recomputed signatures are **cryptographically perfect**. Unlike the Silver ticket, there's no checksum they can't forge.

On top of that, the ticket's timestamps, KVNO, and encryption types are all authentic — a real KDC issued the underlying ticket moments ago.

### So how *is* it detected?

The attacker can make the *ticket* perfect, but not the *context*. Detection lives in the gap between **what the ticket claims** and **what reality says**:

**1. Privilege-vs-ground-truth mismatch (the strongest signal).**
The tampered PAC says "member of Domain Admins." Active Directory says "member of Domain Users." Legitimate privilege elevation *always* leaves a breadcrumb — a group-membership change event (**`4728`/`4732`/`4756`**). A Diamond ticket produces admin rights out of *thin air*, with no group change anywhere. Comparing the privileges a ticket exercises against the account's real AD membership is the one detection the attacker fundamentally cannot evade.

**2. AS-REQ ↔ TGS-REQ identity correlation.**
If the attacker swapped identities (got the TGT as `lowpriv_user` but impersonates `Administrator`), the `4768` and later `4769`s show different accounts from the same source IP with no legitimate bridge. This catches the identity-swap variant — but *not* the subtler "same username, injected groups" variant, which is why layer 1 matters.

**3. Behavioral context.**
The initial AS-REQ is a real event with a real footprint: the attacker's host, IP, and time of day. A low-priv account that has never requested a ticket to `cifs/dc01` suddenly doing so minutes after a TGT request is a UEBA-grade anomaly, no matter how valid the ticket is.

**4. Network-layer PAC inspection.**
Tools that parse Kerberos traffic to the DC (e.g., Microsoft Defender for Identity, NDR platforms) compare PAC authorization data on the wire against synced directory state — flagging tickets whose group claims don't match AD, no matter how perfect the signatures.

**5. Kill the prerequisite.**
A Diamond ticket needs the krbtgt key. Alert on DCSync (replication from non-DCs), `ntds.dit` access, and any interactive logon by the `krbtgt` account itself — it should **never** log on, ever. And the response that erases everything: **rotate the krbtgt password twice**, and every forged or tampered ticket in existence becomes unreadable garbage.

---

## Side-by-Side Comparison

| | **Golden** | **Silver** | **Diamond** | **Sapphire** |
|---|---|---|---|---|
| Forged artifact | TGT | Service ticket (TGS) | Real TGT, tampered | Real TGT + stolen real PAC |
| Secret required | krbtgt key | Service/machine key | krbtgt key | krbtgt key |
| AS-REQ on the DC? | No, skipped | No, skipped | Yes, legitimate | Yes, legitimate |
| TGS-REQ on the DC? | Yes, forged TGT | No, never contacts KDC | Yes, tampered TGT | Yes, plus an S4U2self request |
| KDC visibility | TGS-REQs only | **None at all** | Full, normal-looking | Full, normal-looking |
| PAC signatures | Valid if forged carefully | Server valid / KDC invalid | Cryptographically perfect | Cryptographically perfect |
| PAC contents | Synthetic | Synthetic | Synthetic (hand-edited) | Authentic (stolen from a real user) |
| Scope | Entire domain | One service, one host | Entire domain | Entire domain |
| Classic detection | `4769` w/o `4768`; lifetime & crypto anomalies | Host `4624` ↔ DC correlation gap | Privilege-vs-AD mismatch | Privilege-vs-AD mismatch + odd S4U2self |

Golden and Silver forge the ticket's *structure*; Diamond and Sapphire keep the structure genuine and only cheat on *who the ticket says you are*. The next section covers the stealthiest of the four.

---

## Sapphire Ticket: Stealing a Real PAC Instead of Writing One

### The idea

Diamond's even stealthier sibling, and the last gem in the mine. The Diamond ticket still has one weakness: the attacker *hand-edits* the PAC. Writing group memberships by hand is error-prone — a malformed field, a flag that doesn't match how the KDC would really populate it, a structural tell that a careful parser might notice. The PAC is authentic in its *signatures* but synthetic in its *contents*.

The Sapphire ticket removes even that. Instead of writing a privileged PAC, the attacker makes the KDC **hand them a real one belonging to an actual privileged user** — and drops it, untouched, into their ticket.

The trick abuses two legitimate Kerberos delegation extensions:

- **S4U2self** ("Service for User to self") lets a service request a ticket *to itself* on behalf of any user — originally so a front-end app can get a usable ticket for a user who authenticated some other way. Crucially, the ticket it returns contains that user's **full, genuine PAC**, built by the KDC.
- **U2U** ("User to User") lets the result be encrypted to a TGT the attacker already holds, so they can actually decrypt and read it.

Chain them with the stolen krbtgt key and you can ask the KDC, in effect: *"build me a ticket carrying the Domain Admin's real PAC, and encrypt it so I can open it."* Then you re-encrypt that authentic PAC into your own TGT.

```text
Attacker (holds krbtgt key + a valid account)
   |
   |--- S4U2self "on behalf of Administrator" ----> KDC
   |<-- ticket containing Administrator's REAL PAC -   <- genuinely built by the KDC
   |
   | [Decrypt it (U2U / krbtgt key) and lift the authentic PAC out]
   | [Graft that real PAC into a TGT, re-sign with krbtgt key]
   |
   |--- TGS-REQ (ticket w/ authentic admin PAC) ---> KDC
   |<-- TGS-REP: service ticket -------------------
   |
   |--- AP-REQ ------------------------------------> Service: owned
```

Nothing in the PAC is forged. Every field — group list, SIDs, flags, logon info — is exactly what the KDC itself would produce for the real Domain Admin, because the KDC *did* produce it. It's authentic data, misappropriated.

> **Seen in the wild as:** Impacket's `ticketer.py` with the `-impersonate` flag (its Sapphire mode), which drives the S4U2self/U2U dance for you.
{: .prompt-info }

### Why it's so stealthy

It inherits all of Diamond's strengths — real `4768`, perfect krbtgt-signed checksums, authentic timestamps and crypto — and then closes Diamond's last gap:

- **No structural PAC tells.** Byte-for-byte, the PAC is one the KDC would legitimately emit. PAC-structure heuristics that might snag a hand-built Diamond ticket have nothing to grab onto.

### So how *is* it detected?

The same ground-truth principle that beats Diamond still applies — and Sapphire actually leaves one *extra* footprint on the way in:

**1. Privilege-vs-ground-truth mismatch (still the strongest signal).**
The ticket exercises Domain Admin rights; AD records the account as a plain user; no `4728`/`4732`/`4756` group change ever happened. Authentic PAC or not, the privileges appeared without a paper trail. This is the detection no gem-ticket can evade.

**2. The S4U2self request itself.**
Sapphire has to *ask* the KDC for the victim's PAC, and that request is visible — an **S4U2self / U2U exchange** where the account impersonates a high-privilege user it has no delegation rights over. On DCs that log it, a low-priv principal requesting an S4U2self ticket for `Administrator` is a strong, specific anomaly.

**3. Kill the prerequisite — same as Diamond.**
It still needs the krbtgt key. DCSync alerting, `ntds.dit` protection, and the **krbtgt double-rotation** invalidate Sapphire exactly as they do Golden and Diamond.

The takeaway: Sapphire perfects the *contents* of the ticket, but the *context* — AD's record of who is actually privileged, and the delegation request it had to make — still betrays it.

---

## The Defender's Playbook

Detecting forged tickets isn't a single rule — it's a **stack**, because each variant defeats a different layer:

| Layer | Data source | Catches |
|---|---|---|
| AS↔TGS sequence correlation | DC Security log (`4768`/`4769`) | Golden tickets |
| Lifetime / encryption anomaly rules | DC Security log | Lazy forgeries of all kinds |
| DC ↔ host logon correlation | DC log + target host `4624` | Silver tickets |
| Privilege-vs-AD ground truth | Kerberos events + AD group snapshots + `4728`/`4732`/`4756` | Diamond **and Sapphire** tickets |
| Anomalous S4U2self / U2U requests | DC Kerberos log | Sapphire tickets |
| Network PAC inspection | Kerberos traffic to DCs (MDI/NDR) | Golden, Diamond, Sapphire |
| Behavioral baselining (UEBA) | DC + host logs, netflow | Context anomalies, all variants |
| DCSync / `ntds.dit` alerting | DC logs, Sysmon, replication events | The prerequisite theft |
| krbtgt double rotation + account hygiene | Response & hardening | Invalidates every existing forgery |

A few hardening notes that shrink the whole problem space:

- **Rotate krbtgt twice** after any suspected DC compromise — and periodically anyway. This is the single highest-leverage action in this entire post. (Twice because the account keeps the previous password for one generation; a single reset leaves existing tickets valid.)
- **Enforce PAC validation** and keep DCs patched through the November 2021+ Kerberos updates (`PacSignatureValidationLevel` / the [KB5008380](https://support.microsoft.com/en-us/topic/kb5008380) → [KB5020805](https://support.microsoft.com/en-us/topic/kb5020805) hardening line). This meaningfully blunts Silver tickets and some Diamond/Sapphire variants.
- **Protect service and machine account keys** — gMSAs where possible, no weak SPN passwords, and treat machine-account theft as tier-0 adjacent.
- **Use honey accounts** — a fake Domain Admin that *never* legitimately authenticates turns any forged ticket impersonating it into a zero-false-positive alarm.

---

## Conclusion

The forged-ticket family is a tidy illustration of the red/blue arms race in miniature:

- **Golden** forged the artifact → caught by *sequence analysis*.
- **Silver** removed the KDC from the conversation → caught by *host/DC correlation*.
- **Diamond** made the sequence and the cryptography perfect → caught only by *comparing claims against ground truth*.
- **Sapphire** made even the PAC contents authentic → still caught by that same ground truth, plus the delegation request it had to make.

The lesson for defenders: as forgery gets more sophisticated, detection has to move from *"was this ticket issued correctly?"* to *"**does what this ticket claims match what the directory actually knows?**"* Attackers can mint a perfect ticket. They cannot rewrite AD's memory of who is privileged, erase how a host normally behaves, or survive a krbtgt double-rotation.

And the lesson for red teamers: the quieter your ticket, the more your tradecraft has to carry the weight — because on a mature network, the ticket was never the loudest thing you did.

---

*Questions, corrections, or war stories? Find me on [X](https://x.com/alansarijr_) or [LinkedIn](https://www.linkedin.com/in/moelansari/).*
