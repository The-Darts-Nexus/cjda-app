# CJDA — Technical Handover (Section 14)

> Companion to the business handover (Sections 1–13, held separately).
> This file is the technical source of truth for the CJDA app. Keep it current — every
> schema change, new engine, or business rule shift gets an entry in §14.7.
>
> **Last updated:** 2026-09-03 · **Assessment:** Business 9/10 · Technical **9/10**

---

## 14.0 Snapshot

| | |
|---|---|
| **App** | Single file — [`index.html`](index.html) (~4,240 lines). No build step, no framework. |
| **Front end** | Tailwind Play CDN + `@supabase/supabase-js@2` (UMD). One inline `<script>` from line ~122. |
| **State** | One global `const state = {…}`. `render()` rebuilds `#root.innerHTML` on every change (full re-render). |
| **Backend** | Supabase project `zsynlrutfkdjmmyznhst`. All data scoped to one space (`SPACE_ID = 00ca89b5-86f4-4c46-b853-0503796769de`, "Central Johannesburg Darts Association"). |
| **Auth** | `sb.auth` email + password. Roles: `admin` (in `app_admins` **or** the `ADMIN_EMAILS` list) · `player` (auth user linked to a `players` row via `auth_user_id`) · `viewer` (signed in, not linked) · `guest`. Resolved by `resolveIdentity()` on every sign-in / session restore. Sign-up is invite-only — the email must already sit on an unclaimed `players` row. |
| **Repo** | `main` only, direct commits. `index.html`, `README.md`, this file. Commits co-authored by Claude. |
| **Reference IDs** | Season "2026 Season" `2ca00843-…`; game type 501 `95b17ec3-…`; space `00ca89b5-…`. Night-type IDs in §14.3. |

The multi-tenant platform ("The Darts Nexus") owns many other tables in this same
database (`spaces`, `hub_settings`, `subscriptions`, `revenue`, `opens_*`, `teams`,
`competitions`, `activity_logs`, `app_state`, …). **The CJDA app does not read or write
those** — ignore them unless working on the platform itself.

**Multi-tenant caveat:** every CJDA table (`players`, `nights`, `results`, …) is CJDA-only
data (all rows `space_id = 00ca89b5-…`); other clubs like **Key West** have their own schema
(the `app_state` jsonb-blob pattern). BUT `auth.users` is **shared platform-wide** — one
login can belong to several apps. So admin identity and player-linking are **space-scoped**:
`app_admins` is keyed `(user_id, space_id)`, `is_space_admin(space_id)` is what the RLS
policies call, and `link_my_player(p_space_id)` / `signUp()` only match `players` rows in the
CJDA space. `fishontackle13@gmail.com` is a **Key West** admin and must never appear in CJDA's
`app_admins` or `ADMIN_EMAILS`.

---

## 14.1 Repository / File Structure

There is no `src/`. The app is one HTML file. Logical "modules" are comment-banded
sections of the inline script. Approximate map (line numbers drift — search by name):

| Section | ~Lines | Responsibility |
|---|---|---|
| Config + Supabase init | 122–140 | `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SPACE_ID`, `ADMIN_EMAILS`, `isAdminEmail()`, `sb` client |
| App state | 141–190 | the `state` object (see §14.2) |
| Utilities | 190–215 | `isAdmin`, `showToast`, `formatDate`, `ordinal` |
| Auth | 214–300 | `resolveIdentity` (role + linked player), `signIn`, `signUp` (invite-gated), `requestPasswordReset`, `signOut`, `isAdmin`/`isPlayer`/`canEditProfile` |
| Data loading | 248–276 | `loadAllData()` — one bulk fetch of every table the app uses |
| Competition helpers | 309–396 | `isCompNight`, `getCompNights`, `isTeamComp`, `compTeamsFor`, `compMatchesFor`, `getDoublesStandings`, `getDoublesWinner`, `getCompStandings` |
| GDF attendance | 402–592 | `fmtGdfDate`, `renderGdfCalendar`, `toggleGdfAttendance`, `saveGdfEvent`, `deleteGdfEvent` |
| Night entry / setup | 593–1090 | `viewNight`, `reopenNight`, `openNightEntry`, `saveNight`, block builders, `generateBlocksFromRegister`, player picker widget, `slotPlayerId` |
| Block scoring | 1092–1195 | `nightMatchLegs`, `legColor`, `updateLeg`, accolade record helpers, `calcBlockStats`, `getBlockPositions` |
| DIDO | 1196–1552 | `nightIsDido`, `didoBlockStandings`, `didoQualifiers`, `didoSeeds`, `recomputeBracket`, `didoStateStandings`, `didoNightWinnerId`, `generateDidoPlayoffs`, `renderDidoPlayoffs` |
| Team competitions | 1553–1915 | `compHasResults`, `addCompTeam`/`removeCompTeam`/`setCompTeamPlayer`/`setCompTeamBonus`, `addCompSession`, `updateCompDates`, `updateCompMatch`, `addCompAccolade`, `setCompStatus`, `renderDoublesComp` |
| Night persistence | 1910–2035 | `submitNight`, `purgeNightBlockData`, `writeNightBlocks` (shared by submit + draft save) |
| Season calc | 2115–2300 | `getActiveSeasonId`, **`calcPlayerSeasonStats`**, `getStandings`, `getSeasonAccoladeLeaders`, `accoladeName`/`accoladeClass` |
| Player CRUD | 2307–2333 | `savePlayer`, `deletePlayer` |
| Render — screens | 2334–4030 | `renderAuth`, `renderHeader`, `renderHome`, `renderSeasonLog`, `renderPlayers`, **`renderPlayerProfile`**, `renderAccoladeGuide`, `renderNightSetup`, `renderScheduleAssistant`, `renderBlockEntry`, `renderCompetitions`, `renderConfig`, `renderModal` |
| Player profile | 2778–3152 | `openPlayerProfile`/`closePlayerProfile`, `startProfileEdit`/`saveProfileEdit`/`cancelProfileEdit`, `downscaleImage`, `uploadPlayerPhoto`/`removePlayerPhoto`, `playerSeasonRank`, `playerCompWins`, `playerGdfEvents`, `playerAccoladeBreakdown` |
| Main render + init | 4201–end | `render()` tab switch, `restoreAssistantUI`, bootstrap IIFE (session restore → `loadAllData`) |

**Verification tooling** (not in the repo — kept in the session scratchpad): a headless
Edge harness (`msedge --headless=new --dump-dom`) that stubs `window.supabase`, splices in
the inline script, and asserts against `render()` output. Used to test every UI change in
this file without a live browser.

---

## 14.2 Core Application Flow

```
Sign in ──▶ bootstrap IIFE ──▶ loadAllData()  (one bulk fetch, fills state.*)
                                    │
                                    ▼
                              render()  ── switch(state.tab) ──▶ one render<Screen>()
                                    │                              writes #root.innerHTML
   every onclick handler ──────────┘  mutates state → calls render() again
```

`state` (the important keys):

| Key | Meaning |
|---|---|
| `user`, `role` | auth identity; `role` ∈ `guest` \| `viewer` \| `admin` |
| `tab` | active screen: `home` \| `nightsetup` \| `seasonlog` \| `players` \| `accolades` \| `competitions` \| `config` |
| `players seasons nights nightTypes gameTypes results blocks blockPlayers accolades spaceConfig` | mirror of the DB, refreshed by `loadAllData()` |
| `nightAttendance nightPlayoffs compTeams compMatches gdfEvents gdfAttendance` | secondary tables |
| `activeNight` | night currently being entered/viewed; `.viewOnly` when read-only. When set, Night Setup / Competitions render the entry sheet instead of the list |
| `nightBlocks` | in-memory working copy of the block sheet `[{label,size,slots,legs,accolades,bonus,id}]`. **`slots[i]` is a scalar player-id for singles, an array for team formats** — unwrap with `slotPlayerId()` |
| `activeBlockIdx compSession` | which block / competition-session tab is showing |
| `profilePlayerId profileEdit profileDraft` | player-profile screen state |
| `playersTab` | Players tab category filter (`men` \| `ladies` \| `juniors`) |
| `gdfMonth` | selected month (`YYYY-MM`) in the GDF attendance list |
| `modal` | `{mode, data}` for the Add-Player / Create-Night / Add-GDF-Event / Create-Competition dialogs |
| `assistant` | Schedule Assistant panel (night register + random block draw) |

### The night submission workflow (the critical path)

```
Create Night (draft) ──▶ openNightEntry ──▶ assign players / enter legs / accolades
   │                                          │
   │              Save Draft ◀────────────────┤  writeNightBlocks('draft')  (recovery only)
   │                                          │
   └──▶ Submit Night ────────────────────────▶ purgeNightBlockData()  (delete any draft rows)
                                               writeNightBlocks('locked')
                                               nights.status = 'locked'
                                               loadAllData() → standings + Season Log recompute on next render
```

- **`writeNightBlocks(status)`** — for each in-memory block: insert `blocks` row → insert
  `block_players` (one per filled slot, `slot_index` + `sub_slot`) → insert `results` (one
  row per unordered pair, `home` = lower slot, legs from the matrix) → insert `accolades`
  (expanded: one row per 180/171/HC/L15/9-dart occurrence). A match with `0–0` legs is a
  **walkout** and is skipped by the points engine.
- **`purgeNightBlockData(nightId)`** deletes accolades → results → block_players → blocks
  for that night. Submit always purges first, so re-submitting never duplicates.
- **`reopenNight(nightId)`** (admin) flips a locked league night back to `draft` and its
  blocks to `open`, then re-opens the entry sheet. Season Log recalculates on the next submit.
- Standings and the Season Log are **never stored** — they are recomputed from `results` +
  `accolades` + `blocks` every render by `calcPlayerSeasonStats` / `getStandings`.

---

## 14.3 Core Database Tables

All CJDA tables carry `space_id` (always `SPACE_ID`) + `created_at`. RLS: see §14.6.

### Player & season reference

**`players`** — one row per person.
`dsa_number`, `id_number`, `first_name`*, `last_name`*, `category`* (`men` \| `ladies` \|
`junior_boy` \| `junior_girl`), `status` (`registered` \| `temp`, default `temp`), `email`,
`phone`, `nickname`, `photo_url`, `dart_brand`, `dart_model`, `dart_weight_g`, `dart_notes`,
`auth_user_id` (unique — links to `auth.users`; set by `link_my_player()`),
`base_selection_points`, `base_attendance_points`, `highest_close`, `base_career_180s`,
`base_career_171s`, `base_career_170s`, `base_career_hc_count`.
The `base_*` columns are pre-app career totals folded into the calculated figures.

**`app_admins`** — `(user_id, space_id)` PK. Per-space admins. Read policy: authenticated.
**No write policy** — changed only via SQL / service role. CJDA rows: Alexander (superadmin),
Wynie (TD). The platform super-user (`dartsnexus@outlook.com`) is *not* a row here — it's
handled by `is_platform_superadmin()` and is admin of every space. `is_space_admin()` also
honours a 2-address CJDA email allowlist (Alexander, Wynie) so a fresh admin works pre-seed.

**`seasons`** — `name`*, `start_date`*, `end_date`, `is_active`, plus **two independent date
ranges**: `season_points_start/end` (prize-giving) and `selection_points_start/end`
(national selection). Only one row is `is_active`. Current: SP `2025-11-12 → 2026-11-12`,
Sel `2026-06-01 → 2027-05-01`.

**`game_types`** — `501` only (`95b17ec3-…`), `starting_score` 501, `default_legs` 3.

**`night_types`** — flags that drive every engine:

| Name | id | `counts_match_points` | `contributes_attendance` | `has_block_play` |
|---|---|:-:|:-:|:-:|
| **Wednesday** | `f16ab311-0c17-4437-bd33-846e92076a20` | ✅ | ✅ | ✅ |
| **DIDO** | `1834b6d3-cd7a-4e96-a54a-d700dcccc8fe` | ✅ | ✅ | ✅ |
| Selected Doubles | `fed910d9-e1d3-4877-a3d9-3169613b38a5` | ❌ | ✅ | ✅ |
| Drawn Doubles | `e81f3c5d-e3bf-444d-a99d-45fe9a6d9561` | ❌ | ✅ | ✅ |
| Mixed Triples | `87b70999-8cab-4240-9289-c19a83132348` | ❌ | ✅ | ❌ |
| Mix Doubles | `737cf149-4b4c-4ea3-8b4b-43dfd845f0ca` | ❌ | ✅ | ✅ |
| Champion of Champions | `691aa40d-b635-4b70-97e8-3c6fe83c6827` | ❌ | ✅ | ❌ |
| Home Alone Cup | `d9a3c1fa-e1f9-4e21-966b-06619e59dd56` | ❌ | ✅ | ✅ |
| Open Competition | `ff61dd63-9388-47d6-b5c9-2475b54935a8` | ❌ | ✅ | ❌ |

`counts_match_points = true` ⇒ **league night** (Wednesday + DIDO). Everything else is a
**competition** (`isCompNight()` = `!counts_match_points`).

### Night play

**`nights`** — `season_id`, `night_type_id`, `date`* , `status` (`draft` \| `submitted` \|
`locked`), `notes` (team-comp section label — keep short, e.g. "Men"/"Ladies"), `end_date`
(multi-day comps), `comp_sessions` (smallint, default 1).

**`blocks`** — `night_id`, `game_type_id`, `block_label` ("Block 1"…), `size` (**CHECK 3–8**),
`bonus` (per-player points added to everyone in the block), `status` (`open` \| `submitted` \|
`locked`).

**`block_players`** — `block_id`, `player_id`, `slot_index` (sheet row − 1), `sub_slot`
(0 for singles; 0/1/2 for team formats).

**`results`** — `block_id`, `night_id`, `season_id`, `home_player_id`, `away_player_id`,
`home_legs`, `away_legs`. One row per unordered pair per block. `home` = the lower slot.
`0–0` = walkout (ignored by scoring).

**`accolades`** — `player_id`, `night_id`, `season_id`, `accolade_type`
(`180` \| `171` \| `170_close` \| `high_close` \| `15_dart` \| `9_darter` \| `5_bulls` \|
`6_bulls`), `value` (checkout value for HC/170; dart count for 15-dart; else 1). One row per
occurrence. **Counted regardless of night type** — competition accolades feed player stats.

### DIDO playoffs

**`night_playoff_matches`** — `night_id`, `round` (1 = QF, 2 = SF, 3 = Final),
`match_number`, `slot_a_seed`/`slot_b_seed` (1–8, null for SF/Final), `player_a_id`/
`player_b_id`, `a_legs`/`b_legs`, `winner_id`, `best_of` (default 1). **Never contributes
points** — separate from `results`.

### Team competitions (Selected/Drawn Doubles, Mixed Triples, Mix Doubles)

**`comp_teams`** — `night_id`, `seed` (1..N, grid order), `player1_id`, `player2_id`,
`player3_id` (null = doubles), `bonus`.

**`comp_matches`** — `night_id`, `session` (1,2,3… — multi-night), `team_a_seed` <
`team_b_seed`, `a_legs`, `b_legs`. One row per pairing per session.

### Attendance

**`night_attendance`** — `night_id`, `player_id`. The register on the night-entry screen;
feeds the Season Log "Att Pts" column (via `contributes_attendance` night types).

**`gdf_events`** — `name`*, `date`*, `end_date`, `event_type` (**CHECK** `wednesday` \|
`johannesburg` \| `gauteng`), `notes`.
**`gdf_attendance`** — `event_id`, `player_id`. **Standalone tracker — does NOT feed the
Season Log.** UI: `renderGdfCalendar` (month-tab list, admin adds attendees per event).

### Legacy / unused by CJDA

`competitions`, `teams`, `opens_*`, `accolades_config`, `space_accolades` — platform
scaffolding, not wired into the CJDA screens.

---

## 14.4 Database Relationship Map

```
seasons ─┬─ nights ─┬─ blocks ─┬─ block_players ── players
         │          │          └─ results ─────────┐
         │          ├─ accolades ──────────────────┤
         │          ├─ night_attendance ───────────┤
         │          ├─ night_playoff_matches ──────┤   (DIDO only, no points)
         │          ├─ comp_teams ─────────────────┤   (team comps only)
         │          └─ comp_matches                │
         └─ results.season_id, accolades.season_id │
                                                   │
players ───────────────────────────────────────────┘
        └─ gdf_attendance ── gdf_events            (standalone, no points)

game_types ── blocks.game_type_id
night_types ── nights.night_type_id               (flags drive all scoring)
space_config, hub_settings                         (theme / club config)
```

Cascade deletes are **not** relied on — `deleteNight`, `deleteCompetition`,
`purgeNightBlockData` delete children explicitly in order.

---

## 14.5 Critical Functions

### `calcPlayerSeasonStats(playerId)` → season/career figures for one player

Pure (reads `state`, writes nothing). Wrapped in try/catch → zeroed object on error.

1. Find every night the player appears in (via `block_players` → `blocks` → `night_id`).
2. **Season-points nights** = those with `counts_match_points` **and** `date ∈
   [season_points_start, season_points_end]`. **Selection-points nights** = same but the
   selection date range. **Attendance nights** = `contributes_attendance` in the SP range.
3. `calcPoints(nightSet)` = Σ over the player's `results` in those nights of
   `myLegs` (legs won) `+ 2 ×` (games won), plus each distinct block's `bonus`. Walkouts
   (`0–0`) skipped.
4. Accolades YTD = the player's `accolades` whose night falls in the SP range, tallied by
   type. `highestClose` = max HC/170 `value` (YTD) then `max(…, players.highest_close)` →
   returned as the **career** best.
5. Career accolade counts = `base_career_* + YTD`.

**Returns:** `seasonPoints` ( = SP total **+ `players.base_selection_points`** ),
`selectionPoints` ( = selection total, no base ), `attendancePoints`, `thisWeek`,
`legsWon`, `gamesWon`, `gamesPlayed`, `acc180ytd/career`, `acc171ytd/career`,
`acc170ytd/career`, `accHCytd/career`, `highestClose`, `acc15ytd`, `accolades[]`.

> ⚠ `base_selection_points` is added to **seasonPoints**, and `base_attendance_points`
> appears unused. See §14.8.

### `getStandings(category)` → ranked list for a Season Log tab

`players` in `category` → `+ calcPlayerSeasonStats` → **drop anyone with zero activity**
(no season/selection/attendance points, no accolades, no `base_selection_points`) → sort
`seasonPoints` desc, then `last_name`. Index + 1 = the player's rank. Juniors tab =
`junior_boy` + `junior_girl` merged and re-sorted.

### `getDoublesStandings(nightId)` → team-competition table

Per `comp_teams` row, aggregate `comp_matches` across all sessions:
`total = legsWon + gamesWon×2 + team.bonus`. Sort `total` desc → `legsWon` desc → `seed`.
`getDoublesWinner` = `standings[0].name` when it has games played.

### `getBlockPositions(blockIdx)` (displayed block POS) vs `didoStateStandings` / `didoBlockStandings`

- **`getBlockPositions`** — sort by total points **only**; ties broken by slot/row order
  (stable sort). **No head-to-head.** This is what the block sheet shows.
- **`didoBlockStandings` / `didoStateStandings`** — add an H2H tie-break inside equal-total
  groups. Used **only** for DIDO playoff seeding and the DIDO night winner, never for the
  displayed Wednesday POS.
- Consequence for historical capture: to reproduce a paper sheet whose POS used H2H, insert
  `block_players` in the sheet's final-standings order.

### `submitNight()` / `writeNightBlocks()` / `purgeNightBlockData()`

See §14.2. `submitNight` is admin-only, confirms, purges, writes `locked`, sets
`nights.status`, reloads.

### `didoNightWinnerId(nightId)`

Playoff final `winner_id` if a bracket exists; else `null` if a bracket exists but the
final isn't done; else the top of `didoStateStandings` (the "no-playoff, take it by
standings" fallback).

### `playerCompWins(playerId)` (profile)

Outright winners only: winning `comp_teams` row of a locked team comp (from
`getDoublesStandings[0]`), DIDO champions (`getDidoChampions`), or a singles-comp
`night_playoff_matches` round-3 `winner_id`.

### Photo pipeline — `downscaleImage` + `uploadPlayerPhoto`

`downscaleImage(file, 512, 0.82)` draws to a canvas and `toBlob('image/jpeg')`.
`uploadPlayerPhoto` uploads to Storage bucket **`player-photos`** as `{playerId}.jpg`
(`upsert`), then persists via `setPhotoUrl()` — direct `players.photo_url` update for admins,
`set_my_photo()` RPC for a player editing their own. URL is public + `?v=<ts>` cache-bust.

### Auth / self-service — `resolveIdentity` + the SECURITY DEFINER RPCs

- **`resolveIdentity()`** (client) — after auth: admin? (`is_app_admin` RPC or `ADMIN_EMAILS`).
  Then find the linked `players` row (`auth_user_id = auth.uid()`), or `link_my_player()` to
  link by email. Sets `state.role` + `state.myPlayerId`.
- **`is_platform_superadmin()`** — `auth.uid() = <dartsnexus uid>` OR jwt email
  `dartsnexus@outlook.com`. The platform owner's account — admin of **every** space, always.
- **`is_space_admin(p_space_id)`** — `is_platform_superadmin()` OR `exists(app_admins where
  user_id = auth.uid() and space_id = p_space_id)` OR (CJDA space + email in the 2-address CJDA
  allowlist: Alexander, Wynie). Every CJDA write policy is `USING (is_space_admin(space_id))` —
  a per-space admin can only write that space's rows; the platform super-user writes anything.
  `is_app_admin()` is a back-compat wrapper = `is_space_admin('00ca89b5-…')`.
- **`link_my_player(p_space_id)`** — one-time: links the caller to an unclaimed `players` row
  **in that space** whose `email` matches the JWT email. Raises `NO_PLAYER_FOR_EMAIL` if none.
- **`save_my_profile(patch jsonb)`** — updates **only** `nickname / phone / email / dart_brand /
  dart_model / dart_weight_g / dart_notes` on the caller's own row. A key absent from `patch` is
  left unchanged; an empty-string value clears it. Never category / status / DSA / name / points.
- **`set_my_photo(url text)`** — sets `photo_url` on the caller's own row.
- `signUp()` (client) is gated: the email must be on an unclaimed `players` row or it's refused.

---

## 14.6 Critical Business Assumptions

Do not change without written approval — each drives multiple engines.

1. **A league night and a competition night never share a date.** Enforced by the
   create-night / capture guards, *not* a DB constraint.
2. **DIDO playoff rounds award no season and no selection points.** They live in
   `night_playoff_matches`, which the points engine never reads.
3. **Only DIDO's block phase contributes match points.**
4. **Accolades always count toward player statistics**, regardless of night type
   (league or competition).
5. **Team competitions (Selected/Drawn Doubles, Mixed Triples, Mix Doubles) do NOT feed
   the Season Log** — only the per-player accolades recorded on those nights do.
6. **GDF attendance does not feed the Season Log.** "Att Pts" comes from `night_attendance`
   on `contributes_attendance` nights. The GDF calendar is a standalone federation tracker.
7. **Season points and selection points use independent date ranges** that may overlap.
   Same formula: `legsWon + gamesWon×2 + blockBonus`.
8. **Match formats:** Wednesday = 3 legs/match, most wins (valid 3-0, 2-1). DIDO 4+ player
   block = 1 leg. DIDO 3-player block = best-of-3. Team comps = games-won based, per
   `comp_matches`.
9. **Block POS shown on the sheet has no head-to-head tie-break** (total points then row
   order). H2H exists only for DIDO seeding / DIDO winner.
10. **Standings and the Season Log are always recomputed, never stored.**
11. **Slot model:** a block slot is a scalar player-id for singles and an array for team
    formats. Always read it through `slotPlayerId()`.
12. **Writes are DB-enforced, admin-only, and space-scoped.** Every CJDA table: `Public read`
    (anon SELECT) + `<table> admin write` (`USING (is_space_admin(space_id))`). `anon` has no
    write grant at all. A CJDA admin can only write CJDA rows; a Key West / other-space admin
    gets 0 rows. A **player** can change nothing except the seven self-service fields on
    **their own** `players` row, only through `save_my_profile()` / `set_my_photo()`
    (`SECURITY DEFINER`, keyed on `auth.uid()`). The `player-photos` bucket write policy
    restricts a non-admin to their own `{playerId}.jpg`. Client role checks (`isAdmin`,
    `isPlayer`, `canEditProfile`) are the UX layer; the DB is the real gate.
    (Platform tables outside CJDA — `opens_*`, `accolades_config`, `space_accolades` — still
    carry the old blanket `Auth write` policies; out of scope here.)
13. **Attribution email** `Alexander.Kloppers@outlook.com` is for commit authorship only.

---

## 14.7 Architectural Decision Log

Format: **Date — Decision — Reason — Impact.**

### 2026-08-14 — Split Season Points and Selection Points date ranges
Both rankings run independently and can overlap. → `seasons` gained
`season_points_start/end` + `selection_points_start/end`; `calcPlayerSeasonStats` computes
two night sets.

### 2026-09-01 — Night register + DIDO knockout as first-class data
Attendance needed for selection; DIDO playoffs must never leak into points. → migrations
`add_night_attendance`, `add_night_playoff_matches`. `night_playoff_matches` is deliberately
disjoint from `results`.

### 2026-09-02 — Team competitions get their own tables
`results` is player-vs-player (single uuids); pair-vs-pair could not be stored. → migration
`add_comp_teams` (`comp_teams` + `comp_matches`). Competition entry routes
`isTeamComp` → `renderDoublesComp`, else `renderBlockEntry`. `nights` gained `end_date`
(multi-day) + `comp_sessions` (multi-night round-robin).

### 2026-09-02 — Night-entry screen rebuilt ("Key West" layout)
Usability. → new block sheet, draft auto-save, prominent Submit, Schedule Assistant.
**Side effect / bug:** `openNightEntry` rebuilt every slot as `Array(pps).fill(null)`, so
singles slots became `["<uuid>"]` and `results.home_player_id` received an array → submit
crashed (`invalid input syntax for type uuid: "[\"…\"]"`). Fixed 2026-09-03 with
`slotPlayerId()` unwrap + keeping singles slots scalar in `setBlockSlot`.

### 2026-09-03 — "Drawn Doubles" is its own night type
Initially assumed identical to Selected Doubles; the club corrected it. → new `night_types`
row `Drawn Doubles` (`e81f3c5d-…`, `counts_match_points=false`); the 27 May night repointed.

### 2026-09-03 — Tournament Director granted admin
`wynandcarelse123@gmail.com` added to `ADMIN_EMAILS`. Also introduced `isAdminEmail()`
(trim + lowercase + includes) replacing raw `ADMIN_EMAILS.includes`.

### 2026-09-03 — `reopenNight()` for locked league nights
No way to fix a locked Wednesday night (only competitions had Reopen). A pre-`slotPlayerId`
submit had lost blocks mid-write on the 2026-09-02 night. → `reopenNight` + "Reopen"
buttons in Night History.

### 2026-09-03 — Home screen: per-player accolades, Season + Selection points
Season Standings last column GP → Selection Points. "Legendary Accolades" regrouped one
block per player (season-only, sorted by most). `getLegendaryAccolades` →
`getSeasonAccoladeLeaders`.

### 2026-09-03 — Block leg-cell colours restored to value scheme
The Key West redesign had swapped to a win/loss heuristic. Restored: `legColor(val)` →
0 red / 1 yellow / 2 green / 3+ purple, with a legend under the sheet.

### 2026-09-03 — DIDO winner falls back to block standings when there is no playoff
"Stephen still won it." → `didoNightWinnerId` / `didoStateStandings` (H2H-aware) +
`getDidoChampions` iterates DIDO nights; Home shout-out + `renderPlayers` badge use it.

### 2026-09-03 — Night Setup lists league nights only
Locked competition nights were showing in Night History and "View" hung on "Loading…"
(`renderBlockEntry` needs blocks; comp nights have none). → both lists filter
`!isCompNight`; the active-night guard routes team comps to `renderDoublesComp`.
Commit `8a79ef2`.

### 2026-09-03 — GDF attendance redesigned → month-grouped event list
Was a players × events checkbox matrix. → `renderGdfCalendar` builds a `.nav-tabs` row of
month tabs (`state.gdfMonth`); each event is a card with removable attendee chips + an
"add attendee" `<select>` (admin) + delete. `gdf_events` gained an optional End Date on the
Add form. **Not wired to scoring.** Commits `92d3cc2`, `dd07d89`.

### 2026-09-03 — Players tab: one table, category tabs, ranking column
Was separate cards per category. → single table, Men / Ladies / Juniors tabs
(`state.playersTab`); columns Rank · DSA · Nick Name · First · Surname · Status · Cellphone
· `Profile →`. Rank = position in `getStandings` for that tab; unranked show "—".
New `players.nickname` column (migration `add_players_nickname`). Commit `a6fe4ec`.

### 2026-09-03 — Full player profile page; Edit Player modal retired
The `Profile →` button opens `renderPlayerProfile` (sets `state.profilePlayerId`;
`renderPlayers` returns the profile while it's set). Sections: photo, summary tiles,
details, career/season accolade table, darts setup, competition wins (outright only), GDF
events. **Editing moved onto the profile** (inline `startProfileEdit` → `saveProfileEdit`
→ `savePlayer`); `openEditPlayer` now redirects there; delete moved onto the profile.
New: `players.photo_url` + `dart_brand`/`dart_model`/`dart_weight_g`/`dart_notes`
(migration `player_profile_fields`); public Storage bucket **`player-photos`** (public read
policy + authenticated write policy, 3 MB, jpeg/png/webp). Photos downscaled to 512 px
client-side. Commit `35abdca`. Design mockup:
`https://claude.ai/code/artifact/ab0d2775-a76c-4200-bdcb-41b50387d69f`.

### 2026-09-03 — Self-registering players: linking, `player` role, RLS lockdown
Players will register in the app. Hard rule: **a player may edit nothing but their own player
profile.** → `players.auth_user_id` + `app_admins` table + `is_app_admin()`. All 16 CJDA table
write policies rebuilt from `Auth write (true)` to `<t> admin write (is_app_admin())` — public
read unchanged. Player self-service via three `SECURITY DEFINER` RPCs
(`link_my_player`, `save_my_profile`, `set_my_photo`); `player-photos` storage write scoped to
own file. Client: `state.role` gains `player`; `state.myPlayerId`; `resolveIdentity()` replaces
the inline `isAdminEmail` role assignment on sign-in + session restore; `signUp()` invite-gated;
`canEditProfile(id)`; a top-right **account menu** (`renderAccountMenu` — avatar → Edit my
profile / theme / refresh / sign out) replaces the loose header buttons; the player profile
renders name/DSA/category/status read-only for a player editing their own. Migrations
`player_auth_linking`, `rls_admin_write_lockdown`, `player_self_service_rpcs` + an `execute_sql`
for the storage policy. Verified: anon = no write grant; linked non-admin = 0 rows on every
direct table write, `save_my_profile` touches only whitelisted fields; admin = full write.

### 2026-09-03 — Admin identity + linking made space-scoped
`auth.users` is shared across the Darts Nexus platform (Key West etc.). The first cut had a
global `app_admins` + email allowlist that wrongly included `fishontackle13@gmail.com` (a Key
West admin). → `app_admins` gained `space_id` (PK `(user_id, space_id)`); `is_space_admin(uuid)`
replaces `is_app_admin()` in every policy (`USING (is_space_admin(space_id))`); `link_my_player`
takes `p_space_id`; `signUp()` + `resolveIdentity()` filter `players` by `SPACE_ID`. CJDA
per-space admins = Alexander, Wynie. `dartsnexus@outlook.com` is the **platform super-user**
(`is_platform_superadmin()`, `OR`-ed into `is_space_admin`) — admin of every space, always;
not an `app_admins` row. Migrations `space_scope_admins`, `link_my_player_space_scoped`,
`platform_superadmin`. Verified: other-space admin → 0 rows on CJDA writes; `fishontackle13` →
not admin; `dartsnexus` → admin of CJDA and any other space.

### 2026-09-03 — Profile Save writes an explicit field whitelist
`saveProfileEdit` used to `savePlayer({...profileDraft})`, and `savePlayer`'s update path
spreads every key of the object. A photo uploaded *after* clicking Edit updated
`players.photo_url` + reloaded `state.players`, but not the in-memory `profileDraft`, so the
next Save wrote the stale `photo_url: null` back. → `saveProfileEdit` now `update()`s a
hand-built patch of only the form's fields (never `photo_url`, `base_*`, `id_number`,
timestamps). New `miniAvatar(p, size)` helper (photo `<img>` over an initials disc, removes
itself on error) — used on the profile hero and Home Season Standings. Home Season Standings
name shows `nickname || first_name` (no surname); the Season Log tab keeps full names.
Commit `5b9a3c9`.

### Migration ledger (Supabase, CJDA-relevant)

```
20260901205955  add_night_attendance
20260901220151  add_night_playoff_matches
20260902153310  add_comp_teams
20260902155438  nights_end_date_and_comp_sessions
20260903184031  add_players_nickname
20260903191343  player_profile_fields
20260903......  player_auth_linking        (players.auth_user_id, app_admins, is_app_admin())
20260903......  rls_admin_write_lockdown   (all 16 CJDA tables: admin-only writes)
20260903......  player_self_service_rpcs   (link_my_player, save_my_profile, set_my_photo)
20260903......  space_scope_admins         (app_admins.space_id, is_space_admin(uuid), policies re-pointed)
20260903......  link_my_player_space_scoped
20260903......  platform_superadmin        (is_platform_superadmin() — dartsnexus@outlook.com, admin everywhere)
+ execute_sql (not migrations):  Drawn Doubles night_types row · player-photos bucket + policies
                                 (bucket write policy re-scoped to admin-or-own after the lockdown)
```

---

## 14.8 Known Issues Register

| Issue | Impact | Workaround / Status |
|---|---|---|
| `base_selection_points` is added to **seasonPoints** (not selectionPoints); `base_attendance_points` looks unused. | Low | **Open** — confirm intent with the club before "fixing". |
| `night_types.contributes_selection` is defined but unused — `calcPlayerSeasonStats` keys selection nights off `counts_match_points`. | Low | **Open** — harmless today (both league types have it `true`); revisit if a selection-only night type is ever wanted. |
| ~~RLS lets any authenticated user write any table~~ | — | **Resolved 2026-09-03** — all CJDA tables are admin-only writes (`is_app_admin()`); players get self-service RPCs only. Platform tables (`opens_*` etc.) still open — out of scope. |
| ~~`fishontackle13@gmail.com` in the CJDA admin list~~ | — | **Resolved 2026-09-03** — removed; they're a Key West admin. Admin identity is now space-scoped. |
| ~19 early auth accounts exist with no linked player (public sign-up was open before the invite gate). | Low | **Open** — they're plain `viewer`s and can't write anything. They auto-link if the TD later puts their email on a `players` row. |
| Missing Wednesday league nights: **2026-01-28, 2026-03-25, 2026-06-03, 2026-07-15**. | Low | **Open** — unconfirmed whether played. All other Wednesdays 12 Nov 2025 → 2 Sep 2026 are captured; Dec/early-Jan + 1 Apr (Easter) were scheduled breaks. |
| Perfect 3-way cycle blocks: the app shows 1/2/3, the paper sheet marks all Pos 1. | Cosmetic | **Won't fix** — `getBlockPositions` cannot represent a tie; totals are identical so nothing downstream is wrong. |
| Singles competitions (CoC / Home Alone / Open) have no data and only playoff-final winner detection in `playerCompWins`. | Low | **Open** — revisit if those competitions are ever run. |
| 2026-09-02 night lost blocks 4 & 5 during a pre-`slotPlayerId` submit crash. | — | **Resolved** — reopened + re-entered; `slotPlayerId` fix shipped. |
| Profile Save wiped `players.photo_url` (stale-draft spread by `savePlayer`). | — | **Resolved** `5b9a3c9` — `saveProfileEdit` now patches a field whitelist. |
| Home "Competition Results" used to hide older competitions (grouped by type, latest only). | — | **Resolved** `a25ca67` — now one column per completed competition. |

---

## 14.9 Lessons Learned

### RLS / grants incident (original)
Data appeared empty everywhere → missing Supabase grants. Explicit `GRANT` statements added
for every role. **Lesson:** check permissions before debugging front-end logic.

### The uuid-array slot bug (Key West redesign)
A layout refactor rebuilt all block slots as arrays, so singles `results.home_player_id`
got `["<uuid>"]` and Postgres rejected it on submit. **Lesson:** the block slot data model
is polymorphic (scalar vs array) — every read must go through `slotPlayerId()`, and any
code that *builds* slots must keep singles scalar.

### `execute_sql` DO-block batches
When capturing historical nights via `mcp__…__execute_sql`, only the **last** `SELECT` in
the batch returns rows, and **a failing trailing SELECT rolls back the whole batch**
(including the DO block). Run the DML and the verification SELECT as separate statements.
Resolve UUIDs by lookup (`select id … where lower(first_name)=…`) rather than pasting them —
hand-transcribed UUIDs caused several failed inserts.

### Historical score-sheet capture discipline
Always (a) parse the matrix and reconcile every GW / LW / total on the sheet with a script
**before** inserting, (b) show the club the name→player matches, (c) re-derive standings
with a SELECT **after** inserting and confirm they match. Sheets contain arithmetic slips —
the app recomputes from the matrix, so enter the matrix, not the printed totals, and flag
the slip.

### Headless-Edge render harness
Every change to `index.html` is verified by splicing the inline script into a stub page
(`window.supabase` faked), calling the `render*()` functions with hand-built `state`, and
asserting on the HTML string — no live browser, no Supabase round-trip. The inline script
is lines 123 → `</script>−1`; getting that range wrong yields "Illegal return statement".

---

## 14.10 Domain Glossary

**Season Points** — prize-giving standings. `Σ(legsWon + gamesWon×2 + blockBonus)` over
`counts_match_points` nights within the season-points date range (+ `base_selection_points`).

**Selection Points** — national-team selection. Same formula, over the selection date range.

**Attendance Points** — count of `contributes_attendance` nights the player played in the
season-points range (from `results` presence, plus the `night_attendance` register).

**Block** — a round-robin group of 3–8 players on a league night. Everyone plays everyone;
`bonus` is added to every player in the block.

**Block bonus** — flat points added to all players in a block (e.g. a 3-player block on a
short night).

**Walkout** — a match recorded `0–0`; ignored by the points engine.

**DIDO** — "Double In / Double Out". A competition night = **block phase** (contributes
match points) + **playoff phase** (`night_playoff_matches`, 8-seed knockout, no points).
4+ player blocks play 1 leg; 3-player blocks best-of-3.

**Selected Doubles / Drawn Doubles / Mixed Triples / Mix Doubles** — team round-robin club
competitions (`comp_teams` + `comp_matches`), sometimes over several nights
(`comp_sessions`). Team total = `legsWon + gamesWon×2 + bonus`. Do not feed the Season Log.
"Selected" = players choose partners; "Drawn" = partners randomly drawn.

**Accolade** — an honorary achievement (180, 171, 170 Big Fish, 9 Darter, High Close ≥ 90,
< 15 Dart, 5/6 Bulls). Recorded per occurrence in `accolades`. Never worth league points,
but always counted in player statistics and on the Home "Legendary Accolades" card.

**High Close (H/C)** — a checkout above `space_config.min_high_close` (90). Stored with its
value; `170_close` ("Big Fish") is its own type.

**Career vs YTD / Season** — Career = `base_career_* + this season`. YTD/Season = within the
active season's points range.

**GDF Calendar** — attendance register for Gauteng Darts Federation events
(`event_type` `johannesburg` / `gauteng` / `wednesday`). **Not** a scoring input.

**Registered vs Temp player** — `status`. Temp = a guest/fill-in not on the official roster;
still fully scored. The Season Log shows registered players only; the Players tab shows both.

**Roles** — `admin` (runs the club: all writes) · `player` (a registered member whose login is
linked to their `players` row: edits only their own profile's self-service fields) · `viewer`
(signed in, no linked player: read-only) · `guest` (not signed in: read-only). "Linked" =
`players.auth_user_id` matches the auth user, set by the TD adding the member's email then the
member signing up / logging in (`link_my_player`).

**Space** — the multi-tenant unit on the Darts Nexus platform. CJDA is one space
(`SPACE_ID`). Every CJDA row is scoped by `space_id`.

---

## 14.11 Target State

A developer (or AI session) resuming cold should be able to, from this file alone:
understand the business rules, the schema, where each feature lives in `index.html`, the
night-submission data flow, the historical decisions, and the open issues — and fix a bug
without extra onboarding.

**Current:** Business 9/10 · Technical 9/10.
**Remaining for 10/10 technical:** a generated ER diagram; per-function input/output tables
for the render layer; a documented deployment/hosting path for `index.html`; and resolution
of the `base_*` points questions in §14.8.
