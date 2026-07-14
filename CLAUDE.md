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
- **Branches:** `main` is the live branch (GitHub Pages serves it) — commit and push
  every change straight to **`main`**. No lockstep/backup branch is maintained; rely on
  `main`'s history. (Per-task `claude/*` branches may be created for in-flight work, but
  delete them once merged so they don't accumulate.)
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
   The stamp shows in the footer **and on the lock screen** (`#lockBuild`), so
   the user can eyeball what's deployed before unlocking.
4. Commit + push to **`main`** (Pages serves it). Delete any per-task `claude/*`
   branch once merged — see the ⚠️ below, ref-deletion is blocked here.
5. **Deploy:** GitHub Pages rebuilds on push (~1–3 min). To confirm it's live,
   check the "pages build and deployment" run via the GitHub MCP `actions_list`
   (the live site itself is blocked from this environment). The user then
   hard-refreshes / uses `?x=NN`.
   - ⚠️ **Known-flaky deploy (don't panic, don't chase the code).** The Pages
     job intermittently fails at the final publish step with
     `##[error]Deployment failed, try again later.` — the **build + artifact
     upload always succeed**; it's a transient GitHub Pages hiccup, **never the
     code** (`node --check` already passed). **Fix = just re-trigger the deploy.**
     Most reliable: push an **empty commit** (`git commit --allow-empty -m
     "Redeploy rNN (transient Pages publish error)"`) — that mints a fresh run.
     `actions_run_trigger method:rerun_failed_jobs` sometimes works too but has
     been seen to sit stuck in `queued`, so prefer the empty commit. It usually
     clears on the next attempt (occasionally takes two). This is expected
     during Pages' flaky windows — re-fire and move on; do not start debugging
     `index.html`.
   - ⚠️ **Deleting the `claude/*` branch:** `git push origin --delete …` is
     **blocked by the egress proxy (HTTP 403)** and there is **no delete-branch
     MCP tool**, so a session here *cannot* remove the remote branch. Delete the
     **local** branch (`git branch -D`) and tell the user to click delete at
     `github.com/<owner>/<repo>/branches` (it's merged, so it's harmless). Don't
     re-attempt the remote delete each session — it will 403 every time.
- For logic changes, write a tiny Node test of the extracted function against
  realistic input before shipping (there's no test suite).

## Data model & profile format
- **`docs/data-model.md`** is the source of truth.
  - **§7** = the **Prep Sheet** cards (Claude-authored, app-rendered:
    `## Prep Sheet` → `### Title (table|cashflow|flags|angles|model)`). The **`model`**
    type = interactive **scenario calculators**: `### Scenarios to Model (model)` with
    items `- Title :: kind :: k=v; k=v :: note`. Each gets a **⚙ Model this** button that
    runs the closed-form math in-browser and shows a talk-through pop-up (spoken lines +
    figures + live dials). The engine is **generic — every client/profile going forward**;
    you only author the card. Wired `kind`s + params are tabulated in §7 (`home-vs-invest`,
    `income-change`, `free-cashflow`, `contrib-fv`, `retire-proj`, `annuity-vs-invest`); the calculators/dials
    live in `scenarioModel()`/`modelDials()` in `index.html`. **Whenever a profile has an
    upcoming call where numbers matter (home/income/retirement/college decisions), author a
    `(model)` card with the right `kind`s + the client's figures** — that's what lights up
    the buttons. Directional only: every pop-up labels its assumptions, asserts no saving as
    fact (fidelity rule).
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
- **Labeled Personal bullets parse to structured fields** (since r98). `parseProfile`
  routes each `## Personal` bullet by its label into `family` / `lifeRetire` /
  `spending` / `ssPension` / `health`, and the 💵 Income & Spending and ❤️ Family & Why
  widgets render those **deterministically** (the old free-text regex is now only a
  fallback when the labels are absent). So author the labeled bullets — that's what the
  widgets read. Also derived: `household.retirementGoal` (from an explicit
  `Retirement goal:` line or a `$X/mo|$/yr` figure in Life & retirement), and the **📈
  Income & Asset Streams** widget is populated from the Social Security / pension bullet
  (+ an optional `## Income Streams` section) — it's no longer dead.
- **All `## Objection & Response` pairs are parsed** (since r98), not just the first —
  `intel.objections` is the full array, feeding the 🧯 Reframe coaching and the Overview
  Objection-Handling analytics. List multiple `- Objection:` / `- Response:` pairs freely.
- **`specialFlags` = explicit `Watch:` clauses + a whole-profile signal scan** (since r98):
  crypto / single-stock / annuity / contractor / rental / RMD / LTC / estate keywords found
  anywhere in the profile become flags, so the 🎯 Pitch & Why Empower coaching (its Make-the-case
  differentiators and Mind-these guardrails) fires even when those signals aren't crammed into a
  `Watch:` line. An **Empower-held
  account counts as an anchor** even when its tag is blank.
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
  instead of dropping into **Pending Decisions** (proposal/decision stage) or **No Call
  Booked** (any other stage). (`nextExpectedDate` returns null. A future profile that
  books a real `Next call:` date surfaces normally.)
- **`callSlot(i)` — the one classifier that carves the "no upcoming call" clients into
  the right section (since r122).** `nextExpectedDate` collapses missed / cleared /
  lapsed all into a single `null`; `callSlot` keeps **why** there's no call and returns
  `{slot}` = **`'upcoming'`** (a future call is booked), **`'missed'`** (a booked call's
  date passed with nothing rebooked, OR a hand-marked 📵 — a dropped ball, any stage),
  or **`'none'`** (never booked, or an explicit "No call scheduled" — intentional, not
  dropped). `bookedDateRaw(i)` is the resolved next-call date *without* dropping past
  ones (trusts only the deliberate booking — the manual override or the `Next call:`
  line — so a lapsed booking is distinguishable from a stray narrative date). The
  action sections are carved from these slots so they never overlap. **This is the fix
  for "missed calls not showing":** before r122 an auto-lapsed call only surfaced if the
  client was at proposal/decision stage (via the old combined Pending filter); an
  early-stage client whose booked call passed showed up **nowhere**. Now any `'missed'`
  slot lands in the dedicated Missed Calls section regardless of stage.
- **Missed call** — a client's detail next-call banner has a **📵 Missed** button
  (`markMissedCall`). It's **Drive-synced**: `profileSetMissed` writes a
  `Missed call: <ISO>` line into the profile and **drops the booked `Next call:`
  line**, plus a dated `## My Notes` entry — so the state crosses devices.
  `parseProfile`→`mapImport` read it back into `intel.missedCall` (NOT a local sticky).
  `callSlot` returns `'missed'` for them, so they surface in the dedicated **📵 Missed
  Calls** section (any stage, tagged 📵 MISSED, longest-lapsed first) and are excluded
  from Cooling. **A hand-marked 📵 and an auto-lapsed booked call are both `'missed'`**
  — the marker just makes it explicit / Drive-synced. Setting a new next call
  (`profileSetNextCall`) or logging a call (`profileEdit`) clears the marker.

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
- **When a `(model)` scenario CAN'T be authored because a figure is missing, say so
  IN the profile** — don't silently omit the card. Add a note (a `⚙ …` flags item on
  the Prep Sheet and/or a dated `## My Notes` line) naming the exact inputs to collect
  and which `kind` unlocks once they're in (e.g. "get balances + fee → fee-savings +
  retire-proj models run"). The advisor wants to know a model is *available* pending
  data, not discover the gap on the call. (Never author a model card on invented
  numbers to make a button appear — note the gap instead.)

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
  follow-ups owed — undated or due today/overdue). It's a **collapsible card** (state in
  localStorage `wr_focusCollapsed`, default open); folded, its header keeps a one-line count
  summary (📵 n · 📞 n · 📌 n, `!` = overdue present). Directly under the header a **⏱ "since your
  last visit" delta strip** (`sinceStrip`) calls out what changed since the previous session —
  **🏆 new wins (count + $, good news leads)**, newly missed / cooling / pending clients + the
  pipeline $ move — diffed against the persisted snapshot ledger (which now also carries
  `wonQn`/`wonQval`; the win delta is guarded > 0 so a quarter rollover can't show a negative
  win); it's hidden on the very first visit. Everything below is a
  **stats/analytics board**: six **hero KPI tiles** (Weighted pipeline · Upcoming calls ·
  **Missed calls** (t-red, →`sec-missed`) · Cooling $ at risk · Pending decisions ·
  **Won this month** — the last deep-links to 🏆 Wins
  and its ring shows % of the monthly goal) that **deep-link** into the matching Command
  Center section and each carry a **week-over-week trend chip** (`trChip` — ▲/▼ vs the
  ~7-day-old snapshot, green=good/red=bad per metric direction; absent until a baseline exists),
  then a full-width **🏆 Wins & Momentum** row (a **monthly assets-won goal ring** — target set
  via prompt, cached in localStorage `wr_winGoalM`, accepts "4M"/"4000000", **defaults to $4M**
  when unset since this is a single-user app — it tracks the **current calendar month's** won
  assets (`wonMval`/`wonMn`, mirrored into the snapshot ledger) — beside a 6-month
  SVG bar chart of **$ won per month** with each bar labelled by win count; source is the durable
  won-client records `wonAmount`/`wonDate` **plus a manual monthly backfill**. The Wins row has a
  **📊 backfill** button (`openBackfill`) → a dialog of one $-input per month (current calendar year,
  Jan→this month) so the advisor can enter **raw monthly win totals with no client data** — useful to
  seed history (e.g. first half of the year). Cached in localStorage `wr_winHist`
  (`{"YYYY-MM": dollars}`, de-named — just $ totals); `winHist()`/`winMonthKey()`
  read it and `histIn(start,end)` sums it into the goal ring (`wonMval`/`wonQval`) and every momentum
  bar (it **adds on top of** any client-derived $ for that month, so months with no client records still
  show their real number; backfill carries no win count, so those bars show $ with no count label)).
  **Both the goal and the backfill sync across devices via Drive** (not just localStorage): they're
  written to a tiny non-sensitive JSON file `WarRoom Settings.json` in the `1Remarkable` folder
  (`{winGoalM, winHist, outcomeLog, objLog}` — the last two added r110, de-named close-stat ledgers).
  `pushSettings()` (fire-and-forget through `withDriveToken`) writes on every goal/backfill save **and
  after every decided deal**; it routes through `writeSettingsMerged`, which **unions** the local
  ledgers with Drive's (by row `id`) before writing so neither device's outcomes are lost or doubled.
  `syncFromDrive` calls `syncSettings(folderId)` to **pull** goal/backfill into localStorage
  (remote wins; if Drive has none yet, the local values **seed** it) and union the ledgers both ways.
  `driveReadSettings`/`driveWriteSettings`
  mirror the profile read/PATCH-or-create pattern; the file name has no "Profile" so the client sync
  ignores it. localStorage stays the fast local mirror that the render reads,
  then six analytic panels — **Pipeline Velocity** (SVG bar+line of EV by **next-call week**,
  next 6 wks — forward projection; the subtitle also shows EV **not yet booked** = `weighted −
  bookedEV` so the chart reconciles with the Weighted-pipeline tile instead of silently
  undercounting), **Win Probability** (two conic
  gauges: EV-weighted avg close prob + historical close rate from the durable de-named
  `S.outcomeLog` ledger — `{outcome,date,amount}` per decided deal; losses are deleted from
  state so a live count can't see them; `amount` (since r99) keeps $ won durable past the
  stripped/deleted records; accrues as deals are marked Won/Lost), **Stage Funnel + Win Rate**
  (bars by stage + won/lost rate), **Activity Trend** (8-wk SVG of calls logged/week from
  `callHistory` dates — genuine history; weeks a deal closed carry a **🏆 marker** (×N if
  >1) pinned at the top + a "🏆 N closed" note in the subtitle, so the calls→close rhythm
  reads off one chart), **Discovery Depth** (avg "capture" gauge +
  per-topic coverage bars + thinnest profiles; reuses the client view's data-presence
  detectors `incomeSpendingBody`/`familyWhyBody`/`has(...)` for parity — **each coverage bar is
  click-to-expand**, revealing the clients missing that topic, each row deep-linking to the
  client), and **Objection
  Handling** (your **close rate by objection type** — every objection is bucketed via
  `objCategory()`, and a de-named won/lost **objection ledger** `S.objLog` (encrypted in
  IndexedDB meta, appended in `setStatus` via `logObjectionOutcome`) gives won-vs-lost per
  category; bars show strongest→toughest, **click-to-expand** to the open clients in that
  category. Rates
  accrue going forward since past won/lost objections weren't retained). **Both close-stat
  ledgers (`S.outcomeLog` + `S.objLog`) now also sync across devices via Drive** (since r110):
  each row carries a stable `id`, and they ride along in `WarRoom Settings.json` next to the win
  goal/backfill. `logDealOutcome` stamps the id, appends locally, and fires `pushSettings()`;
  `writeSettingsMerged` **unions** this device's ledger with Drive's (by id; legacy rows by value)
  so losses — which are deleted from client state — survive and never double-count. This is why
  the **Win-Probability "Close rate" gauge** and the **Stage-Funnel "% win rate · W·L"** were
  previously stuck at 0 on a second device / fresh load: the ledger was IndexedDB-local and
  forward-only. (The funnel **bars** still show only the *open* pipeline — a won deal graduates
  out of its stage by design; the **"Avg close prob"** gauge is live current-state EV-weighting.) The board **auto-refreshes
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
- **"This Week" window** (`thisWeekEnd(today0)`, since r118) — the Upcoming "This Week" tab (Command
  Center) and the Overview "N this week" KPI both bound on the **coming Sunday (23:59)**, but **from
  Saturday noon onward (and all day Sunday) the window rolls forward a full week**, so the weekend is
  spent prepping the next working week — "This Week" then shows the upcoming week's calls, not the 1–2
  days left in the week that's ending. Both render sites call `thisWeekEnd`; `weekBucket()` is a token
  that flips at the same pivot, so `ovDayCheck` (the 60 s / focus / visibilitychange auto-refresh)
  re-renders a left-open Overview tab when the rollover fires (it stamps `S.ovRenderWeek` alongside
  `S.ovRenderDay`). If you add another "this week" boundary, reuse `thisWeekEnd` so they stay in sync.
- **Command Center** sections, in order: Upcoming Calls (This Week / Future tabs)
  → **📵 Missed Calls** → Top 10 → **⏳ Pending Decisions** → **📇 No Call Booked** →
  Follow-ups (tasks owed, next 2 weeks) → Cooling → Wins. **Missed Calls** and **No Call
  Booked** are mutually-exclusive *action* sections carved from `callSlot`: **Missed
  Calls** (`slot==='missed'` — a scheduled call that lapsed or a 📵 marker, ANY stage,
  most urgent); **No Call Booked** (`slot==='none'` + any *non*-proposal/decision stage —
  early-stage clients who need a next touch scheduled, previously invisible). **Pending
  Decisions is NOT a call-state bucket — it's a CROSS-CUTTING "closest to close" status
  board (r124).** It lists **every deal awaiting a yes/no** (`awaitingDecision(i)` —
  proposal delivered, no won/lost yet), *regardless* of call state, so it **overlaps**
  Upcoming / Missed / Cooling on purpose (a booked decider shows in Upcoming AND Pending).
  Each row carries a **status tag** (`pendTag` — 📅 booked call · 📵 missed · 🧊 cooling ·
  ⏳ no call) + the deal's EV, sorted **closest-first** (booked call soonest → `stageProb`
  → EV). Before r124 Pending was a thin mutually-exclusive slice (proposal + no-call +
  not-cooling) that read **zero** because every decider got siphoned into Upcoming/Missed/
  Cooling — the redesign is purely additive (the action sections are unchanged; Pending
  just *also* surfaces the deciders). `awaitingDecision` gate = Proposal/Decision stage
  (inferStage sets these only from a HELD call, so stage already means "proposal
  delivered" — incl. proposals delivered on a screen-share whose notes mention it) OR a
  past/undated proposal|decision call in `callHistory`. Cooling excludes any `'missed'`
  client (was just `!missedCall`), so an auto-lapsed proposal reads as **Missed**. Every
  client row shows a **note flag** (green pill + count when notes exist;
  click = notes popup inline). A **priority-filter chip row**
  (`.fchip`, state in `S.ccFilter`) narrows every active-derived section at once
  (All / CRITICAL / HOT / WARM / PIPELINE — only priorities present are shown). The
  **KPI strip is the sole section-nav surface** (the old "Jump to" `.secnav` row was
  removed in r101 — every widget is now a tappable jump): **eight chips, one per
  section** (Weighted→Top 10 · Upcoming · **Missed** · Pending · **No call booked** ·
  Follow-ups · Cooling · **Won this quarter**→Wins) — every `sec-*` group has a matching
  `data-jump` chip (r126; keep it that way when adding a section),
  each a **button that deep-links** to its section (`data-jump`→`scrollIntoView`) and
  carries a visible **tap affordance** (`.kc-go`, e.g. "Top 10 ›", "Wins ›") naming the
  destination, plus (when no filter is active) a **week-over-week trend chip**
  (`trChip`, reusing the Overview's `S.ovWeekAgo`
  baseline; suppressed while filtered since the baseline is the full book). The Wins chip
  shows quarter-won $ (`wonQval`) + the lifetime win rate; Top 10 and Wins have no
  like-named widget so the Weighted-pipeline and Won-this-quarter chips are their labeled
  entry points. (`.secnav` still styles the per-client Prep-Sheet jump nav in
  `renderDetail`.) Inline row
  actions: **Upcoming** rows have a 📞 **Log** button (`openLog`) + a `TODAY` tag on
  same-day calls; **Missed Calls**, **Pending Decisions**, and **No Call Booked** rows all
  have a 📅 button (`data-resched` → `quickReschedule` — opens the **schedule picker**,
  saves, clears any missed marker, syncs to Drive); **Cooling** keeps its ✉️ re-engage. **Wins** has a This-week / This-month / This-quarter
  / YTD / All **time-frame toggle** (`S.winFrame`, filtered by `parseLogDate(wonDate)` against
  `_wfBounds`; week = Sunday-start) and shows a lifetime **win rate** (from `S.outcomeLog`) in
  its header. Changing the timeframe **repaints only the Wins card in place** via `repaintWins()`
  (no full `renderDashboard()`), so the scroll never jumps — a full re-render reset it, and
  restoring `#main.scrollTop` didn't help on phones where the **body** (not `#main`) is the scroll
  container. `repaintWins()` is also called once on dashboard mount to do the initial Wins paint.
  On **narrow phones
  (≤560px)** the data tables drop the low-signal probability column (`.col-opt`) and
  tighten spacing instead of scrolling sideways.
- **Client intel** top bar = two widget rows: **🎤 On the call** (coaching, four
  buttons mapped to the call arc: Opening → Questions → Pitch & Why Empower → Next-step
  play) and **📊 Client data** pop-up widgets — **consolidated 9→5 in r107** to five
  sectioned pop-ups: **📋 TL;DR** (the close summary, now also carrying the **Decision
  pending** row — the old standalone Pending widget was folded in; its risk badge just
  duplicated the header priority badge), **🏦 Assets & Fee** (account breakdown +
  transferable/fee/retirement-goal roll-up — old Assets + Fee merged), **💵 Income**
  (income/asset streams + spending, with SS/pension shown once — old Streams + Income &
  Spending merged; SS/pension feeds `incomeStreams`, so its bullets render only when no
  structured streams exist), **❤️ Why** (family + life/retirement + goals — old Family &
  Why + Goals merged), and **📞 Call history** (a **scannable timeline** — since r131 each
  entry is a compact card: header = `date · type` + a right-aligned relative **"when"** chip
  (`agoLbl` → "2 wks ago"); **"what happened"** (the raw `summary`) is **CSS line-clamped to 2
  lines** so you scan the gist, and `wireCallHist()` flags only the entries that actually
  overflow as `.ch-clamped` (tap the card → toggles `.ch-open` to reveal the full text — nothing
  is lost, it's all still in the DOM); the **next step** we had in place renders on its own
  `→ Next:` line (`.ch-next`, was the plain "Left off:" line). Newest-first). Each opens in the shared pop-up via
  `dataWidget(c,key)`; the merged bodies stack `incomeSpendingBody`/`familyWhyBody`/
  `tldrBody` plus per-section `.fin-grp`/`.fin-lbl` sub-headers (a legacy free-text income
  profile still falls back to `incomeSpendingBody`'s regex path). Below the widgets sits only the **🧾 Prep Sheet** group
  (the deeper Claude-authored cards). There's no inline "History & Notes" section: TL;DR
  and Call history are widgets above, and **notes** live in the top-bar **🗒️ Notes** dialog.
  The **❓ Questions** pop-up always **leads with the discovery topics this client's
  notes are missing** (the same six `DISCO_TOPICS` the Overview Discovery Depth stat
  scores), then the situational asks — so every gap gets probed on the next call.
  The **🎯 Pitch & Why Empower** pop-up is the **whole body of the call in one widget**
  (consolidated r104–r106 from the old How-to-pitch + Why-Empower + Reframe-objection +
  Watch-outs buttons). It renders three sections: **🎯 Make the case** (recommendation →
  frame → their drivers → flag-tailored Empower differentiators → planning-depth/team),
  **🧯 If they push back** (the on-file objection + its verbatim `response` + a tactical
  reframe — only when an objection is on file), and **🚩 Mind these** (flag cautions +
  the standing "verify linked accounts" guardrail). The **close** (product/goal/ask) is
  NOT restated here — it lives in **🤝 Next-step play**, so the two don't overlap. All
  built in `coachContent(c,'pitch')`.
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
- **Date parsing — THREE paths, each must range-check & honor "no call" (r115–r116 fix).** A
  number that merely *looks* like `MM/DD` (an **age** like "retire at 57/58", a **rate** like "3/6",
  a **kid's age**) must never become a phantom next-call date. The banner showed a bogus "57/58
  DISCOVERY" call because these weren't guarded. The three places that turn text into a date — keep
  them in sync:
  1. **`parseLogDate(s)`** — its `MM/DD` regex range-checks month 1–12 / day 1–31; an out-of-range
     match **falls through** to the month-name patterns (it does NOT return a junk `Date`).
  2. **`extractNextCall(text)`** — has its **own** date regex (used by `mapImport` to populate
     `intel.nextCall`, which the banner can print **literally**). Same range restriction baked into
     the regex (`(0?[1-9]|1[0-2])/(0?[1-9]|[12]\d|3[01])`). **Reads the explicit `Next call:` line
     FIRST (r121 fix).** The Close Action often *opens* with a past reference ("6/30 touch-base held
     …") and then books the real date on a `Next call: <Type> — Aug 5 10:00 AM` line. Scanning the
     whole blob grabbed that first `6/30` and masked `Aug 5`, so the client showed the call we'd just
     held. Now it isolates the `next call:\s*(...)` clause and scans only that; a `Next call:` line
     with no parseable date returns `''` (don't fall back to a stray narrative date — let
     cleared/free-text handling take over). Only profiles with **no** `Next call:` line scan the
     whole blob (back-compat). So **always put the booked date on a `Next call:` line** — a date
     buried only in the narrative can be out-ordered by an earlier past reference.
  3. **`nextExpectedDate(i)` + `mapImport`** — both treat `intel.nextCallCleared` (the profile's
     explicit "No call scheduled", see Data-model "No next call") as **authoritative**, exactly like
     `missedCall`: `nextExpectedDate` returns null, and `mapImport` forces `nextCall=''` rather than
     scraping the close/summary text. So a "No call scheduled" client lands in **Pending Decisions**,
     not Upcoming. **Past-date guard (r121):** `nextExpectedDate` now drops **any resolved date that's
     already in the past** to null (a `fresh()` helper; today still counts as upcoming) — a booked
     call whose date has passed with nothing rebooked is treated like missed/cleared, so the client
     falls into **Pending Decisions** instead of clinging to a stale Drive-scheduled date. The
     authoritative `i.nextCall`/close-text explicit-date branches are **decisive** (future→return,
     past→null) — they never fall through to the "the 30th"→next-month heuristic, which would
     otherwise resurrect a phantom future call from a past reference. This generalizes the missed-call
     fix to **every** lapsed booking, not just explicitly-marked ones.
  Rule of thumb: if you add/edit any code that reads a date out of free text, **range-check it,
  short-circuit on `nextCallCleared`/`missedCall`, prefer the `Next call:` line, and drop past
  dates** — and remember stored `intel.nextCall` only refreshes on a ☁️ Drive **re-sync** (re-runs
  `parseProfile`→`mapImport`), not a plain reload.
- **Weekly schedule ingest** (since r117) — drop a plain **`WEEKLY SCHEDULE …`** file or Google Doc
  into the `1Remarkable` folder (day header like `Tuesday, June 30`, then rows `9:30 AM  Client Name —
  Type`) and every ☁️ Drive sync **books the listed calls**. It's purely **ADDITIVE**: a client who
  already has an upcoming call is **never overwritten**, past-dated rows are skipped, and unknown names
  are ignored (counted as skipped). **In-app state wins (r120):** a client marked **📵 missed**
  (`intel.missedCall`) or **"No call scheduled"** (`intel.nextCallCleared`) is **skipped** — without this
  guard the schedule re-added the very call you just dismissed on every sync (their `nextExpectedDate` is
  null, so the "already has a call" check didn't catch them; a same-day missed row also slips past the
  past-date skip). The marker clears the normal way (in-app reschedule / log-call / a profile that books a
  real `Next call:`), and only then does the schedule book again. `syncFromDrive` calls
  `applyWeeklySchedule(folderId)` **after**
  `loadClients()` (so it sees the freshly-synced `_md`/`_driveFileId`/`nextCall`): `driveReadSchedule`
  finds the newest file whose name contains "SCHEDULE" but not "Profile" (Google Docs are `export`ed to
  text/plain, plain files read via `alt=media`); `parseWeeklySchedule(text)` returns
  `[{name,date,type,time}]` (year from the title, else 2026; em/en/hyphen separators; "(No calls)" /
  "HOLIDAY" lines naturally don't match). For each eligible client (`nextExpectedDate` null or past, not
  won/lost) it writes `profileSetNextCall` into the profile (PATCH to the known `_driveFileId`, or
  `driveLogCall` create as a fallback) and sets the in-app `nextCallSet`/`nextCallTypeSet` so it shows
  immediately; the booking clears any `missedCall`/`nextCallCleared`. Because the booking lands in the
  profile, the next sync's override-preservation drops the temp override in favor of the profile-derived
  `nextCall` — so it's durable and crosses devices. Idempotent (re-syncing the same file re-skips
  everyone already booked). The sync alert reports `· booked N call(s) from schedule`. The schedule file
  is **not** parsed as a client (no "Profile" in the name).
- **Schedule picker** (since r119) — every "set/change a next-call date" surface is a
  **month/day/time dropdown dialog** (`#schedDlg`), never a free-text prompt. `openSched(opts)`
  returns a Promise → `{date,type}` | `{cleared:true}` | `{auto:true}` | `null` (cancel). It
  populates a rolling 14-month month list, a weekday-labelled day list (past days excluded), a
  **8:00 AM–7:00 PM ET / 15-min** time list (+ "No specific time"), and a call-type select.
  `schedFmt(y,mo,day,time)` builds the stored string so the **existing parsers** read it
  unchanged — 2026 (the working year) renders readable `"Jul 13 2:00 PM"` (parsed by
  `parseLogDate`'s month-name path, which defaults to 2026); other years use `"M/D/YY 2:00 PM"`
  so the year survives. The time always lands where `nextCallMinutes` can read it. Three callers:
  `quickReschedule`, the client-page `#setNcBtn` (Set/Change), and the **Log-call dialog's
  `#logNext`** field (now read-only, driven by a 📅 **Pick** button → `"<Type> call: <date>"`).
  `{auto}` deletes the override (revert to notes), `{cleared}` sets `nextCallSet=''` (no call).
  If you add another date-entry surface, route it through `openSched`, don't add a `prompt()`.
- **Log call** — `openLog` → `driveLogCall(name, editFn)` with `profileEdit` appends a
  Call Log entry, updates Stage/next-step, and writes back to Drive in place.
- **iPhone widget feed** (since r112) — `gatherWidgetFeed()` builds a compact digest
  `{today:{count,transferable,calls[]}, week[7], missed, followups, overdue}` (today's calls =
  name/type/clock-time/priority/moveable, transferable $ summed via `assetVal`; the `week` strip is
  today..+6 with per-day count+$). `driveWriteWidget`/`driveWidgetFile` mirror the settings
  read/PATCH-or-create pattern, writing **`WarRoom Widget.json`** into the same private `1Remarkable`
  folder as the profiles (name has no "Profile" so the client sync ignores it; the filename also
  isn't `WarRoom Settings.json` so the settings sync ignores it). It's refreshed on **every ☁️ Drive
  sync** (`syncFromDrive`, inline `await driveWriteWidget` → `driveShareWidget` → `maybeShowWidgetLink`)
  and after in-app **reschedules** (`syncNextCallToDrive` → `pushWidgetFeed()`, fire-and-forget, no-ops
  without a live token, skips the write when nothing changed; it shares too). **Consumed by ONE
  Scriptable widget script** (code lives outside the repo, pasted into the user's Scriptable app —
  named "Warroom") that renders both a **lock-screen** view (today's call count + total transferable $)
  and a **large home-screen** view (today's calls `name · type/time · $` + a 7-day overview strip +
  missed/follow-up counts), branching on `config.widgetFamily`.
  **Transport = share-by-link, NOT OAuth** (Google's device-flow OAuth rejects Drive scopes —
  `invalid_scope` — so a Scriptable widget can't sign in to Drive that way). Instead `driveShareWidget`
  sets the feed file to **anyone-with-link → reader** (idempotent, cached via localStorage
  `wr_widgetShared`), and `maybeShowWidgetLink` hands the user the **File ID** once via a copyable
  `prompt` (re-show anytime with `showWidgetId()` in the console; id cached in `wr_widgetFileId`). The
  widget fetches `https://drive.google.com/uc?export=download&id=<FILE_ID>` with a plain request — no
  credentials. **Tradeoff (user-accepted):** that one tiny digest file is readable by anyone holding
  the ~40-char unguessable link; it's never listed/searchable. The full profiles stay fully private
  (only the digest is shared). **Still nothing client-related in the public repo** — the shared file
  lives in the user's Drive, not the repo.
- **Re-engage email** — `openReengage` drafts a re-engagement email; `driveSaveEmail` saves it
  as a Google Doc in the `1Remarkable` folder (offered for cooling/proposal clients).
- **Scenario modeling (⚙ Model this)** — `scenarioModel(kind, params)` = the closed-form calc
  engine (helpers `_pmtMo`/`_loanBal`/`_fv`/`_fvMo`); `modelDials(kind, params)` = the live
  sliders; `openModel`/`paintModel` drive the `#modelDlg` pop-up (spoken lines + figures +
  dials that re-run on input). Fed by a **`(model)` Prep Sheet card** parsed in `parseProfile`
  (items get `{title, kind, params, body}`; `parseModelParams` reads the `k=v` list) and
  rendered by `prepCardBody(card, cardIdx)` as a **⚙ Model this** button per scenario
  (`data-model="cardIdx:itemIdx"`, wired in `renderDetail`). **Generic across all clients** —
  the button only appears when a scenario has a recognized `kind`. Adding a new scenario type =
  add a `case` to both `scenarioModel` and `modelDials` (keep param defaults in sync) + a row in
  data-model §7. Fidelity: every pop-up carries an assumptions note and asserts no figure as fact.
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
- **Responsive width (r132):** the shared content container is `.pad` (`max-width:1000px`). The
  **client-intel page** (`renderDetail`) opts into a wider desktop layout via `.pad wide` — at
  **≥1200px** it grows to ~1460px (1680px ≥1750px) and the **Prep Sheet** cards flow **two-up**
  (`.prepgrid` → 2-col grid; on phone/tablet `.prepgrid` is an unstyled block, so cards stack as
  before). All desktop widening is behind **`min-width`** media queries so the ≤820px phone/tablet
  layout is untouched; the Command Center + editor stay at 1000px (no `wide` class). If a new
  detail-page section should also fill the desktop width, put it inside `.pad.wide` and gate any
  multi-column CSS on `@media(min-width:1200px)`.
- The current build is in the footer (`const BUILD`); bump it every shippable change.
