# War Room — Client Intel

A local-first, encrypted app that turns structured client profiles into the
two-page **war-room prep sheets** (Data sheet + On-Call Helper) in the Remarkable
landscape format — generated on demand from one client record.

## What it does

```
Profile  ->  Encrypted intel record  ->  generates  ->  [Data sheet] + [On-Call Helper]
             name device-only;                           Remarkable-ready PDF
             numbers + signals encrypted
```

- **Client list / add / edit** — a focused form for accounts, income streams,
  goals, objections, product match, pitch flow, and the close.
- **Generate Prep** — opens the two-page prep sheet; in Chrome use
  **File → Print → paper 209.6mm × 157.2mm, Landscape, Margins None, Background
  graphics ON → Save as PDF**, then transfer to the Remarkable.
- **Import** — paste a `warroom-data.json` feed (or an array of records); names
  are split into the device-only store automatically.

## Security model (see `docs/data-model.md`)

- **Two-key split.** Names (`MK_id`) are device-only; numbers + signals
  (`MK_intel`) are encrypted and could later sync as ciphertext only. The two
  never sit together off-device.
- **AES-256-GCM at rest**, fresh random IV per write, GCM auth tag for tamper
  detection.
- **Passphrase-derived keys** (PBKDF2, 310k iterations). The passphrase is never
  stored or transmitted — there is **no recovery** if lost.
- Nothing more sensitive than **name + dollar amounts + structured signals** ever
  enters the system. No SSN, account numbers, address, or DOB.

> Built for a single advisor's own prep on his own device. Before storing real
> client data, confirm with Empower compliance that a personal productivity tool
> is permitted — encryption does not override firm policy.

## Run it

It's a single self-contained file — no build, no server:

1. Open `warroom-app.html` in Chrome.
2. First run: create a passphrase (this builds the vault and seeds the **Josh**
   example).
3. Pick a client → **Generate Prep** → print to PDF.

Data lives only in this browser's IndexedDB on this device. Use **Lock** to wipe
the keys from memory.

## Roadmap

- Argon2id key derivation (currently PBKDF2).
- Per-record DEK envelope encryption + zero-knowledge sync (Option B).
- Auto-build intel from the existing Drive `Profile.md` files.
- Enum vocabularies for `goals` / `objections` / `motivation` for consistency.
