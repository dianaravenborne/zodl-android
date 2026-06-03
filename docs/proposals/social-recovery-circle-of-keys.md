# Social Recovery for ZODL — Design Proposal

**Status:** Draft for discussion
**Author:** dianaravenborne
**Target product:** ZODL Android (parallel iOS in scope, out of this doc)
**Type:** Product / UX / UI design
**Scope:** Naming, user stories, flows, wireframes, copy. **No implementation yet.**

---

## 1. Why this exists

ZODL's current backup story is the standard one: 24-word seed phrase, write it on paper, hide the paper, hope. That model is solid against malicious adversaries who get the phone, and terrible against the failure modes that actually delete users' funds: house fires, hard drives, paper getting eaten by a baby or a hamster, "I put it somewhere safe" turning into "I have no idea where I put it," and the catastrophic case of self-custody — **the user knew the rules, followed them, and still lost everything because one piece of paper got destroyed.**

We can do better. SLIP-39 splits a seed into N shares such that any T of them recombine to the wallet, and fewer than T leak nothing. The classic deployment is "write shares on metal plates, distribute geographically." This proposal builds an in-app workflow where **the shares live on the phones of people the user trusts** — friends, family, partners — and the user can recover from any T-of-N of them showing up to help.

The hard part isn't the cryptography (SLIP-39 is mature; libraries exist). The hard part is **the social and product layer**: making this feel safe, making the mental model legible, making the share-exchange ceremony feel like a moment of trust rather than a chore, and making sure people understand what they just did to their money.

That's what this document designs.

---

## 2. Naming

I'll lead with three candidates and then explain.

**Recommended: Circle of Keys** (or just "Circle")

Backup candidates considered:
- **Trust Circle**
- **Keepers**
- **Guardians**

Tiered out:
- "Buddy backup" (user's working name) — friendly, but reads as casual. Backup of your wallet sounds like backup of your photos. The user is putting *real money* into someone else's hands; the word should carry weight.
- "Social recovery" — accurate but jargon. Fine as a category descriptor in docs and settings; bad as the product name.
- "Friends backup" / "Family backup" — locks the user into one social relationship model. A user might genuinely want a lawyer + their sibling + a friend.
- "Shamir Backup" (Trezor's term) — accurate, technical, and intimidating. Trezor can use it because their users self-select for technical literacy. ZODL is targeting a broader audience.
- "Recovery Network" — corporate.
- "Safety Net" — close, but evokes "if you fall" — already-failed framing.
- "Lifeline" — same problem, plus medical connotations.
- "Keepers" — solid. The people who hold pieces of your wallet are "Keepers." Verb-able: "Add a Keeper."
- "Guardians" — slightly heavier. Used by Argent and others in account-abstraction recovery. Crowded.
- **"Circle of Keys"** — recommended.
  - Evokes a physical, tangible thing (a circle, a key).
  - "Circle" is the user-facing noun (their **Circle** is the group of trusted people).
  - "Keepers" is the per-person noun ("Add a Keeper to your Circle").
  - "Key" surfaces what's actually being shared without the technical baggage of "share" / "Shamir share."
  - Action verbs are clean: *form your Circle, send a Key to a Keeper, recover your wallet with your Circle.*
  - Marketing-able without being saccharine.
  - Survives the Spanish/Portuguese/German translation test (Círculo, Círculo, Kreis).

For the rest of this doc I'll use "Circle" for the feature, "Keeper" for a person holding a share, "Key" for the share itself.

If "Circle of Keys" is too poetic for the team's taste, the runner-up is **"Trust Circle"** (more literal, less evocative).

---

## 3. User stories

### 3.1 Primary personas

**P1 — Existing user with funds.** Has a ZODL wallet, knows about seed phrases, has been meaning to back up. Heard about a friend who lost ZEC and is finally taking action.

**P2 — New user.** Installing ZODL for the first time. We can offer Circle setup as one of the backup options during onboarding, alongside seed phrase.

**P3 — Keeper.** Doesn't necessarily have a ZODL wallet of their own (yet). Their friend wants to give them a Key. Today's app requires them to install ZODL to accept it. **Design implication:** the install/onboard path for a Keeper-only flow needs to be lightweight; we can't gate Key acceptance behind full wallet creation.

**P4 — Recovering user.** Has lost their phone, has a new device, has their Circle, needs to walk through reconstitution.

### 3.2 User stories

**As an existing ZODL user (P1), I want to**
- Convert my existing BIP-39 wallet into a SLIP-39 Circle without changing my Zcash receive addresses, so my existing balance and history are preserved.
- Choose how many Keepers I want and how many of them have to cooperate to recover.
- See clearly which Keys have been delivered, which are still on my device waiting to go out, and who holds what.
- Hand a Key to a Keeper in person via QR-scan, with confirmation on both sides, and have the Key wiped from my device once delivered.
- Be reminded (gently, persistently) when I have undelivered Keys sitting on my device — those are a liability.
- Replace a Keeper later if they lose their phone, ghost me, or our relationship changes.
- Test that recovery works without actually losing my wallet.

**As a new ZODL user (P2), I want to**
- Be offered Circle backup as a first-class backup option during onboarding, framed clearly relative to seed-phrase backup so I can make an informed choice (or do both).

**As a Keeper (P3), I want to**
- Accept a Key from a friend with a clear explanation of what I'm taking on (I'm not getting their money; I'm holding a fragment that needs T fragments to reconstitute).
- See in my app a list of whose Keys I'm holding, with their nickname.
- Be able to participate in their recovery when asked, with explicit consent.
- Optionally delete a Key I'm holding (with strong friction and warning).
- Not be a single point of failure (the math guarantees this; the UI should reinforce it).

**As a recovering user (P4), I want to**
- Start a recovery on a new device.
- Contact my Keepers via the app or out-of-band (call, text), have them open ZODL, and approve sending their Key back to me.
- See my progress (X of T Keys collected) and know how many more I need.
- Have the recovery either complete (wallet restored) or fail clearly with what to do next.

### 3.3 Non-goals (this proposal)

- Multisig (this is recovery of a single-signature wallet; key-share custody is different from spending-share custody).
- Time-locked recovery / dead-man switches.
- Inheritance flows (recovery after the user's death). These are real and important and out of scope for v1; the Circle data model should not preclude them in v2.
- Recovery via untrusted third-party (custodian-assisted recovery).
- Reissuing a Key to the same Keeper without re-running the threshold ceremony (potential v2; raises subtle threat-model questions).

---

## 4. The mental model we're teaching

This is the part that decides whether the feature works or not. The user has to leave the flow understanding three things:

1. **Each Keeper holds a piece, not a copy.** No single Keeper can spend my money.
2. **It takes T Keepers cooperating to recover, never T-1.** If T=3 and N=5, then any 3 of the 5 can rebuild the wallet; any 2 of them have nothing.
3. **If too many of my Keepers' phones die, I can't recover.** N-T is the slack. T=3 of 5 means I can lose up to 2 Keepers (40%) and still recover; if I lose 3 of 5, I'm done.

The product surface for this is two things: the **threshold picker** (§5.3), and the **Circle health screen** (§5.10).

**Mental model anti-pattern to avoid:** never use the word "share" in user-facing copy. Engineers know what a share is; users hear "share" and think "social media share, send to all." Use "Key" or "Key piece." When we have to be precise (settings → advanced), we can say "SLIP-39 share" once in a tooltip and move on.

---

## 5. Screen-by-screen design

I'll wireframe each screen as ASCII (good enough to communicate layout to engineering; the actual Compose work will follow). Each screen lists: purpose, entry point, primary CTA, secondary actions, key copy, edge cases.

### 5.1 Entry points

- **Onboarding** (P2): final backup step adds "Circle of Keys (recommended)" alongside "Seed phrase." Default to Circle; surface seed as alternative.
- **Settings → Backup & Recovery** (P1): existing path. Add "Circle of Keys" as a top-level item alongside today's seed-phrase backup.
- **Home banner** (P1): if the user has chosen Circle but has undelivered Keys, persistent banner *"You have 3 Keys still on this device. Meet up with your Keepers."*
- **Notifications**: weekly reminder while undelivered Keys exist on the device.

### 5.2 Circle landing screen

```
┌───────────────────────────────────────────────┐
│ ←  Circle of Keys                         ⋮  │
├───────────────────────────────────────────────┤
│                                               │
│        ◯ ◯ ◯ ◯ ◯                              │
│        ━━━━━━━━━━ (visualization of circle)   │
│                                               │
│   Your Circle is the group of people who      │
│   can help you recover your ZODL wallet if    │
│   you ever lose access to it.                 │
│                                               │
│   No single Keeper can spend your funds.      │
│   It takes a group to recover your wallet.    │
│                                               │
│                                               │
│   ┌─────────────────────────────────────┐    │
│   │     Set up your Circle              │    │
│   └─────────────────────────────────────┘    │
│                                               │
│   I'm holding Keys for someone else  ›        │
│   Recover a wallet using my Circle   ›        │
│                                               │
└───────────────────────────────────────────────┘
```

- **Primary CTA**: Set up your Circle.
- **Secondary actions**:
  - I'm holding Keys for someone else → entry for P3.
  - Recover a wallet using my Circle → entry for P4.
- **Why three CTAs from one screen?** Because P1, P3, P4 each arrive here from different mental contexts. Stacking them avoids hiding the Keeper/Recovery flows behind a hamburger.

### 5.3 Threshold picker — "How many Keepers?"

This is the load-bearing screen for the mental model.

```
┌───────────────────────────────────────────────┐
│ ←  How many Keepers?                       ⋮  │
├───────────────────────────────────────────────┤
│                                               │
│   Choose a Circle size                        │
│                                               │
│   ┌─────────────────────────────────────┐    │
│   │  ●  3 of 5  (recommended)           │    │
│   │     Best balance. Tolerates 2       │    │
│   │     Keepers being unavailable.      │    │
│   ├─────────────────────────────────────┤    │
│   │  ○  2 of 3  (simpler)               │    │
│   │     Easier to gather. Tolerates     │    │
│   │     1 unavailable.                  │    │
│   ├─────────────────────────────────────┤    │
│   │  ○  4 of 7  (resilient)             │    │
│   │     For larger trusted networks.    │    │
│   │     Tolerates 3 unavailable.        │    │
│   ├─────────────────────────────────────┤    │
│   │  ○  Custom...                       │    │
│   └─────────────────────────────────────┘    │
│                                               │
│   ┌─────────────────────────────────────┐    │
│   │     Continue                         │    │
│   └─────────────────────────────────────┘    │
│                                               │
└───────────────────────────────────────────────┘
```

- **Three presets + custom.** Presets cover the common case; custom is gated behind a tap for the power user.
- **Copy is explicit about tradeoff.** "Tolerates 2 Keepers being unavailable" is the right framing — what the user gains from N-T slack.
- **Custom screen** shows a live preview: two segmented sliders ("How many people in your Circle?" 2–9, "How many to recover?" 1–N), with the same human-readable text underneath ("Tolerates X unavailable").
- **Hard floor**: T≥2, N≥3, T≤N. Reject T=1 (defeats the purpose) and N=2 with T=2 (no slack, worse than just two paper backups).

### 5.4 Naming your Circle (and identifying your Keepers)

```
┌───────────────────────────────────────────────┐
│ ←  Add your Keepers                       ⋮  │
├───────────────────────────────────────────────┤
│                                               │
│   Add 5 Keepers to your Circle.               │
│                                               │
│   Give each one a name so you remember who    │
│   holds which Key. They'll see this name too. │
│                                               │
│   ┌─────────────────────────────────────┐    │
│   │ Keeper 1                            │    │
│   │ [ Mom                             ] │    │
│   ├─────────────────────────────────────┤    │
│   │ Keeper 2                            │    │
│   │ [ Sam                             ] │    │
│   ├─────────────────────────────────────┤    │
│   │ Keeper 3                            │    │
│   │ [ Jay                             ] │    │
│   ├─────────────────────────────────────┤    │
│   │ Keeper 4                            │    │
│   │ [ Lawyer                          ] │    │
│   ├─────────────────────────────────────┤    │
│   │ Keeper 5                            │    │
│   │ [ Safety deposit box (paper)      ] │    │
│   └─────────────────────────────────────┘    │
│                                               │
│   ┌─────────────────────────────────────┐    │
│   │     Generate Keys                    │    │
│   └─────────────────────────────────────┘    │
│                                               │
└───────────────────────────────────────────────┘
```

- Pre-populate from existing address book if the user has nicknamed contacts; offer suggest-as-you-type.
- "Safety deposit box (paper)" — important: one of the slots can be **a paper backup the user keeps themselves**. That's the user being their own Keeper. Show it in §5.5 with a different icon and a paper-export action (QR + printable text).
- Names are local + sent with the Key. The Keeper sees "You're holding a Key for [user's display name]" when accepting (§5.7).

### 5.5 Keys generated — the "go meet your Keepers" state

```
┌───────────────────────────────────────────────┐
│ ←  Your Circle is ready                   ⋮  │
├───────────────────────────────────────────────┤
│                                               │
│   ╭───────────────────────────────────╮      │
│   │  5 Keys created.                   │     │
│   │  Meet up with each Keeper to       │     │
│   │  hand them their Key.              │     │
│   ╰───────────────────────────────────╯      │
│                                               │
│   Mom              ● Key on device   Send  ›  │
│   Sam              ● Key on device   Send  ›  │
│   Jay              ● Key on device   Send  ›  │
│   Lawyer           ● Key on device   Send  ›  │
│   Safety deposit   ● Key on device   Save  ›  │
│                                               │
│                                               │
│   ⚠  These Keys are sitting on this device.   │
│      Until you deliver them, your Circle is   │
│      not protecting anything.                 │
│                                               │
│                                               │
│   Why not just print all five and hide them?  │
│                                            ›  │
└───────────────────────────────────────────────┘
```

- **Status dots** make undelivered visible at a glance: orange "on device," green "delivered."
- Tapping "Send →" opens §5.6 (delivery flow).
- "Save →" for the paper slot opens an export sheet (printable PDF + QR).
- **Recurring warning**: until all N (or N-1) Keys are off the device, the device itself is a single point of failure — same security posture as a regular seed-on-phone setup. The home-screen banner reinforces this until delivery is complete.
- **"Why not just print all five"** is a tooltip-link, not a hidden FAQ. Anticipated user objection; answering it inline reduces support load and builds confidence.

### 5.6 Delivery ceremony (in-person QR exchange)

Two-screen split: the giver's flow and the Keeper's flow, designed to happen at the same physical table.

#### Giver's view (P1)

```
┌───────────────────────────────────────────────┐
│ ←  Sending Mom's Key                      ⋮  │
├───────────────────────────────────────────────┤
│                                               │
│   Have Mom open ZODL and tap                  │
│   "I'm holding Keys for someone else."        │
│                                               │
│   When she's ready, scan her QR code.         │
│                                               │
│   ┌─────────────────────────────────────┐    │
│   │                                     │    │
│   │       [ camera viewfinder ]         │    │
│   │                                     │    │
│   └─────────────────────────────────────┘    │
│                                               │
│   Or share via secure messenger              ›│
│                                               │
└───────────────────────────────────────────────┘
```

Once the giver scans the Keeper's "I'm ready" QR (which carries an ephemeral pubkey for envelope encryption — see §6), the screen flips:

```
┌───────────────────────────────────────────────┐
│   Show this to Mom                            │
│                                               │
│   ┌─────────────────────────────────────┐    │
│   │                                     │    │
│   │     [ QR with encrypted Key ]       │    │
│   │                                     │    │
│   └─────────────────────────────────────┘    │
│                                               │
│   Waiting for Mom to confirm…  ●○○            │
│                                               │
│   ✓  Mom confirmed receipt.                   │
│      This Key has been removed from your      │
│      device.                                  │
│                                               │
│   ┌─────────────────────────────────────┐    │
│   │     Done                             │    │
│   └─────────────────────────────────────┘    │
└───────────────────────────────────────────────┘
```

- **Two-way handshake.** Giver scans Keeper's request QR → Giver displays encrypted Key QR → Keeper scans it → Keeper computes an ack hash → Keeper displays ack QR → Giver scans ack → Giver wipes the Key locally.
- **Why two QRs each way**: prevents shoulder-surfing (an encrypted Key QR captured by a bystander camera is useless without the Keeper's ephemeral private key) AND gives the giver cryptographic proof of delivery before wiping. If the ack QR is never scanned, the Key stays on the device.
- **"Or share via secure messenger"** is the fallback for remote handoff (Signal, etc.). De-emphasized but present — physical meetings aren't always possible. Same crypto envelope; just a different transport. Important UX caveat in §6.3.
- **Confirmation copy** is explicit: "removed from your device." We don't leave room for the user to wonder whether a copy persists.

#### Keeper's view (P3) — the same ceremony from the other side

```
┌───────────────────────────────────────────────┐
│ ←  Hold a Key for a friend                ⋮  │
├───────────────────────────────────────────────┤
│                                               │
│   Diana wants you to hold a Key for her       │
│   wallet recovery.                            │
│                                               │
│   What this means:                            │
│   • You're not getting Diana's money.         │
│   • You'll hold a piece of her recovery.      │
│   • If she needs to recover, she'll ask you   │
│     to share this Key back.                   │
│   • She'll need pieces from several friends,  │
│     not just yours.                           │
│                                               │
│   ┌─────────────────────────────────────┐    │
│   │     I understand — show my QR        │    │
│   └─────────────────────────────────────┘    │
│                                               │
│   Decline  ›                                  │
│                                               │
└───────────────────────────────────────────────┘
```

- **Educates first, asks second.** The Keeper might not have any context. Even if they do, the explicit list re-grounds them.
- After tapping "I understand," they get a "show this QR to Diana" screen with their ephemeral pubkey. After receiving the encrypted Key, the app confirms reception and shows the ack QR for Diana to scan.

### 5.7 Keeper's wallet view — "I'm holding"

```
┌───────────────────────────────────────────────┐
│ ←  Keys you're holding                    ⋮  │
├───────────────────────────────────────────────┤
│                                               │
│   You're holding 3 Keys for friends.          │
│                                               │
│   Diana                                     ›│
│   added today, in person                      │
│                                               │
│   Alex                                       ›│
│   added 4 months ago                          │
│                                               │
│   Jordan                                     ›│
│   added 1 year ago                            │
│                                               │
│                                               │
│   Holding a Key never costs you anything      │
│   and you can never spend their money. You    │
│   can delete a Key, but they'll need to ask   │
│   you to replace it before they're at risk.   │
│                                               │
└───────────────────────────────────────────────┘
```

Tap a row → detail screen with delete (friction: confirm twice), "respond to recovery request" (becomes active when the owner initiates recovery), and metadata (when added, in-person vs remote, owner display name).

### 5.8 Home banner (P1, undelivered Keys outstanding)

```
┌───────────────────────────────────────────────┐
│  ⚠  3 Keys still on this device.              │
│     Until they're delivered, your Circle      │
│     isn't protecting anything.                │
│                              Continue  ›      │
└───────────────────────────────────────────────┘
```

Persistent banner on home until either (a) all Keys delivered, or (b) user explicitly dismisses via "I changed my mind, just give me a seed phrase" path (which we offer for honesty and safety; see §7.1).

### 5.9 Recovery flow (P4)

```
┌───────────────────────────────────────────────┐
│ ←  Recover your wallet                    ⋮  │
├───────────────────────────────────────────────┤
│                                               │
│   You'll need to gather Keys from your        │
│   Circle. Reach out to your Keepers and       │
│   ask them to send their Keys back.           │
│                                               │
│   You need 3 of your 5 Keepers.               │
│                                               │
│   Keys collected:    0 of 3                   │
│   ▓░░                                         │
│                                               │
│   ┌─────────────────────────────────────┐    │
│   │     Scan a Key from a Keeper         │    │
│   └─────────────────────────────────────┘    │
│                                               │
│   Receive via secure messenger      ›         │
│                                               │
│   How do I contact my Keepers?      ›         │
│                                               │
└───────────────────────────────────────────────┘
```

Then after one Key collected:

```
│   You need 3 of your 5 Keepers.               │
│                                               │
│   Keys collected:    1 of 3                   │
│   ▓▓░                                         │
│                                               │
│   ✓ Mom                                       │
│                                               │
│   ┌─────────────────────────────────────┐    │
│   │     Scan another Key                 │    │
│   └─────────────────────────────────────┘    │
```

After T collected, the wallet rebuilds automatically and we land on the success screen (mirror of regular onboarding success).

- **Recovery does not require the original device.** The user can recover on any fresh ZODL install.
- **Recovery does not consume the Keepers' Keys.** The Keepers retain their Keys after the recovery completes; we just transmit (not move). This is a deliberate UX choice — it means the user can re-recover later (e.g. if the first recovery device is also lost) without re-running the ceremony.
- **However:** the user is strongly prompted at recovery success to **re-form their Circle with fresh Keys** because the act of recovery transmitted shares across new transport, and even with envelope encryption, conservative users may want fresh shares. Surface this as a recommendation, not a requirement.

### 5.10 Circle health (settings → Circle)

```
┌───────────────────────────────────────────────┐
│ ←  Your Circle                            ⋮  │
├───────────────────────────────────────────────┤
│                                               │
│      ◯ ◯ ◯ ◯ ◯                                │
│      ─────────                                │
│   3 of 5 needed to recover                    │
│                                               │
│   ✓  Mom               delivered Mar 2026     │
│   ✓  Sam               delivered Mar 2026     │
│   ✓  Jay               delivered Mar 2026     │
│   ⚠  Lawyer            on device              │
│   ✓  Safety deposit    saved                  │
│                                               │
│   Test recovery                            ›  │
│   Replace a Keeper                         ›  │
│   Change Circle size...                    ›  │
│                                               │
│   ┌─────────────────────────────────────┐    │
│   │     Dissolve Circle                  │    │
│   └─────────────────────────────────────┘    │
│                                               │
└───────────────────────────────────────────────┘
```

- **Test recovery** is critical. It's a tap that simulates pulling shares from T Keepers (each Keeper gets a notification, approves, sends ack-only — not the actual Key). Tests the social graph without spending the trust. We absolutely need this; if the first time a user actually runs recovery is the real recovery, they will have a bad time.
- **Replace a Keeper** triggers a re-share ceremony for that one slot (cryptographically nontrivial — see §6.4 — but UX is "scan with new Keeper, ack old Keeper's Key invalidated").
- **Change Circle size** is a full re-share with new shares. Big-friction.
- **Dissolve Circle** is "this isn't for me anymore." Drops all on-device Keys, sends a (best-effort) notification to all Keepers telling them their Keys are now orphaned. Notification is courteous, not load-bearing — the security model doesn't depend on Keepers complying.

### 5.11 Onboarding integration (P2)

After seed-phrase introduction, before the standard "Back up your seed":

```
┌───────────────────────────────────────────────┐
│   How do you want to back up your wallet?     │
│                                               │
│   ┌─────────────────────────────────────┐    │
│   │  ⊙  Circle of Keys (recommended)    │    │
│   │     Distribute your backup across   │    │
│   │     trusted friends and family.     │    │
│   │     No single piece can spend your  │    │
│   │     money.                          │    │
│   │     Setup: ~5 min + meeting your    │    │
│   │     Keepers later.                  │    │
│   ├─────────────────────────────────────┤    │
│   │  ○  Seed phrase                     │    │
│   │     24 words you write down and     │    │
│   │     keep safe. Tried and true, but  │    │
│   │     if you lose it, you're done.    │    │
│   │     Setup: ~2 min.                  │    │
│   ├─────────────────────────────────────┤    │
│   │  ○  Both                            │    │
│   │     Belt and suspenders.            │    │
│   │     We recommend this if you have   │    │
│   │     a meaningful amount of ZEC.     │    │
│   └─────────────────────────────────────┘    │
│                                               │
│   ┌─────────────────────────────────────┐    │
│   │     Continue                         │    │
│   └─────────────────────────────────────┘    │
└───────────────────────────────────────────────┘
```

- **"Both" is the third option.** This is the honest answer for most users. A seed phrase in a safe + a Circle of three people = two genuinely-independent recovery paths.
- **Defaulting to Circle** for the highlight is a product bet: we believe the Circle UX produces better real-world recovery outcomes for typical users. The seed-phrase option is not hidden; it's clearly labeled as the alternative.

---

## 6. Threat model and the cryptography around it (UX-relevant only)

This proposal is design, not crypto, but the design decisions are informed by the security model. Engineering will spec the protocol; this section flags the UX-load-bearing items.

### 6.1 Envelope encryption around each share

When the giver sends a Key, the Keeper's ephemeral X25519 pubkey is what the giver scans first (Keeper's "I'm ready" QR). The giver encrypts the SLIP-39 share to that pubkey + a freshly-generated one of their own (NaCl box / age recipient). The encrypted blob is what goes in the second QR.

**Why this matters for UX:** the displayed QR codes are not raw shares. A bystander photographing the giver's QR can't reconstruct the share without the Keeper's ephemeral private key (which never leaves the Keeper's phone). This lets us tell the user "it's safe to do this in a busy café" — which we should say in the help copy, because otherwise users will avoid using the feature in any public setting and the friction will kill adoption.

### 6.2 Ack proves receipt, not retention

The Keeper's ack QR contains a MAC over the encrypted share they received, signed with their long-term recovery pubkey. This proves the Keeper received the share. It does **not** prove they kept it (they could delete it the next minute). The UX says "Keeper confirmed receipt" — not "Keeper has safely stored your Key forever." Subtle wording difference; do not get this wrong.

### 6.3 Remote handoff caveat

"Send via secure messenger" works for the same envelope crypto, but breaks the in-person assumption. The threat is that the Keeper's secure messenger account is compromised at exchange time. We should:

- Surface this caveat in the remote-handoff flow.
- Strongly prefer in-person.
- Allow it but require an extra confirmation tap ("I trust the messenger I'm about to use").

### 6.4 Replace-a-Keeper

This is the hardest cryptography in the proposal. Naively, replacing one share requires re-sharing the entire secret (because share sets are not composable). UX-wise we should:

- v1: replacement is a full Circle re-share. Friction is high but model is honest.
- v2 (research): proactive secret sharing schemes that allow share refresh without full reconstruction. Out of scope for v1 docs.

The UI for v1 should be honest about this — "Replacing a Keeper will create new Keys for all your Keepers. You'll need to meet with each of them again." Not great. But it's the truth.

### 6.5 Owner identity to Keeper

Today's wireframe shows "Diana wants you to hold a Key" without any cryptographic basis for "this is actually from Diana." This is fine because the handshake is in-person — the Keeper sees Diana's face. For remote handoff, the giver's identity should be tied to their Zcash address (display: "Diana — z1abc...xyz"); the Keeper accepts based on that.

**UX implication:** we never claim to verify identity. We claim to verify the receipt. Identity is the user's job.

---

## 7. Honest discussion of failure modes

A good product surface acknowledges what can go wrong. These are the items I'd surface either in help docs or as inline tooltips.

### 7.1 "I changed my mind"

Some users will start a Circle setup, get cold feet, and want out. The UI must support this gracefully:

- Before any Key is delivered: trivial. Just dissolve.
- After some Keys are delivered: dissolve is best-effort (Keepers are notified that their Key is now orphaned; the Key is useless without the others). The user falls back to either (a) a fresh seed phrase or (b) the original BIP-39 if they opted into "Both."
- **Critical**: never block "I want a regular seed phrase instead" behind Circle completion. If the user wants a paper backup, give them a paper backup. Don't trap them.

### 7.2 "My Keepers are unreachable"

Real failure mode: T=3 of 5, two Keepers have died, one is in a coma, one ghosted me, one is my mom who answers. Recovery is impossible.

This is the correct outcome of self-custody with a too-thin Circle — but the product should help users avoid getting here. The threshold picker should bias toward generous N-T slack (recommended preset is 3-of-5, which tolerates 40% loss). Settings should periodically remind the user to test recovery (§5.10).

### 7.3 "A Keeper's phone is stolen"

If a Keeper's phone is compromised, one share leaks. Below threshold, this leaks nothing about the seed. The product should reassure the user that this is fine; the product should ALSO prompt the user to consider replacing that Keeper (which is high friction; see §6.4) if they're worried.

### 7.4 "All my Keepers are at the same dinner party and the building collapses"

Geographic distribution is a UX recommendation we should surface in the help copy. We can't enforce it, but a tip at Circle setup time — "consider including Keepers in different cities" — is good practice and free.

### 7.5 "My Keeper is malicious and colludes with T-1 other Keepers"

Below the cryptographic threshold there's nothing to steal. At or above the threshold, the user has chosen the wrong Keepers; the product can't help.

### 7.6 "I forget who my Keepers are"

The Keeper names live on the user's device. If the user loses the device, the Keeper list is gone. **Mitigation:** include the Keeper name (or a hint) inside the recovery flow's first prompt — but the Keepers themselves know they hold a Key for the user. The recovery initiative can come from either side: the user asks the Keepers, or a Keeper sees a recovery request notification.

### 7.7 "What if SLIP-39 has a vulnerability discovered later?"

Honest answer: the user is exposed. Same risk as any seed format. Surface in advanced settings; don't bury, don't dwell.

---

## 8. Phasing

### Phase 0 — Design lock (this document)
- Naming agreement
- User flows reviewed
- Threat model reviewed
- Wireframes reviewed
- Eng spike: confirm SLIP-39 library availability for Kotlin + iOS

### Phase 1 — MVP: in-person ceremony only
- Onboarding integration (P2)
- Circle creation (5.2–5.5)
- In-person QR handoff (5.6, both sides)
- Keeper holding screen (5.7)
- Home banner (5.8)
- Recovery flow (5.9)
- Circle health (5.10) minus "Test recovery" and "Replace a Keeper"

### Phase 2 — Maturity
- Test recovery (essential; phase 1.5 if eng capacity allows)
- Replace a Keeper (the painful version: full re-share)
- Remote handoff via secure messenger (with caveats)
- Convert-existing-BIP-39 path (P1's primary use case if they already have a wallet)

### Phase 3 — Advanced
- Proactive share refresh (cryptography research)
- Inheritance flow
- Keeper directory across organizations (e.g. "ZODL Trusted Keeper Network" — partnered Keepers who hold shares for a fee, like a custodial backup)

---

## 9. Open questions for the team

These are the decisions I need from product/eng/security before this becomes implementation work:

1. **Naming.** "Circle of Keys" vs "Trust Circle" vs "Keepers" vs other. I have an opinion but don't insist.
2. **Default threshold.** I've recommended 3-of-5; some teams prefer 2-of-3 for adoption reasons (lower friction). Tradeoff is real. **My take: 3-of-5 default, present 2-of-3 prominently as "simpler.**"
3. **Onboarding default.** Do we default new users to Circle, to seed phrase, or to "Both"? **My take: Circle default, with "Both" as the strongly-suggested option for users who indicate non-trivial holdings.**
4. **Keeper-only install path.** Do we let a Keeper install ZODL with no wallet of their own and use the app purely as a Keeper? **My take: yes. The friction of "you must create a wallet to hold a friend's Key" is a deal-breaker.** Design implication: a "Keeper-only" install mode.
5. **Remote handoff in MVP.** Include or defer? **My take: defer to phase 2. In-person only for v1 keeps the trust model clean.**
6. **Replace-a-Keeper friction.** v1 is full re-share; the UX is painful. Do we want this in MVP at all, or hide it behind "advanced settings → reform Circle"? **My take: hide in MVP; explicit "Replace a Keeper" surfaces in phase 2.**
7. **Convert existing BIP-39 to SLIP-39.** This is the P1 primary use case (existing users with funds). The SLIP-39 spec defines BIP-39 → SLIP-39 conversion but with subtle properties (passphrase handling, account model). Does the SDK support this? **Need engineering spike before promising this in MVP.**
8. **Backwards compatibility.** If a user creates a Circle, can they ever go back to plain seed phrase later without dissolving? **My take: yes — "export as seed phrase" should always be available in advanced settings, with strong friction. Self-custody means user always retains escape hatch.**
9. **Multi-wallet users.** ZODL supports multiple wallets per install. Is each wallet's Circle independent? **My take: yes, each wallet has its own Circle. Sharing one Circle across wallets is conceptually possible but creates correlation surface.**
10. **iOS parity.** This proposal is Android-focused. Does iOS ship at parity? **My take: yes. The whole feature degrades if cross-platform handoff doesn't work (e.g., my Keeper Mom has iOS, I have Android). The protocol must be platform-neutral.**

---

## 10. Acknowledgments

This proposal stands on the shoulders of:

- **SLIP-39** (Andrew Kozlik et al., SatoshiLabs) — the cryptographic foundation.
- **Trezor Shamir Backup** — the proof that the model can ship in a consumer wallet.
- **Argent / Soul Wallet account abstraction recovery** — the inspiration for social recovery as a default rather than a backup option.
- **Casa multikey** — the reminder that recovery UX is the actual product, the cryptography is the table stakes.

Mistakes in framing or threat-model exposition are mine.

---

## Appendix A — Copy bible

Words we use:
- **Circle** (the feature, the group)
- **Keeper** (a trusted person holding a Key)
- **Key** (one SLIP-39 share, in user-facing language)
- **Recover** (the action of rebuilding a wallet from Keys)
- **Form your Circle** (the setup action)
- **Send a Key to a Keeper** (the delivery action)
- **Hold a Key for someone** (the Keeper's action)

Words we avoid in user-facing copy:
- **Share** (already overloaded; means social-media-share to users)
- **Shamir** (technical jargon; surface only in advanced/tooltip)
- **SLIP-39** (technical jargon; surface only in advanced/tooltip)
- **Threshold** (abstract; we say "how many Keepers need to cooperate")
- **Secret** (overdramatic and imprecise)
- **Backup** (we say "recovery" — distinguishes from the failure-implying-it-already-failed framing)

---

## Appendix B — Visual concepts I want a designer to take from here

This document uses ASCII to communicate layout. The brand-aligned visual work I'd push for:

- **The Circle as the dominant visual metaphor.** A literal circle of avatars / dots / silhouettes. Each Keeper is a position on the circle. Delivered = filled. Undelivered = outline. The visual itself teaches the model.
- **Color signal for share state.** Orange = on-device-undelivered (liability), green = delivered, gray = paper.
- **Iconography for "in-person ceremony"** — two phones tipping toward each other, QR-codes-meeting. We want this to feel like a moment, not a transaction.
- **Avoid lock/vault iconography** — every wallet uses it; the Circle metaphor is more distinctive.
