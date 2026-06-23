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

## 8. Threat model

| Threat | Defended by |
|---|---|
| Cloud / sync breach | E2EE — only ciphertext leaves the device |
| Sync key leak | Two-key split — names need `MK_id`, which never syncs |
| Lost / stolen device | Passphrase gate + auto-lock; keys wiped from memory on lock |
| Weak-password guessing | PBKDF2 310k iters (Argon2id on the roadmap) |
| Tampered record | AES-GCM auth tag rejects it |
| Re-identification from notes | Narrative generalized to structured signals before storage |
