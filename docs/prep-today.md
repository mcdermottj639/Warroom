# Morning Prep Sheet run — daily runbook

This is the instruction a **scheduled morning session** runs to auto-generate
Prep Sheet cheat-sheet cards for every client with a call **today**, and write
them back to the `1Remarkable` Google Drive folder. The War Room app then shows
them as 🧾 Prep Sheet widgets at the top of Client Intel (it auto-dedups the
older copy on sync).

## How to schedule it (one-time, in Claude Code on the web)
Create a **scheduled trigger** on this repo's environment (Environments →
Triggers → "Scheduled"), set to run each morning (e.g. weekdays 7:00 AM ET),
with this prompt:

> Follow the runbook in `docs/prep-today.md` for today's date. Generate and write
> Prep Sheets for every client with a call today, then report who got one.

The trigger needs the **Google Drive connector** enabled (read + create) and the
repo in scope. Nothing else to do — it runs unattended.

## What the session should do each run

1. **Today's date.** Use the session's current date (ET). Note today's M/D and
   the ISO date.

2. **Find today's calls.** Search the `1Remarkable` Drive folder
   (`1SIWV5Xl9382wWvL7HhEBtlkiOPrsvw41`) for profile files whose **Next call**
   is today:
   - `search_files`: `fullText contains '<M/D>'` (e.g. `'6/23'`) AND
     `title contains 'Profile'`.
   - Also try the bare ordinal (`'the 23rd'`) and weekday for that date, in case
     a profile wrote the date that way.
   - **Read each candidate** and confirm the `## Close Action / Next Step`
     "Next call:" line (or Snapshot stage line) actually resolves to **today**.
     Skip past-dated or future-dated matches.

3. **Author a Prep Sheet** for each confirmed client **that doesn't already have
   a `## Prep Sheet` section** (skip ones that do, unless the notes changed
   materially since). Use the four card types — see
   `docs/data-model.md` §7 for the exact format. Pick the cards the client's
   notes actually support; omit a card if there's nothing real to put in it:
   - **Fee Delta / cost (table)** — fee or cost comparison / asset breakdown.
   - **Income & Cash Flow (cashflow)** — spending, income, timeline, the primary
     consolidation opportunity.
   - **Planning Flags (flags)** — the things to watch / plan around.
   - **Strategy Angles (angles)** — how to play the call.

3b. **Author an `## Opening`** for each confirmed client too — a **verbatim,
   second-person** recap the advisor reads aloud (the app renders it in place of
   the auto-generated opening). Three or so blank-line-separated beats: (1) a real
   personal question grounded in the Personal notes, (2) a recap of where things
   stand in the client's own terms — what matters most to them, (3) what today is
   about + the drive to the next step. Write it like a sharp advisor would actually
   talk — natural, specific, no third-person ("your daughters", "you value …", not
   "his"/"he"). Same fidelity rules apply. Refresh it if the notes changed.

4. **Fidelity rules (non-negotiable):**
   - Author **only** from what's in the profile notes. Never invent a number.
   - If a fee/savings figure isn't confirmed in the notes, label it
     *illustrative* and include the ask to confirm it (see Brandon Chambers'
     Fee Delta card for the pattern). Never assert an unverified fee saving.
   - Tie each flag/angle to something the notes actually say.

5. **Write it back.** Insert the `## Opening` section (just before `## Prep Sheet`)
   and the `## Prep Sheet` section just **before** `## Close Action / Next Step`,
   keep everything else byte-for-byte, and
   `create_file` the full updated profile into the same folder
   (`parentId = 1SIWV5Xl9382wWvL7HhEBtlkiOPrsvw41`, mime `text/markdown`,
   `disableConversionToGoogleType: true`). The app keeps the newest copy and
   deletes the duplicate on its next sync — so creating is safe.

6. **Report.** End with a short list: which clients got a Prep Sheet today, which
   already had one, and any call you couldn't confirm. Keep it brief.

## Notes
- The connector here is create-only, so each write makes a new file; the app's
  full-Drive-scope sync dedups to one canonical profile per client. Don't worry
  about duplicates.
- If there are **no** calls today, do nothing and say so — don't fabricate work.
