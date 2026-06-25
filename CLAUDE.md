# War Room — project brief for Claude

Read this first. It's the standing context for every session.

## ⚠️ Keep this file current (standing rule)
> Whenever you change the architecture, build pipeline, data model, or add/alter a
> feature, **update the relevant section of this file in the SAME change** — and
> bump anything that's now stale (paths, function names, the feature map). Future
> sessions rely on this file being accurate, so don't wait to be asked. Treat
> CLAUDE.md as part of the diff, not an afterthought.

## What this is
**War Room** is a single-file, local-first, encrypted PWA for **one Empower
financial advisor** (the repo owner) to prep for client calls. It syncs client
"intel" from a Google Drive folder of `— Profile.md` files, decrypts on-device,
and renders an Overview landing page, a Command Center, and per-client coaching/
planning views. The whole app
is **`index.html`** (vanilla JS, no build step), hosted on **GitHub Pages**.

The owner's loop: he takes a photo of his call notes → **pastes it to Claude** →
Claude writes/updates that client's `— Profile.md` on Drive → he hits **☁️ Load
Drive** in the app. The app is the polished read/coach surface; Claude is the
writer.

## Default action: a bare photo / notes = write the profile
If the user sends a **photo or screenshot of call notes** (or pastes call notes)
with little or no instruction, that IS the instruction — do NOT ask what to do:
1. Convert if needed (HEIC → jpg via pillow-heif) and read it.
2. Identify the client; **search the `1Remarkable` Drive folder by that name** to
   decide update-vs-create.
3. Write/update their `— Profile.md` to Drive following `docs/data-model.md §8`
   (capture everything, fidelity, labeled Personal bullets; add a Prep Sheet if
   there's an upcoming proposal/decision call). Use "about"/"≈", never bare `~`.
4. Reply with a short summary of what you captured and flag anything the notes
   didn't contain as "confirm" (don't invent it).

So the user can just drop a photo with no text. (A couple of words like the
client's name or "new notes" never hurt, but aren't required.)

## Hard rules (do not violate)
- **Public repo.** Never commit secrets, client data, names, balances, or
  profiles. No passphrase in the repo (it's set via the app UI, never hardcoded —
  hardcoding would publish it). No real client info in code, commits, or docs.
- **Fidelity mode.** When writing profiles/coaching, use only what's in the
  source notes. Never invent figures. If a number is implied but not stated, flag
  it "confirm"/"not captured" rather than guess. Emails/coaching must never assert
  an unverified fee saving.
- **Model identity** must not appear in commits, PR bodies, code, or any pushed
  artifact — chat only.
- **Branches:** push every change to **`main`** AND to
  **`claude/1remarkable-drive-access-twhos4`** (keep them in lockstep).
- **Commit footer** (every commit):
  ```
  Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
  Claude-Session: https://claude.ai/code/session_018MqCSiGsZiTcth3wCLYFVR
  ```
- **Don't open a PR** unless asked.

## Architecture
- **`index.html`** = the entire app. Find functions with grep; it's large.
- No service worker. Cache-busting = a `BUILD` stamp (`const BUILD` near the top,
  shown in the footer) + the user appending `?x=NN` / hard-refreshing.
- **Crypto:** Web Crypto AES-256-GCM + PBKDF2; two-key split (names need a master
  key that never syncs); IndexedDB (`warroom_db`). No passphrase recovery by design.
- **Drive sync:** Google Drive REST v3 via GIS OAuth. The app holds **full Drive
  scope** (read/write/delete) and **auto-dedupes** — one canonical `— Profile.md`
  per client, keeping the newest and deleting the rest on each sync.
- OAuth client ID + passphrase live in the browser (localStorage / user's head),
  **never** the repo.

## Dev workflow
1. Edit `index.html` directly (single file; match the surrounding terse style).
2. **Syntax check:** extract the `<script>` blocks and `node --check`:
   ```
   python3 -c "import re;open('/tmp/c.js','w').write('\n;\n'.join(re.findall(r'<script>(.*?)</script>',open('index.html').read(),re.S)))" && node --check /tmp/c.js
   ```
3. Bump `BUILD` (e.g. `r69` → `r70`) so the user can confirm the deploy is live.
4. Commit + push to **both** branches.
5. **Deploy:** GitHub Pages rebuilds on push (~1–3 min). To confirm it's live,
   check the "pages build and deployment" run via the GitHub MCP `actions_list`
   (the live site itself is blocked from this environment). The user then
   hard-refreshes / uses `?x=NN`.
- For logic changes, write a tiny Node test of the extracted function against
  realistic input before shipping (there's no test suite).

## Data model & profile format
- **`docs/data-model.md`** is the source of truth.
  - **§7** = the **Prep Sheet** cards (Claude-authored, app-rendered:
    `## Prep Sheet` → `### Title (table|cashflow|flags|angles)`).
  - **`## Opening`** (optional, Claude-authored) = a **verbatim, second-person**
    opening recap. When present it **overrides** the app's generated 🎬 Opening
    (blank-line-separated paragraphs become spoken beats). Author it on every
    notes→profile write and in the morning Prep routine — the template generator is
    a fallback; a hand-written opening is far better. Open with a real personal
    question, recap what matters in their words, then drive to today's purpose.
  - **§8** = the **canonical profile template + "capture everything" routing
    table.** Follow it for every notes→profile write: every fact from the photo
    goes into the section the app reads it from; nothing summarized away.
- The app parses these profile sections: `## Snapshot` (Stage, moveable assets,
  Accounts, Product, Confidence, Solution, Why), `## Personal / Relationship`
  (labeled bullets: Family / Life & retirement / Spending / Social Security-pension
  / Health), `## Objection & Response`, `## Opening` (authored verbatim recap),
  `## Prep Sheet`, `## Close Action / Next
  Step` (incl. the `Next call:` line and the `Missed call: <ISO>` line), `## Tasks`,
  `## My Notes`, `## Call Log`, and a `Status:` line (won/lost).
- **Sticky fields** (preserved across sync even if a re-parsed profile lacks them):
  the manual next-call override, call-type override, `## My Notes`, `## Tasks`,
  `## Prep Sheet`, and won/lost status. **Exception — the next-call override (date +
  type) only sticks when the profile is silent about the next call.** If a synced
  profile's Close Action either **books a real `Next call:` date** *or* **explicitly
  says no call is booked** (see "No next call" below), the **profile wins** and the
  stale in-app override is dropped — otherwise an old date/type pins the client to a
  call that already happened (e.g. still showing today's decision call after the notes
  rebooked a follow-up). In-app reschedules write the date+type back to Drive
  (`syncNextCallToDrive`), so a legit override survives the re-parse.
- **No next call** — when a call ends with nothing rebooked (a pending decision the
  client will "get back to you" on), write it explicitly in `## Close Action / Next
  Step`: lead with **`No call scheduled`** (or use `Next call: none` / `TBD` /
  `not scheduled` / `to be determined`). `parseProfile` reads this into
  `intel.nextCallCleared`, which **clears any stale manual next-call override** on the
  next sync (mirrors `missedCall` being authoritative). Without it, an old in-app
  next-call date sticks around and the client wrongly stays in **Upcoming Calls**
  instead of dropping into **Pending Decisions**. (`nextExpectedDate` returns null →
  Pending. A future profile that books a real `Next call:` date surfaces normally.)
- **Missed call** — a client's detail next-call banner has a **📵 Missed** button
  (`markMissedCall`). It's **Drive-synced**: `profileSetMissed` writes a
  `Missed call: <ISO>` line into the profile and **drops the booked `Next call:`
  line**, plus a dated `## My Notes` entry — so the state crosses devices.
  `parseProfile`→`mapImport` read it back into `intel.missedCall` (NOT a local sticky).
  `nextExpectedDate` returns null for a missed client, so they surface at the **top of
  Pending Decisions** (any stage, tagged 📵 MISSED) and are excluded from Cooling.
  Setting a new next call (`profileSetNextCall`) or logging a call (`profileEdit`)
  clears the marker.

## Writing profiles from photos (the main workflow)
- HEIC photos: convert with `pillow-heif` (`pip install pillow-heif`; register the
  opener, then PIL `.save(...jpg)`), then Read the jpg.
- Resolve the Drive folder by **searching for the `1Remarkable` folder by name**
  (don't hardcode its ID in this public repo); write with the Google Drive MCP
  `create_file` (`text/markdown`, `disableConversionToGoogleType: true`). The MCP
  connector here is **create-only**, so each write makes a new file — the app
  dedupes on the next sync. Always check for an existing profile first (search by
  name) to update rather than duplicate.
- Use the §8 template, labeled Personal bullets, and add a Prep Sheet for clients
  with an upcoming proposal/decision call. Use "about"/"≈" for approximate values,
  **never a bare `~`** (markdown viewers render `~...~` as strikethrough).

## Other workflows
- **Morning Prep routine** (`docs/prep-today.md`): a scheduled Claude Code
  *Routine* (claude.ai/code/routines) that each weekday finds clients with a call
  that day and writes their Prep Sheets to Drive.
- **Notes & tasks** sync to Drive (`## My Notes`, `## Tasks`) so they cross
  PC↔phone. **Tasks** = things the advisor owes the client (send an email, get/
  confirm specific info) — NOT proposal-building (the proposal call covers that)
  and NOT the client's own to-dos.

## Feature map (what already exists — don't rebuild)
- **Overview** is the **landing page** the app opens to (🏠 nav button above Command
  Center). It's a neon-HUD "command deck" (glassmorphism, scoped to `.ovwrap` so it
  stays dark even in light theme). Top = the only **action** block: **🎯 Today's Focus**
  (📵 missed calls to reschedule, surfaced FIRST as the most urgent action · calls today ·
  follow-ups owed — undated or due today/overdue). Directly under the header a **⏱ "since your
  last visit" delta strip** (`sinceStrip`) calls out what changed since the previous session —
  newly missed / cooling / pending clients + the pipeline $ move — diffed against the persisted
  snapshot ledger (see below); it's hidden on the very first visit. Everything below is a
  **stats/analytics board**: four **hero KPI tiles** (Weighted pipeline · Upcoming calls ·
  Cooling $ at risk · Pending decisions) that **deep-link** into the matching Command
  Center section and each carry a **week-over-week trend chip** (`trChip` — ▲/▼ vs the
  ~7-day-old snapshot, green=good/red=bad per metric direction; absent until a baseline exists),
  then six analytic panels — **Pipeline Velocity** (SVG bar+line of EV by **next-call week**,
  next 6 wks — forward projection; the subtitle also shows EV **not yet booked** = `weighted −
  bookedEV` so the chart reconciles with the Weighted-pipeline tile instead of silently
  undercounting), **Win Probability** (two conic
  gauges: EV-weighted avg close prob + historical close rate from the durable de-named
  `S.outcomeLog` ledger — losses are deleted from state so a live count can't see them;
  accrues as deals are marked Won/Lost), **Stage Funnel + Win Rate**
  (bars by stage + won/lost rate), **Activity Trend** (8-wk SVG of calls logged/week from
  `callHistory` dates — genuine history), **Discovery Depth** (avg "capture" gauge +
  per-topic coverage bars + thinnest profiles; reuses the client view's data-presence
  detectors `incomeSpendingBody`/`familyWhyBody`/`has(...)` for parity — **each coverage bar is
  click-to-expand**, revealing the clients missing that topic, each row deep-linking to the
  client), and **Objection
  Handling** (your **close rate by objection type** — every objection is bucketed via
  `objCategory()`, and a de-named won/lost **objection ledger** `S.objLog` (encrypted in
  IndexedDB meta, appended in `setStatus` via `logObjectionOutcome`) gives won-vs-lost per
  category; bars show strongest→toughest, **click-to-expand** to the open clients in that
  category. Rates
  accrue going forward since past won/lost objections weren't retained). The board **auto-refreshes
  on a calendar-day rollover** (`ovDayCheck` on visibilitychange/focus + a 60 s interval) so a
  left-open tab's Today's Focus / greeting / dates never go stale past midnight. On Overview the
  **whole left sidebar is hidden** (`#app.noside`, toggled in
  `renderList`) so it's a **full-screen** board; a **top toolbar** replaces it — brand +
  `+ Client` / `☁️ Drive` / `🌙 Theme` / `⚔️ Command Center` / `🔒 Lock` (these reuse the
  sidebar buttons' handlers via `.click()`). Client search/list lives in Command Center,
  one click away. Most values reuse existing computations; the only new storage is the
  **`S.ovSnap` snapshot ledger** — a de-named (codes only), encrypted (`meta/ovsnap`, under
  `MK_intel`) daily record of the headline KPIs + risk client-code sets, written once per
  session by `recordOverviewSnap` and read back to power the trend chips (#7) and the
  since-last-visit strip (#6). Baselines are **frozen once per session** (`S.ovSnapSession`) so
  theme/midnight re-renders don't reset the deltas to zero. `renderOverview()`.
- **Command Center** sections, in order: Upcoming Calls (This Week / Future tabs)
  → Top 10 → Pending Decisions → Follow-ups (tasks owed, next 2 weeks) → Cooling →
  Wins. Every client row shows a **note flag** (green pill + count when notes exist;
  click = notes popup inline). Below the section nav a **priority-filter chip row**
  (`.fchip`, state in `S.ccFilter`) narrows every active-derived section at once
  (All / CRITICAL / HOT / WARM / PIPELINE — only priorities present are shown). The
  **KPI strip** has five chips (Weighted · Upcoming · Pending · Follow-ups · Cooling),
  each a **button that deep-links** to its section and (when no filter is active) carries
  a **week-over-week trend chip** (`trChip`, reusing the Overview's `S.ovWeekAgo`
  baseline; suppressed while filtered since the baseline is the full book). Inline row
  actions: **Upcoming** rows have a 📞 **Log** button (`openLog`) + a `TODAY` tag on
  same-day calls; **Pending Decisions** rows have a 📅 **Reschedule** button
  (`quickReschedule` — prompts, saves, clears any missed marker, syncs to Drive);
  **Cooling** keeps its ✉️ re-engage. **Wins** has a This-month / This-quarter / All
  **time-frame toggle** (`S.winFrame`, filtered by `parseLogDate(wonDate)`) and shows a
  lifetime **win rate** (from `S.outcomeLog`) in its header. On **narrow phones
  (≤560px)** the data tables drop the low-signal probability column (`.col-opt`) and
  tighten spacing instead of scrolling sideways.
- **Client intel** top bar = two widget rows: **🎤 On the call** (coaching:
  Opening, Questions, How to pitch, Why Empower, Reframe objection, Next-step play,
  Watch-outs) and **📊 Client data** pop-up widgets (📋 TL;DR, Assets, Streams, Income &
  Spending, Family & Why, Fee, Goals, Pending, 📞 Call history) — each opens in the shared
  pop-up via `dataWidget(c,key)`. Below the widgets sits only the **🧾 Prep Sheet** group
  (the deeper Claude-authored cards). There's no inline "History & Notes" section: TL;DR
  and Call history are widgets above, and **notes** live in the top-bar **🗒️ Notes** dialog.
  The **❓ Questions** pop-up always **leads with the discovery topics this client's
  notes are missing** (the same six `DISCO_TOPICS` the Overview Discovery Depth stat
  scores), then the situational asks — so every gap gets probed on the next call.
- **Opening recap**: if the profile carries a `## Opening` section (Claude-authored),
  the 🎬 Opening pop-up renders **that** verbatim. Otherwise it falls back to a
  **call-type-aware generated recap** (Decision recaps the solution+value and
  drives to a decision; Enrollment = paperwork; Discovery = exploratory; Proposal =
  present-the-plan) opening with a **verbatim personal question** from the notes.
  Pop-up content is **what the advisor says out loud**; coaching meta goes in the
  small footnote. Profile clauses (solution/goals are written third-person) are run
  through **`youify()`** so the spoken beats read second-person ("his daughters" →
  "your daughters", "he values" → "you value") — i.e. truly verbatim.
- Editable **next-call date + type** (writes back to the Drive `Next call:` line);
  **Won** strips the profile to a minimal record; **Lost/Delete** removes the Drive
  file entirely.

## Systems in the code worth knowing (grep for these)
The file is large; these are the load-bearing pieces a new session will likely touch.
- **Pipeline ranking** — `expectedValue(i)` = moveable × `stageProb(i)` × `recencyFactor(i)`;
  powers the **Top 10** list.
- **Cooling / deal-state** — `dealState(c)` + `isCooling(c)` decide whether a proposal is
  fading; drives the **Cooling** section and the OVERDUE / "Nd quiet" tags.
- **Next-call inference** — `nextExpectedDate(i)` honors the manual override and parses fuzzy
  dates via `parseLogDate` (e.g. "the 30th", "late June"); `nextCallType(i)` infers
  Intro/Discovery/Proposal/Decision/Enrollment from the notes (rule of thumb: advance one
  stage past the call just held). These drive **Upcoming Calls** + the next-call banner.
  `nextCallMinutes(i)` extracts the call's clock time so same-day Upcoming rows sort earliest-first.
- **Log call** — `openLog` → `driveLogCall(name, editFn)` with `profileEdit` appends a
  Call Log entry, updates Stage/next-step, and writes back to Drive in place.
- **Re-engage email** — `openReengage` drafts a re-engagement email; `driveSaveEmail` saves it
  as a Google Doc in the `1Remarkable` folder (offered for cooling/proposal clients).
- **In-app profile editor** — `renderEditor(c)` edits fields and writes back via `driveLogCall`.
- **KPI strip** — `deriveKpis(c)` synthesizes the chip row when a profile carries none.
- **Surgical Drive writers** — `profileEdit`, `profileSetStatus`, `profileSetTasks`,
  `profileSetNotes`, `profileSetNextCall`, `profileSetMissed`, and `stripSection` each edit
  ONE part of the markdown losslessly; `driveLogCall` reads the live newest file, applies the edit, writes it
  back in place, then deletes duplicate copies (the auto-dedupe).
- **Views / navigation** — three top-level views set `S.active`: `renderOverview()`
  (`'__overview__'`, the landing page), `renderDashboard(jumpTo)` (`'__dash__'`, the
  Command Center; optional `jumpTo` = a `sec-*` anchor id to scroll to, used by the
  Overview hero tiles), and `renderDetail(c)` (a client). `renderList()` paints the
  sidebar and collapses it on Overview. Each sidebar row shows a **call-scheduled
  checkbox** before the name (filled green ☑ when `nextExpectedDate` is today/future,
  empty ☐ when none), and a sub-line `callType · moveable · 🗓 last-conversation date`
  (`effTouch(c)`), with the date color-aged fresh→warm→cold→stale (≤7d / ≤14d / ≤30d /
  >30d) so quiet clients pop when scanning. Boot opens `renderOverview()`.
- **Parse → map → render** — `parseProfile(md)` → `mapImport(s)` (→ encrypted `intel`) →
  `renderDashboard()` / `renderDetail(c)`. Add a new profile field in all three.

## Conventions / gotchas
- Pop-up/coaching text = verbatim spoken lines; put advice/meta in a footnote.
- Approximate values: "about" or "≈", never bare `~` in profiles.
- Keep new code in the file's existing compact idiom.
- The current build is in the footer (`const BUILD`); bump it every shippable change.
