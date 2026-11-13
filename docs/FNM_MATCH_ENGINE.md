# FNM Match Engine and Pitch View

Reference for Football Nations Manager (FNM, Flutter, repo `fnm26`, app code in `lib/`). Written as inspiration and context for Footie Soccer Manager. Concepts, data flow and numbers only. All figures below were read from the code as of build 1.1.1 (49).

## 1. Big picture

FNM has two match simulators that share one idea: a match is a pure, deterministic function of (teams, seed).

| | MatchEngine (live) | RatingMatchSimulator (background) |
|---|---|---|
| Used for | every match the manager plays or watches, and his skipped matches use a parity path of it | every AI vs AI fixture in the world |
| Granularity | minute by minute, 90 + stoppage (+30 extra time) | one scoreline from two strength numbers |
| Players | real XI, bench, roles, energy, cards | none; a box score is derived afterwards |
| Cost | heavy (thousands of lines, many RNG draws per minute) | a few arithmetic operations |
| Files | `domain/services/match/match_engine.dart` (~3900 lines) | `match_simulator.dart`, `background_match.dart` |

Key product decision: **the world is never simulated with the full engine.** Only the manager's fixtures are. Everything else is dressed after the fact so the world feels real (cards, injuries, ratings, scorers) without the cost.

## 2. Determinism and RNG streams

- RNG is `SeededRng` (mulberry32, 32-bit arithmetic, identical on every platform). A match is keyed by (save seed, fixture id), so a replay or a cloud restore reproduces it exactly.
- The live screen **re-simulates the whole match from minute 1** every time the manager changes anything (sub, tactic, talk). That is why subs, tactical changes and team talks are timed inputs to `play()` rather than mutations. Same inputs give the same timeline, so nothing already watched ever rewrites itself.
- The engine uses **seven RNG streams** so adding a feature never shifts old results:
  1. main stream: open-play chances, goals, cards, injuries;
  2. assists (`state ^ const`);
  3. set pieces;
  4. long throws (only drawn by a side told to throw long);
  5. display detail: shot zone, how a miss ends (never decides an outcome);
  6. "play" stream: penalties and direct free kicks;
  7. own goals (re-credits a goal that was going in anyway).
- Rule of thumb the owner holds: new texture goes on a new forked stream, never on the scoring stream. Guard tests pin scorelines.
- Calibration is done with lab tests (`FNM_LAB=1` match grid, a career lab of ~1200 careers) and guard tests in `test/unit/match/` (baseline, balance, penalty rate, extra time, weather, skip parity). Retune by lab, never by feel.

## 3. Inputs

### 3.1 Player model
`lib/domain/entities/player.dart`, `player_attributes.dart`.
- Three attributes only, each 1..99: **physical** (pace, power), **technical** (all on-ball skill and awareness), **stamina**. Collapsed from ten in schema 32.
- **Overall** is position weighted, derived not stored: by category (GK / DEF / MID / FWD) a different mix of the three. Example: keeper = 0.60 technical, 0.30 physical, 0.10 stamina; defender = 0.42 physical, 0.40 technical, 0.18 stamina.
- **Potential is not stored on the player.** Hidden development potential is derived from the player id (`PlayerLifecycle.developmentPotential`). "Superstar" = potential at or above 1.70 (a multiplier). Aging ticks on 1 December; development was slowed ~35% in one batch.
- Also on the entity: id, nationId, name, age, position, club name/country/stars (cosmetic, derived), `value` (euros from overall with an age curve, display only: `((overall-44)/12)^3` million times age factor 1.2 / 1.1 / 0.9 / 0.6 / 0.35 for <=23 / <=27 / <=30 / <=33 / older).
- **Traits** (derived per save, or fixed by a world pack): hothead (cards), iron man (slow tiring), big-game player (+bonus in knockouts), leader (lifts team mates, not himself), wasteful (finishing penalty). Supplied to the engine as a map so the engine never needs the save seed.
- **Roles** (`player_role.dart`): none, targetMan, poacher, playmaker, ballWinner, invertedWinger, ballPlayingDefender. A role is a set of multipliers on scorer weight, assist weight, aerial weight and set-piece threat (poacher 1.7x scorer weight, 0.5x assist; playmaker 1.8x assist, 0.8x scorer). Roles never change raw team strength, only who gets the chances.
- **Condition** (form, fatigue, morale, captain, staff, scout dossier) is a hidden per-player rating shift in points applied inside the engine only. The displayed overall never moves, so one player never shows three different ratings on three screens. `StrengthFactors` is the single reading all screens use to explain what is moving the side (familiarity, fatigue, club form, morale, captain, staff, dossier).

### 3.2 Tactics model
`lib/domain/entities/tactics.dart`, `formation.dart`.
- `TacticalInstructions`: six dials 0..100, 50 neutral: mentality, pressing, tempo, width, defensive line, directness.
- **Playstyle** = a named preset of the six dials: balanced, possession, gegenpress, counter, direct, lowBlock, wingPlay. Moving a dial by hand makes it `custom`. (Example: gegenpress = mentality 75, pressing 90, tempo 80, width 55, line 85, directness 60.)
- `Tactic` = formation + lineup (11 slot to player id) + instructions + playstyle + full-back arrows + slot switches (changing a place within the shape) + throw-in style and taker.
- 19 formations (4-4-2, 4-3-3, 3-5-2, 4-2-3-1, 5-3-2, 4-1-4-1, 3-4-3, 4-2-2-2, 4-2-4, 4-1-2-1-2, 4-4-1-1, 4-3-1-2, 4-5-1, 5-4-1, 3-4-2-1, 3-4-1-2, 5-2-1-2, 4-3-2-1, 3-5-1-1). Each gives a position per slot and a `ShapeProfile` (attackers vs defenders count against a neutral).
- **Position fit** (`PositionFit.factor`): a player out of position keeps counting for the line he plays in, at a reduced rating (the tactics screen shows e.g. 84 to 62).
- **Team chemistry**: a familiarity multiplier (about 1.0 for a rotating side, up to about 1.06 for a long-settled shape) and a hidden predictability (how many matches in a row the same plan was played). The AI reads the manager's most recent plan and counters it more sharply the more predictable he is. A change of plan resets it.

### 3.3 Match context
`MatchTeam` (xi, instructions, formation, roles, penalty and dead-ball takers, condition map, traits, scout bonus, full-back arrows, switches, throw-in style) plus: neutral venue and host id, atmosphere multiplier, big-match flag (knockout/final), injury factor per team (medical budget), chemistry per team, climate (`MatchConditions`), timed substitutions, tactical changes, team talks, and which sides are AI-managed.

## 4. How a minute is decided (MatchEngine.play)

Each minute of 90, then stoppage (a few minutes, longer the more interrupted), then optionally 91-120:

1. **Housekeeping**: apply timed subs, tactical changes, talks for that minute. AI-managed sides re-derive their instructions from score and clock (from about the hour mark a trailing side pushes mentality, tempo and line up, a leading side sits).
2. **Energy drain** per player: `0.62 (keeper 0.12) * workload * (1.4 - stamina/100) * ageFactor * ironMan * heat/altitude`. Workload = `1 + (tempo-50)/250 + (pressing-50)/200`, times `1 + 0.30 per man sent off`. Age factor `1 + (age-28)*0.04` clamped 0.9..1.35.
3. **Effective rating** of each player: `overall + condition shift + scout bonus (+ big-game, + leader)`, times position fit, times an energy curve. The energy curve is kinked: 100% energy = 1.00, 72% = 0.95, then falls steeply to about 0.51 at empty. Fresh legs matter, so subbing a tired man is an obvious call.
4. **Line strengths** (mean effective rating by slot category, not natural position):
   - attack = 0.55 FWD + 0.30 MID + 0.15 DEF,
   - defence = 0.55 DEF + 0.25 GK + 0.20 MID.
5. **Adjustments** on both: mentality (attack +0.10 per point off 50, defence -0.07), tempo (+0.03 attack), defensive line (+0.04 attack, -0.04 defence), width (-0.015 defence), shape (+/-1.25 per extra attacker or defender), team chemistry multiplier, a short-handed penalty (flat 26 rating points per red, escalating by 8 per extra red, 85% of it off defence). Width and directness have no flat value: they are decided by the **match-up** below.
6. **Home advantage**: +3 attack and +2 defence, times an atmosphere factor (full, loud ground = more). Neutral finals give nothing, except to the host if on the pitch. Team-talk swing and **momentum** (a goal lifts the scorer, rocks the conceder, fades each minute; added to ratings, never to an RNG) are added too.
7. **Tactical match-up** (`matchupBonus`, weight 1.85): how each plan and each side's attributes fare against the other's: pressing vs passers, a high line vs pace in behind, width vs narrow shapes, directness vs a deep line. This is what makes a plan good or bad depending on the opponent.
8. **Chance creation**: `rate = 0.078 * edge^1.2 * (0.85 + tempo/333) * (0.7 + 0.6 * possessionShare)`, clamped 0.010..0.280 per minute, then multiplied by damping (fatigue "legs", weather tempo, long-ball weather, full-back arrows, throw-in style, a blow-out damper once a side leads by 3+, and **openness**: committed sides open the game for both). `edge` is the attack/defence ratio measured above a replacement baseline of 26, which is convex in the gap. The cap is what keeps 6-goal routs rare.
9. **Finishing**: `p = 0.145 * ratio^0.25`, clamped 0.03..0.20. Deliberately flat: favourites win by creating more, not by absurd conversion. The shooter is picked by weight (position, attributes, role, wasteful trait) and his energy sharpens or dulls the finish (0.74..1.0).
10. **xG** of each chance is accumulated; shot zone and how a miss ends (saved, wide, blocked, woodwork) are drawn on the display stream.
11. **Penalties and free kicks**: a fraction of finished chances (0.175) becomes a foul in the box, about 0.40 penalties a match (tuned so the owner sees one every 2-3 matches). Penalty base conversion 0.79 adjusted by taker and keeper; direct free kick base 0.07. A penalty can bring a card (15%).
12. **Set pieces** (own stream): about 0.035 per minute of corner/free-kick deliveries (60% corners), headers weighted by physical, role and position; taker = the manager's designated dead-ball taker or the best technical outfielder (keeper never by default). Long throws (0.009 per minute for a side told to throw long) and safe-throw styles shift attack and exposure slightly. Routine throw-ins (0.45/min) and goal kicks (0.18/min) are only counted for the picture. Idle fouls 0.085/min, offsides 0.021/min. Own goals: 2.5% of goals from shots, 7% from crosses.
13. **Discipline**: yellow 0.015 per team per minute (about 1.3 a team a game), straight red 0.00015, a foul by a booked man costs the second card 45% of the time. The culprit is drawn weighted to hard-working defenders, the tired and the already booked. Team fatigue multiplies fouls and knocks by up to about 1.5.
14. **Injuries**: 0.0016 per team per minute, scaled by the nation's medical investment. The player leaves, the side plays a man down unless a sub is made.
15. **Weather** (`weather_play.dart`): multipliers on finishing, long-range shots, long balls, crossing, tempo, passing, fouls. Heavy rain/thunder, snow/frost, gale, fog all nudge things; all exactly 1.0 in clear or light rain. Calibration bar: no weather moves goals a match by more than about 3%. Heat and altitude act through energy drain (`MatchConditions`), derived from the ground's city altitude and the country's monthly climate.
16. **Extra time and shootout**: a level knockout plays 91-120 with chance rates times 1.1 (calibrated so live extra time settles about half of level ties, matching the background path). Then a shootout; the manager can name the order (interactive shootout). Regulation score and events are untouched by whether extra time is played.

## 5. Outputs

`MatchResult`: events (goal with assist/penalty/set-piece/long-throw/own-goal flags, yellow, red with second-booking flag, injury, substitution), regulation score, extra-time totals, stoppage minutes, xG series per minute, shots per minute, momentum per minute (for the momentum bar), possession, end energy per player, per-player marks, man of the match, the plan each side held longest (what the next opponent scouts), the AI instruction log, and **moments**.

- **Moments** (`match_moments.dart`): every real thing the engine decided, in order, for the picture: shots (zone, outcome goal/saved/wide/blocked/woodwork, origin, header), set pieces (corner, free kick, long throw), fouls (with advantage), offsides, subs. Plus counts of routine throw-ins and goal kicks per minute (flow, never staged).
- **Ratings** (`performance_mark.dart`, `live_rating.dart`): one formula for the engine and the background world: base 6.5, goals, assists, result, clean sheet, cards, penalties missed/saved, clamped 3.0..10.0. `MatchTally` counts events so the live line-up at full time equals the stored mark by construction. **Player of the match** = best mark, ties broken by line order.

## 6. Background simulator (AI vs AI)

- **Strength anchor** (`strengthOf`): from world ranking, `92 - (rank-1)*0.21 - min(rank,20 elite band)-1)*0.4`, clamped 40..92. It is only a quarter of a background side's strength; the rest is its real fielded XI.
- **Expected goals** multiplicative in the gap: `xG = base * exp(0.06 * gap)` with the gap run through tanh and capped around 25 points, both sides divided by `cosh(x)^0.3` so mismatches do not inflate totals. Base xG between equals: home 1.235, away 1.0925, neutral the mean (home advantage is +5 in this scale).
- Both scorelines are sampled segment by segment in step, so a side far ahead eases off (matches the live engine's blow-out damping).
- Knockout ties in finals: gap speaks 1.5x louder (`finalsGapStretch`) plus a pedigree edge, so top seeds medal more often than minnows. Level ties get extra time and a shootout weighted by `exp(0.1 * strength)`.
- **Reference points** (neutral, thousands of sims, earlier tuning): even sides about 1.35-1.32 (39/25/36 W/D/L), 88 vs 76 about 1.9-0.95 (59% win), 88 vs 42 about 5.5-0.2.
- `BackgroundMatch.detail` **dresses** the scoreline: possession from strength, shots from xG, assists, cards (per-player rates, friendlies far lower), a rare red (about one per 6-7 matches, some are second bookings), knocks, subs at a reduced share, and a mark for all 22 players. It runs on a forked RNG (`fixtureId ^ salt`) so no existing result changes. Cards and knocks are real engine events so suspension and injury rules apply to the whole world the same way as to the manager's squad.
- **Skip parity**: when the manager skips a match, `planEffect` converts his plan, shape and the opponent's into a rating delta and an openness multiplier on the simulator's scale, so a skipped match tells the same story as a played one (`skip_parity_test`).

## 7. AI behaviour

- `ai_tactics.dart`: the opponent's posture is chosen deterministically from the fixture seed and the strength gap vs the manager: clear favourites press high and take the initiative, heavy underdogs sit deep and counter. Small per-field jitter. Seed string is `aitac:<opponent>:<fixture>:<career seed>` and is shared by live and skip paths. Strength = mean of the best 11 by overall.
- `ai_counter.dart` / `opponent_scouting.dart`: the opponent prepares for the manager's most recently played plan; sharper the more predictable he is. Scout staff add a rating bonus and a dossier.
- `ai_substitutions.dart`: 2-5 changes in a match (most often 4), 3-8 in a friendly (window opens at half time), weakest outfielder for best same-line bench player, never the keeper; an extra-time sub exists. Deterministic from the RNG so re-sims reproduce them.

## 8. Venue, crowd, attendance

- `home_ground.dart`: a nation has several grounds; big nights go to the national stadium, others rotate over the biggest grounds; mountain nations (Quito, Bogota, La Paz) always play at altitude. Finals are at the host's stadiums. Everything derived from fixture id.
- `attendance.dart`: attendance follows the occasion (a final fills the ground, a friendly does not), deterministic per fixture. It feeds the atmosphere factor, so a packed house is worth more home advantage.
- Stadium and city data in `match/stadium/`: stadium spec, ground character and scale, crowd culture and crowd show, away fan share, daylight (day/dusk/night from kick-off and latitude), city style, horizon, season, landmarks (about 80 silhouettes authored, 30 or so wired to cities), weather kind (clear, rain, snow, haze, heavy rain, thunderstorm, fog, dust).

## 9. The 2D pitch view

Added in 1.1.0. Pure Dart domain half plus Flutter painters. **The picture never decides anything.**

### 9.1 Pipeline
```
MatchResult (events + moments + plans + instruction log)
  -> PitchPacer      match minutes -> screen seconds
  -> StoryLayer      believable build-up between real moments
  -> MomentStaging   each real moment played out from its data
  -> PitchScript     the whole match as a sampled function of screen time
  -> painters        stadium (baked), actors, ball, weather, referee, confetti
```
All of `domain/services/match/pitch/*` is pure Dart (no Flutter), so it can be tested headless and moved to a shared package.

- **PitchScript** (`pitch_script.dart`, ~1100 lines): sampling `script.sample(t)` returns every actor's position and facing, ball phases (carried at feet, travelling in a line with optional lob peak, at rest), camera (centre x and zoom), cues (whistle, card, flag, shot, goal), cuts (kick-off after a goal, set piece ready, new phase) and resets (after intervals).
- **PitchPacer**: three speeds (slow, mid, fast). Staged moments always get their full fixed length; quiet minutes share the remainder of the budget evenly with a minimum each. A quiet minute is about 1-2 screen seconds, so the clock runs 30-70x faster than the picture. Slots: minutes 1-90, 90+1.. stoppage, 91-120. Half time takes no screen time (the screen pauses separately).
- **StoryLayer** (`story_layer.dart`, ~3200 lines plus `ball`, `lead_in`, `restart`): the filler football. Passes, carries, switches, pressing, recovery with each side holding its formation, slot switches and instructions. Speeds are real football speeds in screen seconds (jog 3-6 m/s, sprint cap 8.8 m/s, passes 11-26 m/s); compression changes the *choice* of football (longer passes, more ground per possession), never running speed. Pressing sets the block, directness the passes, tempo the ball speed, counter-minded sides break, roles move men. It leads into each moment (right side on the ball, roughly in place) and picks up from where the moment leaves players. The manager's live changes reshape the lines at their minute; the AI's instruction drift is followed from the minute after each change.
- **MomentStaging** (`moment_staging/`): one stager per moment type: shot (by origin), dead ball (corner, free kick, long throw), penalty, foul and offside, sub and injury, rebound (an open-play box goal sometimes drawn as parry then follow-up), and the penalty shootout kick by kick. Outcome comes from the record (goal ends in the net, save in the keeper's hands, wide passes a post, blocked is stopped by a defender in the path, every pass in a goal's move finds an onside man). Stagers react to the state they are handed, so a window staged from mid-way (a resume) is coherent. Their own randomness (penalty lane, spot within a zone) comes from the window seed only.
- Supporting: `formation_slots`, `pitch_geometry`, `switched_layout`, `display_spacing` (keeps tokens from overlapping), `referee_card`.

### 9.2 Determinism
- Display randomness is seeded from the match seed and the moment itself, so a re-watched or re-simulated minute looks and reads the same (commentary wording included: several lines per (moment type, outcome), chosen by seed).
- The script is built **off the UI thread** (`Isolate.run`) by `PitchModeController`; the screen keeps the rules (intervals, goal flash, injuries, full time) and asks the controller what the picture says at a clock position. Re-simulation after a manager change rebuilds the script; events are matched across re-sims by identity.
- Known parked issue: shootout ticker not held on app background.

### 9.3 Flutter side (`lib/features/match/pitch/`)
- **PitchClock**: a Ticker drives screen time, `pacer` maps it back to a match minute, speed multiplier on top.
- **PitchProjection**: pitch metres (x length, y across) to a tilted screen plane (far touchline narrower; tilt parameter, 0 = flat).
- **StadiumPainter** (~1650 lines): sky, land, stands, crowd, boards, mown grass, markings, goals, floodlights. **Baked once** in three key looks (day, dusk, night) and cross-faded, so nothing is re-baked while a match plays and the sun going down is a fade between ready images. Dressed per occasion (finals band in the competition colour, gold for the final).
- **Crowd**: blocks of seats with fill level and whose fans sit where (home/away), crowd culture decides shows.
- **Backdrop**: horizon (the country's nature) + city skyline in a regional style in two depth rows + a landmark silhouette, all seeded per ground so a skyline never changes. 80 silhouettes drawn as code paths.
- **WeatherPainter**: rain streaks, snow, fog banks, dust, lightning, drawn once into a tile. Haze, fog grey, dust ochre, frost are baked into the ground looks.
- **Actors / tokens**: shirt in the kit with number and a facing chevron, shadow, slim name label when named; referee painter; confetti; sound director.
- **Commentary**: one short line per real moment, nothing for routine ones.
- **Shootout stage**: its own clock and painter, one kick every few seconds.
- Phone and tablet use the same script; only layout differs.

## 10. Reuse for a club manager

### Lift as-is (shared package candidates)
- `SeededRng` and the stream-forking discipline.
- Pure engine core: `MatchEngine.play`, `MatchTeam`, `MatchResult`, moments, `performance_mark`, `live_rating`/`MatchTally`, `position_fit`, `shape_profile`, `PlayerRole`, tactics entities, formation slots, weather play, penalty/dead-ball takers, discipline rates. Almost everything is already pure Dart with injected inputs.
- The whole `domain/services/match/pitch/*` pipeline and the painters in `features/match/pitch/*` (they take a MatchResult and kits, not a nation).
- The two-tier idea: live engine for the manager's matches, cheap simulator plus dressing for the rest, with skip parity and calibration labs.

### Nation specific (needs rename or abstraction)
- `nationId` is the team key almost everywhere: `MatchTeam.nationId`, `Substitution.teamNationId`, event `teamNationId`, maps keyed by nation (injury factor, chemistry, AI-managed set). Replace with a generic `teamId` (club id). Mechanical but wide.
- Venue logic assumes tournaments: neutral finals, host nation, `HomeGround` rotating over a nation's grounds. Club version: one home stadium per club (with capacity and character), neutral only for cup finals.
- Strength anchor from world ranking (`strengthOf(Nation)`), finals gap stretch, world pedigree: replace with club rating (Elo or squad value) and league-table pedigree.
- AI tactics seed uses opponent and fixture ids and "best 11 mean": fine, rename.
- Calibration numbers (goals per match, cards, penalties) were tuned to international football; club football has more goals variance by tier and 38-46 matches a season. Re-run the labs.
- Cosmetic clubs and "club stars" on the player are derived from nation pools; a club game owns the real contract.

### Needs change for weekly seasons and thousands of clubs
- **Cost**: the live engine is for the manager's one match a week. 5,000 clubs means roughly 2,500 AI matches per matchday across leagues; the RatingMatchSimulator is cheap enough, but `BackgroundMatch.detail` (22 marks, cards, knocks per match) should be **lazy**: derive detail only for fixtures a player could see (own league, followed clubs, cup ties) or only aggregate (top scorers need goals only; compute names on demand from a seeded stream keyed by fixture).
- **Lazy player generation**: FNM's seed pool is 4,807 named players for 209 nations and the rest are derived by `poolAt(nation, cycle)` on a pure function of ids (intake per year, aging, retirement). For clubs use the same trick: a club squad is a function of (club id, season, seed) until the player touches it (signs, transfers, plays for the manager); only then persist a delta. Keep player ids stable and encode (club, intake year, slot).
- **Persistence of results**: nation game stores every fixture. For thousands of clubs store results plus compact scorer lists, prune detail after the season, keep standings and records.
- **Calendar**: nation game is event-driven by cycles (4-year loop, windows). A club game needs a weekly tick; the Continue button advances to the next event of interest for the manager and fast-forwards the rest in batches, with a "world sim step" per matchday run off the UI thread.
- **Condition and fatigue** are derived from recent ratings and minutes. With weekly games this becomes the main driver (rotation, injuries, form). It already exists as `Condition` (form, fatigue, morale) but was tuned for ~10 matches per cycle.
- **Match-day plans**: slot switches, full-back arrows, set-piece takers, throw-in styles, team talks and half-time re-sim are manager-side only; keep them as the "depth" layer, with AI sides using presets by default.
- **Training and development**: replace the 1 December aging tick with weekly or monthly training nudges; keep the 3-stat model or expand to a few more attributes if the owner wants more depth (the 10 to 3 collapse was made for readability).
- **Derived-provider staleness** is the main bug class in FNM: reads that do not notice a sim step. Design the provider graph around one "world revision" counter.
