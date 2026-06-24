# War Room — Data Model & Security Design

This is the authoritative spec for how client data is structured, classified, and
encrypted in the War Room app. The guiding rule: **a client's name and their
financial details never sit together anywhere that can be read off-device.**

## 1. Sensitivity ceiling

The app holds **nothing more personal than what already lives in the `1Remarkable`
Drive folder**: a name, dollar amounts of accounts, and structured "closing
signals." No SSN, no account numbers, no address, no DOB, no contact info — those
never enter the system.

## 2. Three tiers

```
Identity (device only):   J-01 -> "Josh"                 <- only the name
Numbers  (encrypted):     custodians + $ values          <- financial, de-named
Signals  (encrypted):     goals / objections / drivers   <- analyzable, generalized
```

| Tier | Contents | Key | Syncable? |
|------|----------|-----|-----------|
| 0 Identity | code -> name | `MK_id`  | **No — device only** |
| 1 Intel | accounts, dollars, income streams, fees | `MK_intel` | Yes (ciphertext only) |
| 1 Intel | goals, objections, drivers, flags, script inputs | `MK_intel` | Yes (ciphertext only) |

If the encrypted tier ever leaked, an attacker sees `J-01 · $72K at Empower ·
$100K crypto` with no person attached. Re-attaching the name requires `MK_id`,
which never leaves the device.

## 3. Free-text -> structured "closing signals"

Raw call narrative is the re-identifying part, so it is **generalized into
structured fields** on the way in — keep the selling intel, drop the fingerprint.

| Keep (drives the close) | Generalize / drop (identifies the person) |
|---|---|
| Life stage, goals, objections | exact dates ("retired last week" -> "recently retired") |
| "concentrated single-stock position" | the ticker |
| "rental real estate" | "4 rentals + primary, $15K/mo" |
| custodian + dollar amount | — (allowed) |

## 4. Key hierarchy

```
passphrase --PBKDF2(SHA-256, 310k)--> KEK (wrapping key, never stored)
                                        |
                  +---------------------+---------------------+
                  v                                           v
            MK_id (identity)                            MK_intel (intel)
            wrapped by KEK, device-only                 wrapped by KEK, syncable
                  |                                           |
            AES-256-GCM per record                     AES-256-GCM per record
            (random 96-bit IV each write)              (random 96-bit IV each write)
```

- **At rest:** AES-256-GCM. Every write uses a fresh random IV; the GCM auth tag
  gives tamper detection for free.
- **Keys:** derived on-device from the passphrase. The master keys are wrapped by
  the KEK and stored; the KEK itself is never persisted.
- **Upgrade path:** swap PBKDF2 for Argon2id, and add a per-record DEK wrapped by
  the master key (envelope encryption) when sync is added.

## 5. Record schemas

```ts
// Tier 0 — device only, encrypted under MK_id
interface IdentityEntry { code: string; name: string; }

// Tier 1 — encrypted under MK_intel, keyed by code, NEVER a name
interface IntelRecord {
  code: string;
  // header / status
  stage: "Inbound" | "Discovery" | "Proposal" | "Decision";
  priority: "CRITICAL" | "HOT" | "WARM" | "PIPELINE" | "INTRO";
  callType: string;                 // "Follow-Up Review", "WSS", ...
  confidence: number;               // 0-100
  schedule?: { day: string; time: string; tz?: string };
  // numbers
  moveable: string;                 // "$122K-$233K+"  (outside/non-anchor)
  accounts: { custodian: string; value: string; tag: string; anchor?: boolean }[];
  incomeStreams?: { label: string; amount: string; note?: string }[];
  household?: { incomeEst?: string; magiEst?: string; retirementGoal?: string };
  fee?: { from?: string; to?: string; savings?: string };
  // signals
  goals?: string[];
  motivation?: string[];
  objections?: { objection: string; response: string }[];
  complications?: string[];
  specialFlags?: string[];          // "Crypto concentration", "Contractor income"
  decisionDrivers?: string[];
  product?: string;
  productMatch?: { product: string; rationale: string; fit?: string }[];
  solutionOffered?: string;
  closeAction?: string;
  // prep-sheet narrative (generated or stored)
  introScript?: string;
  whatToAsk?: string[];
  pitchFlow?: { title: string; desc: string }[];
  answerKey?: string[];
  callHistory?: { date: string; callType: string; summary: string; leftOff: string }[];
  sourceModifiedTime?: string;
}
```

## 6. Pipeline

```
Profile.md  ->  IntelRecord (name split out)  ->  generates  ->  [Data sheet] + [On-Call Helper]
                name in Tier 0; rest encrypted                   Remarkable landscape PDF
```

The two prep sheets are produced **on demand** from one `IntelRecord` + the Tier-0
name lookup. The join happens only in memory, on-device, at render time.

## 7. Prep Sheet cards (Claude-authored, app-rendered)

A profile may carry a `## Prep Sheet` (or `## Cheat Sheet`) section — the rich,
visual "cheat sheet" cards that show up as click-in widgets at the top of Client
Intel. The **content** is authored by Claude (depth/analysis); the **app**
parses it deterministically and renders the styled widgets. Format:

```
## Prep Sheet
### <Title> (table)
- Label: Value
- Annual savings: ~$1,600/yr        # a row whose label has "saving/delta" → green highlight
- Total / net ... : ...             # label with "total/net/annual drag" → gold total row

### <Title> (cashflow)
- Household income: ~$14K/mo
- Primary consolidation opportunity: $350,000 rollover   # label with "opportunity/consolidat/primary" → highlighted box

### <Title> (flags)
- Flag title :: One-paragraph explanation of the planning flag.

### <Title> (angles)
- Angle title :: One-paragraph strategy angle / how to play it.
```

Rules:
- Card **type** is the parenthetical `(table|cashflow|flags|angles)` at the end of
  the `###` title. If omitted, it's inferred from the title text
  (fee→table, cash/income→cashflow, flag→flags, strateg/angle→angles).
- `table`/`cashflow` cards use `- Label: Value` rows (first colon splits).
- `flags`/`angles` cards use `- Title :: paragraph` items (` :: ` separator).
- **Fidelity:** author only from verified notes. Don't assert a fee/savings figure
  that hasn't been confirmed — label it illustrative and prompt to confirm (see the
  Brandon Chambers Fee Delta card for the pattern).
- The app skips any empty card and omits the whole group if there's no Prep Sheet.

## 8. Authoring profiles from call notes — capture EVERYTHING

When a call (or a photo of call notes) is turned into a profile, the **governing
rule** is: every fact in the source notes goes into the profile, in the section
the app reads it from. Don't summarize detail away — the app pulls different
fields at different moments (a fee on the proposal call, the family/why on the
opening, SS/pension during planning), so anything dropped is invisible forever.
Fidelity still holds: capture what's actually said, never invent.

### Canonical profile template

```
# <Full Name> — Profile
Updated: <Mon D, YYYY>

## Snapshot
- Stage: Inbound | Discovery | Proposal | Decision | Enrollment | Follow-up
- Outside / moveable assets: ~$<total> (<one-line make-up>)
- Accounts:
 - <Custodian> — <$value> — <tag/role>; anchor accounts already at Empower say "anchor"
- Product: <Premier IRA | Personal Strategy | …>
- Confidence: <0–100>
- Solution offered: <the recommendation in one sentence>
- Why consolidate: <reason; reason; reason>

## Personal / Relationship
- Family: marital status, spouse (name/age/work), kids (count/ages/where), parents/
  dependents, any caregiving — the layout and dynamics.
- Life & retirement (the why): hobbies, travel, downsizing/relocating, what they
  want to be DOING in retirement, lifestyle goals, legacy wishes.
- Spending: monthly/annual spend figures, debts.
- Social Security / pension: estimates and timing (e.g. "$1,288 SS; $1,700 pension at 67").
- Health / other: Medicare, LTC, anything else from the notes.

## Objection & Response
- Objection: <their hesitation, verbatim where possible>
- Response: <how to reframe it>

## Opening               ← optional but ideal: a VERBATIM, second-person opening to read aloud
<A real personal question grounded in the notes.>

<Recap of where things stand, in their words — what matters most to them.>

<What today is about + drive to the next step. Each blank-line paragraph = one spoken beat.>

## Prep Sheet            ← optional but ideal for proposal/decision calls (see §7)
### <Title> (table|cashflow|flags|angles)
- …

## Close Action / Next Step
Next call: <Type> call — <date + time>. <what the next call is for; Watch: …>
# If nothing is rebooked (a pending decision they'll "get back to you" on), say so
# explicitly: lead with "No call scheduled" (or "Next call: none | TBD | not scheduled").
# The app reads that as authoritative and CLEARS any stale in-app next-call override on
# sync, so the client drops into Pending Decisions instead of lingering in Upcoming.

## Tasks                 ← only real to-dos you owe them (emails, getting/confirming info)
- [ ] YYYY-MM-DD — <task>

## Call Log (most recent last)
### MM/DD/YY — <call type>
- Summary: <what happened>
- Left off / next step: <where it ended>
```

### What the app pulls from each section

| In your photo | Put it in | App surfaces it as |
|---|---|---|
| Accounts, balances, custodians | Snapshot → Accounts | 🏦 Assets widget; asset cards |
| Total moveable, current/target fee | Snapshot / fee line | 💸 Fee widget; Fee Delta prep card |
| Family layout & dynamics | Personal → Family | ❤️ Family & Why widget; recap personal line |
| Hobbies, retirement wants, lifestyle | Personal → Life & retirement | ❤️ Family & Why widget (the emotional why) |
| Spending figures, debts | Personal → Spending | 💵 Income & Spending widget |
| Social Security / pension estimates | Personal → Social Security / pension | 💵 Income & Spending widget |
| Their goals / why move | Snapshot → Why consolidate | 🎯 Goals widget; Opening/Pitch coaching |
| Their hesitation | Objection & Response | 🧯 Reframe-objection coaching; Pending |
| The recommendation + value | Snapshot → Solution; Prep Sheet | Decision-call recap; Prep cards |
| Next call date/type | Close Action → "Next call:" | Next-call banner; Upcoming Calls; morning Prep routine |
| To-dos you owe them | Tasks | Follow-up tasks on Command Center |
| What happened on the call | Call Log entry | Call history; last-touch; coaching recaps |

A profile that fills these in lets the app pull "whatever is needed, when it's
needed." Sparse sections just mean empty widgets — so err toward capturing more.
This template is the standing instruction for every notes-to-profile conversion
(this session, the regular Claude project, and the morning Prep routine).

## 9. Threat model

| Threat | Defended by |
|---|---|
| Cloud / sync breach | E2EE — only ciphertext leaves the device |
| Sync key leak | Two-key split — names need `MK_id`, which never syncs |
| Lost / stolen device | Passphrase gate + auto-lock; keys wiped from memory on lock |
| Weak-password guessing | PBKDF2 310k iters (Argon2id on the roadmap) |
| Tampered record | AES-GCM auth tag rejects it |
| Re-identification from notes | Narrative generalized to structured signals before storage |
