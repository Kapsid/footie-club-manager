# Footie Soccer Manager: Design Brief

> Brief for Claude Design. Everything marked **[TBD]** is open for the owner to decide.

## 1. The product in one paragraph

A deep football (soccer) club manager for phone and tablet. You take charge of one club anywhere in a huge world of leagues and run everything: squad, tactics, transfers, contracts, youth, staff, finances and the board. Matches are simulated, and you can watch them live. The game is offline, has no ads and no accounts. The first season is free, and one purchase (**[TBD] €13.99**) unlocks endless play.

## 2. Who it's for

- Football Manager fans who want real depth on a phone, not a casual clicker.
- People who play in short sessions on the go (2–10 minutes) and also in long evenings on a tablet.
- They love numbers, tables and history, and they read everything.

## 3. Design goals

1. **A look of its own.** It must not look like Football Manager, FIFA/EA FC or any other existing game. No copied layouts, colours or iconography.
2. **Dense but calm.** There's a lot of data on screen, so it needs a clear hierarchy, quiet surfaces and colour only where it means something (form, morale, money, danger).
3. **One thumb on phone, two panes on tablet.** The phone layout is built for one-handed use. The tablet layout shows a list and detail side by side, like a desktop manager game.
4. **The Continue button is the heartbeat.** The main action (move to the next event) is always in the same place and is never hidden.
5. **Fast to scan.** Tables, ratings and form strips should be readable at a glance. Numbers are tabular and aligned.
6. **Feels alive.** News, inbox messages, the press and social posts give the world a voice. Matchday should feel like an event.

## 4. Hard constraints

- **Fictional world.** No real club names, crests, league names, competition names, kits or sponsor brands. Countries and cities are real, and everything else is invented.
- **Generated crests and kits.** Crests are drawn by the app from shapes, colours and initials, so they need a crest system that looks good for over 5,000 clubs. Kits follow the same idea (primary and secondary colour plus a pattern).
- **Localisation.** The app ships in English and Czech first and grows to about 20 languages. Allow for strings 40% longer than English, and German and Czech compound words.
- **No em dashes in the app's text.**
- **Light and dark themes.** Both are first-class.
- **Accessibility.** Never rely on colour alone (form W/D/L uses letters too). Contrast must meet WCAG AA, and Dynamic Type scaling up to 130% must still fit.
- **Built in Flutter.** Use custom components rather than stock Material, but they must be buildable in Flutter.

## 5. Brand

- **Name:** Footie Soccer Manager **[TBD final name]**
- **Personality:** **[TBD]** For example: "serious, smart, a little warm. A broadsheet sports section, not a betting app."
- **Palette:** **[TBD]** Propose 2–3 directions.
- **Typography:** one family for UI and one for display, with tabular figures available **[TBD]**.
- **App icon:** **[TBD]**

## 6. Navigation

- **Phone:** a bottom bar with 5 destinations plus the Continue button, which sits in a fixed spot (a FAB or a bar slot, for Design to propose).
  - Home
  - Squad
  - Matches
  - World
  - Club
- **Tablet:** a left rail with the same destinations, list and detail panes, and Continue in the top bar.
- Club, player and competition pages are deep links reachable from anywhere (tap any name).

## 7. Screens to design (first pass)

### Start
1. **Splash and title.**
2. **New career.**
   - Pick a nation, then a league, then a club.
   - Club cards show expectation, budget, squad strength and stadium.
   - Manager profile: name, nationality, avatar, background.
3. **Save slots.**

### Home
4. **Dashboard.**
   - Next match card.
   - Inbox preview.
   - League position.
   - Squad alerts (injuries, contracts, unhappy players).
   - Board confidence.
   - Finances snapshot.
5. **Inbox.** Messages from staff, the board, players and agents, with actions inside messages (accept, reject, reply).
6. **Calendar.** Month view of fixtures, transfer windows and deadlines.

### Squad
7. **Squad list.**
   - Sortable, switchable columns (overview, attributes, condition, contract, stats).
   - Filters.
8. **Player profile.**
   - Header with face or avatar, position, age, nationality, value and contract.
   - Tabs: Overview, Attributes, Stats, History, Contract, Development.
   - Attribute layout of about 20 visible attributes plus personality traits **[TBD exact list]**.
   - Position map on a mini pitch.
9. **Tactics.**
   - Pitch with draggable players.
   - Formation picker.
   - Roles and instructions.
   - Set pieces.
   - Saved plans.
10. **Training.** Weekly schedule, focus per unit and per player, workload and injury risk.
11. **Youth academy.** Intake day reveal, prospect list and promotion.

### Matches
12. **Fixtures and results.**
13. **Match preview.** Opponent analysis, form, likely line-up, team talk.
14. **Live match.**
    - Pitch view or commentary feed.
    - Live stats and player ratings.
    - Substitutions and shouts.
    - Speed controls.
15. **Match report.** Ratings, stats and key moments.
16. **Press conference.** Questions with tone choices.

### World
17. **Competition page.** Table, fixtures, stats leaders and history. Must support split leagues, groups and play-offs.
18. **Cup bracket** and **continental competition** groups and knockouts.
19. **Nation and league browser.** About 100+ nations and 300+ leagues, so search is essential.
20. **Club page** (any club). Squad, staff, history, honours, stadium.
21. **News feed** and **social feed** (fictional platform).
22. **Hall of fame and records.**

### Club
23. **Transfers.**
    - Scouting search with filters.
    - Shortlist.
    - Bid and negotiation flow (offer, counter-offer, agent fee, clauses).
    - Transfer window clock.
24. **Contracts.** Renewal talks, wage structure.
25. **Finances.** Income and expenses, budget split, wage-to-revenue ratio, debt.
26. **Staff.** Hire and fire, staff attributes.
27. **Board.** Objectives, confidence meter, requests.
28. **Stadium and facilities.**

### Other
29. **Paywall.**
    - Shown at the end of the free season.
    - A respectful, single screen: what you get, one price, restore purchase.
    - No dark patterns.
30. **Settings.** Language, theme, match speed, backups.

## 8. Key components (design system)

- **Crest generator.** Shield shapes, patterns, initials, colours. Needs to look good at 16, 24, 48 and 128 px.
- **Kit swatch** (shirt icon showing club colours).
- **Rating badge** (player overall) and **attribute bar or number** style.
- **Form strip** (last 5: W/D/L with letters).
- **Morale and condition indicators.**
- **League table row** with zone colouring (title, continental places, promotion, play-off, relegation).
- **Money display** (compact: €12.4M, €850K).
- **Player row** (list item), in compact and comfortable densities.
- **Data table** with a sticky first column and horizontal scroll on phone.
- **Inbox message card** with action buttons.
- **Continue button** with states: idle, simulating (progress), blocked (needs action).
- **Live match event row** (goal, card, sub, injury, VAR).
- **Empty, loading and error states.**

## 9. What I want back from Claude Design

1. 2–3 visual directions (palette, type, mood) applied to the **Dashboard**, **Squad list** and **League table**.
2. After a direction is picked:
   - The design system: tokens (colour, type scale, spacing, radius, elevation), core components and the crest generator rules.
   - Screen designs for section 7, phone first, then tablet.
3. Light and dark mode for every screen.

## 10. Open questions [TBD]

- Final name and logo.
- Player faces: generated avatars, silhouettes or none?
- Live match: 2D pitch dots, commentary only, or both?
- How many attributes, and on what scale (1–20, 1–100, letters)?
- Currency: always euros, or local currencies?
- Price: €12.99 or €13.99?
