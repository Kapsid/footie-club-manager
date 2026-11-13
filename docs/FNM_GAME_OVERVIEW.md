# FNM Game Overview

Football Nations Manager (FNM) is an offline, endless international football manager for phone and tablet (Flutter, build 1.1.1 (49)). You manage a national team across four-year cycles in a world of 209 nations. This document describes each system as a product, then what it would become in the club manager Footie Soccer Manager (FSM). Sources: `lib/domain/services/*`, `lib/features/*`, the owner's project notes. Numbers are from code or notes; see "Unverified" at the end.

## 1. Core loop

- **Hub + Continue**: the hub shows the next thing that needs attention and one Continue button. Continue advances the calendar to the next *event* that needs the manager: a draw, a call-up window, a match, a press conference, the budget, a ceremony, the cycle rollover. Everything else is simulated in between. Events are an ordered queue (`hub_event.dart`): ceremonies, press, finals watch, budget/sponsor, etc.
- **A cycle is four years**, built around the World Championship (cycle 0 ends 2030, then 2034...). Rough shape: friendlies and warm-ups, qualifiers (draws, groups of 4 per confederation), the continental cup (live for all six confederations), the Nations Cup ladder, the World Championship finals, then the **cycle rollover** (budget, board verdict, retirements, new intake, new sponsor round, new qualifying draw).
- **Before each match window**: call-up the squad (min 16), pick formation, XI, roles, set-piece takers, then play it live, watch it, or skip it (skipped matches use the fast path with parity to the live engine).
- **After**: results, news, ratings, press and the Y feed react.
- Club version: Continue becomes the weekly heartbeat (matchday, transfer window, board meeting, press). Same event queue, much higher event frequency, with fast-forward batching for the rest of the world.

## 2. Competitions

- **Friendlies**: scheduled in windows, choose opponents, warm-up offers before a tournament, a few more substitutions allowed, cards rarer.
- **World Championship qualifiers**: confederation groups of 4 (round code `CQ` for continental), intercontinental play-offs, tiered qualification (32 finals berths in the original format; the finals later grew to 48 teams). Animated "balls from pots" draw ceremonies for every draw. Draw ceremonies read stored groups, never re-run a draw.
- **World Championship finals**: pot draw, groups, knockout R32 to final plus third place place-off, ties always resolved (extra time, shootout). Neutral venues; the host keeps its crowd. Never call it "World Cup" in English copy (trademark). Stored competition names stay canonical English and are translated at display.
- **Continental cups**: all six confederations run live and in the background; qualifiers then group stage then knockouts.
- **Nations Cups**: Europe (ladders), North America (ladders merging tiers under 4), Africa (zones plus Finals Eight). Asia, South America and Oceania have none.
- **Hosts**: the host is drawn from a weighted shortlist (square-law weighting over the top 8 nations, any confederation except the last host's, no last-3 hosts). Players can **bid to host** (host bidding, 1.0.7); host cities are the biggest cities of the country.
- **Background tournaments** (`TournamentSim`) run for every competition the manager is not in.
- Club version: league pyramids per country (promotion/relegation), domestic cups, continental club cups, a super cup. Reuse the draw ceremony, group table, bracket, round schedule and background tournament simulation widgets and services.

## 3. Federation (the "club" of a national team)

- **Budget levels**: after each cycle the federation invests income in departments: Youth, Commercial, Medical, Naturalisation, Board Relations. Money is spent in whole levels of EUR 4M, 10 levels per department (Medical 6, because injury reduction hits a 70% floor). Leftover under one level stays banked. Effects are computed from euros so old saves keep what they bought.
- **Finances**: opening cash scales with world standing; cycle income from sponsors, results and commercial level; staff wages deducted.
- **General sponsor**: one round per cycle before the budget, 3 offers (1 risky + 2 plain, different deal shapes, never last cycle's brands) out of 32 fictional brands. Cycle-sized deals: plain EUR 2.5M..10M by ranking points; risky 2x + EUR 2.5M; marketing deals paid back 1.6x at cycle close. Board and morale reactions attach to the deal.
- **Staff** (`manager/staff.dart`): 3 roles (fitness coach, scout, and one more) in tiers none/basic/good/elite. Fitness coach reduces injuries, scout gives an opponent dossier bonus.
- **Manager skills**: 4 skills (man-management, tactical, youth development, negotiation) on a 1..20 scale, starting at 5 (exactly neutral), points earned 2 per cycle + 1 per trophy. Training focus (4 options).
- **National stadiums**: each nation has a national stadium and other grounds with capacity; a stadium map; the manager can see and use them (with the host rules).
- **Naturalisation**: recruit eligible foreign-born players, a dual-nationality dilemma, department-gated.
- Club version: maps almost 1:1 onto club finances (wages, transfer budget, facilities, youth academy, sponsors, stadium), but weekly cash flow and contracts replace the cycle budget. Levels-of-investment is a good UI pattern to keep (each step must visibly change an effect).

## 4. Squad, call-ups and tactics

- **Player model**: 3 stats (physical, technical, stamina), derived overall, hidden potential derived from the id. 4,807 named seed players plus procedural newgens by nation. See FNM_MATCH_ENGINE.md.
- **Call-ups**: the manager nominates a squad from the nation's eligible pool (age 15+ for academy picks); drafts, call-up advice, injury cover, grievances (unhappy omitted players), camp scandals (switched off in 1.0.8), international retirement.
- **Condition**: form (recent ratings), fatigue (recent minutes), team morale (results), captain bonus, all derived, hidden shift in the engine; shown through a pulse/legend UI and `StrengthFactors`.
- **Tactics**: 19 formations, 6 dials, 7 playstyles, roles, full-back arrows, slot switches, throw-ins, penalty/dead-ball takers, tactic library and presets, in-match changes and team talks, opponent scouting.
- **Training camp**: pre-tournament camps and base camps (co-host base camps).
- Club version: the same, but squad is 25-40 contracted players, plus lineups per competition, rotation, set instructions per match, training schedules.

## 5. Board and expectations

- **Board objectives** per cycle (reach QF, qualify for finals...), on a 2 (qualify) to 7 (champions) scale; objectives can go below "qualify". Judged at rollover; brief locks in the moment it is reached (a finals draw counts as qualified, locked-in finals losses cost -1 each).
- **Board satisfaction** 0..100: recent form moves it gently, tournament results heavily; carry-over between cycles 0.85.
- Drives the **job market**: sacking, offers from other nations, job dossiers, **unemployed start**, manager history and stints.
- Club version: season objectives per club and per competition, confidence meter, sacking, vacancies, job offers by reputation.

## 6. Press, Y and news

- **Press conferences** (`press/`): rare and fact driven. Triggers: heroic or humiliating result that matters, crisis (3 winless with 2 losses, or board < 40 after a non-win), or his "judge me on 3" broke. 45-day quiet, max 6 per cycle. Personas with derived stance, tone-banded copy. **Statements** (win next, job on the line, back player, blame player) are judged later (open/kept/broken/lapsed) and credibility is hidden. Effects land as morale/board deltas; a blamed player gets -2.
- **Y** (social feed, parody of a social network): fan accounts, pundits, local voices, tournaments posts; replies keyed by *mood* so one set of wordings serves every event; sentence parts assembled at render for localisation (Czech has seven cases). Trends, profile sheets, fan mood.
- **Inbox/news**: messages popup, intake report, squad development report, transfer report (cosmetic), youth star news, manager changes report, player of the year card, watch news.
- Club version: the same three layers are a strong identity (the world "has a voice"). Add transfer rumours, agents, club-specific fan accounts, pundits per league.

## 7. Youth pyramid U-13 to U-21

- Boys enter at U-13, grow through U-15, U-17 (a "watchlist" of U-21), reaching senior eligibility at 17+. The senior world is held still by a reconstruction: intake generates the player *as he will be at 17*, then subtracts childhood growth; aging adds it back over six years. Senior pool settles at ~126-130 per nation.
- `poolAt(nation, cycle, minAge)` is a pure derived function; default minAge 17 keeps world sim cost flat; the squad screen asks for 15.
- Youth department level and manager youth skill raise intake quality; golden generations; youth star news.
- Club version: this is the natural core (academy, loans, scouting). The lazy derived-pool trick should extend to thousands of clubs.

## 8. Clubs system (cosmetic)

- Every player has a club (fictional per-country names, 5 leagues tiers) as flavour: a deterministic function of id, rating, save seed, contract clock (deals 2-4 seasons, half of expiries are renewals, 6% break early). About 4.4-5.5 clubs per 19-season career. Home retention per country is data from real squads. Club form feeds condition. Club history card on player page, transfer window report.
- No persistence, no effect on the sim apart from club form.
- Club version: replaced by real clubs, real contracts, real transfers. The existing generated names, per-league tiers and standing bands can seed the world.

## 9. World managers

- Every AI nation has a fictional manager with a backdated spell, style and reputation; pool of free managers. A **decision engine** at each decision point (tournament settled, rollover): sackings, resignations, contracts, retirements, poaching, hiring; news items; relations with the player. Pure, deterministic from the career seed on its own stream.
- Club version: AI managers per club, tied to board confidence and the job market (the player competes with them for vacancies).

## 10. Ranking, records, achievements, campaigns

- **World ranking**: Elo-based (`ranking/elo.dart`), with a finals "placing" exchange (group exit 0 to champion 1 credit, scale `Elo.placement`) in addition to match results. Starting rankings can come from a world pack.
- **Records and history**: all-time records, record book, head-to-head and meetings, rivalries, legends hall of fame, Player of the Year and all-time XI, tournament awards and team of the tournament, nation vitrine (trophy cabinet).
- **Stats**: career stats foundation, team stats, overall history chart.
- **Achievements**: 28 defined (grep count) in categories; **challenges**: ~38 (grep count). Popups on unlock.
- **Campaigns**: hand-built scenarios with a pinned world seed (rival shares your group, nation not handed host), graded and stamped, difficulty badge from a calibration lab. About 7 campaigns.
- **Share cards**: post-match scoreboard and trophy cabinet as share images (square or stories), with a watermark.
- Club version: league tables history, club records, hall of fame, Ballon-style award, challenges per club, career saga stamps; scenario campaigns (save a club from relegation).

## 11. Editor and world packs

- **World pack** (`.fnmdb`, gzipped JSON, format versions 1-7 migrating stepwise): replaces or adds nations, players with traits, clubs and club strengths, sponsors with industries, Y accounts and handles, press reporters, stadiums, starting rankings, managers, localisation. A pack is written in the oldest format that holds it.
- **Editor** (`features/editor`): database editor on phone and tablet: players, names, trophies (with crop), history, merge, ranking and region filters, pack validation (profanity and status checks). A default world can be set in settings; a career can start from a pack.
- Club version: strongly recommended. A club game lives and dies by community databases of real leagues; the same format and merge tooling can carry clubs and leagues. Fictional names are a hard constraint in the FSM brief, so packs stay user-owned.

## 12. Saves, backup, cloud

- Local SQLite (drift), schema migrated stepwise (currently ~54+), **additive migrations only**, "never break saves" is an owner hard rule: new rules apply from the next cycle via per-cycle markers, every bump ships with an upgrade test over real old saves.
- Whole-database backup with `VACUUM INTO` (never a file copy because of WAL), restore swaps with a `.previous` copy; per-career bundle `.fnmcareer` (gzipped JSON, id remapping on import, merges rather than replaces); automatic cloud backup policy/store.
- Several careers (saves slots), each with a deterministic world seed (`rngSeed`) so worlds can be pinned.
- Offline-first, no accounts, no backend.

## 13. Monetisation (IAP)

- **One free cycle, then one purchase.** Cycle 0 is the full game (qualifying, continental cup, World Championship); `startNextCycle` is the single gate. One-time non-consumable `com.fnm.fnm.premium`, EUR 12.99. No ads, no subscription.
- Entitlement is store-verified at launch (StoreKit 2 / Play Billing), cached offline and only ever in the player's favour, with a 3-day grace window so an offline phone never locks a paying player. Default is locked even in debug. Restore purchase is user initiated. No RevenueCat to avoid a privacy policy and processor.
- Club version: the FSM brief has first season free, one purchase (TBD EUR 13.99); the same gate and cache design applies.

## 14. Localisation

- 28 languages in `lib/l10n/*.arb` (bg, cs, da, de, el, en, es, fi, fil, fr, hi, hr, hu, id, it, ja, ko, ms, nb, nl, pl, pt, ro, sv, th, tr, uk, vi). English is the fallback. Czech is the owner's reference language.
- **Copy round-trip**: `copy/strings.csv` (about 4,400 lines) has columns key / where / placeholders / en / cs; `tool/export_copy.dart` and `tool/import_copy.dart`. The importer is **all-or-nothing**: a lost or renamed `{placeholder}` or changed plural skeleton aborts with a list of bad rows. UTF-8 with BOM so Excel opens Czech. No em dashes in user-facing copy.
- Names of nations, competitions and news translated at display, stored values canonical English. Guards exist for raw English leaking and ALL CAPS strings. Czech is declined with grammatical cases, so sentences are assembled at render.
- Design for strings 40% longer than English and compound words.

## 15. Layouts: phone and tablet

- Phone: bottom navigation with a hub, Continue always reachable; single column, bento home.
- Tablet (landscape, window width >= 1000, `kTabletLayoutMinWidth`): adaptive shell with sidebar, top bar with Continue, list plus detail panes (`detail_pane_host`), per-screen `_PhoneLayout`/`_TabletLayout` split, about 53 screens; sheets become dialogs. FM-style "TV bar" match header, scoreboard and score bug, full-screen pitch.
- Match screen: preview, live with summary/stats/ratings/flow charts tabs, event flash, red-card banner, interval talks, full-time tabs, 2D pitch mode, TV header, sound director.

## 16. Design language

- **Fonts**: Hanken Grotesk (UI) and JetBrains Mono (numbers, tabular), both bundled variable fonts. Fixed font sizes, never scaled down to fit (premium refresh rule: no FittedBox on refreshed screens).
- **Palette** (`lib/core/theme`): dark, calm, silver-grey. Surface #121416, containers #1A1C1E to #333537, on-surface #E2E2E5, primary silver-blue #C3CDD9, outline #8E9195. Colour only where it means something (rating colours, kit colours, money, danger).
- **Premium refresh (planned 1.1.x)**: "simple but more premium", silver surfaces, one row per competition (trophy, one-line name, holder flag and name, status chip), nation select as a silver flag wall (3 columns, summary bar, one Manage button).
- Pitch visuals are drawn in code (stadiums, skylines, landmarks, weather), not images.
- Trophies and flags are assets; kits and crests in FSM must be generated (FSM brief).

## 17. What FSM should take, in one list

Take as-is: determinism and RNG discipline, the two-tier match simulation, the 2D pitch pipeline, event queue + Continue, press/Y/news voice, board loop, achievements and share cards, localisation round-trip, editor/world pack, backup and migrations discipline, IAP gate.
Rebuild: calendar (weekly), squad (contracts, transfers, wages), youth (academy), AI managers (per club), finances, league pyramids.

## Unverified

- Achievement and challenge counts are from pattern counts in the code (28 and 38) and may be off by a few.
- The exact staff roles list (third role name), current schema number and the 2030 first World Championship year were taken from notes, not re-checked in code.
- The exact finals format size (32 vs 48 teams) changed over versions; check `qualification_format.dart` before quoting.
- Landmark count "about 30 wired" comes from a note; 80 silhouettes exist in `landmark_paths.dart`.
