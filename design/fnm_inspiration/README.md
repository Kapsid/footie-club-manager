# FNM screens: inspiration for Footie Soccer Manager

Screens from **Football Nations Manager 1.1.1** (the sister game: international football, same owner). They are here as **reference for density, flow and tone**, not as a look to copy. Footie should get its own visual identity (see `../../DESIGN_BRIEF.md`, section 3).

All names, clubs and managers in the screens are fictional. Captured on an iPhone 17 Pro and an iPad 13" simulator, dark theme, English.

## What is worth taking from FNM

- **Hub + one primary action.** Home is a hub with one big next-step button (the "Continue" heartbeat in the brief).
- **Calm dark surfaces, colour only where it means something** (form, money, danger, our team).
- **Mono figures for data** (JetBrains Mono) next to a grotesk for UI (Hanken Grotesk).
- **2D match view** with stadium, crowd, weather and a live commentary line, plus stats under it.
- **Tabs instead of long scrolls** on dense pages (player, finances, squad).
- **Tablet = list + detail side by side.**

## Phone (`phone/`)

| File | Screen |
|---|---|
| `01_hub`, `n_n01_hub` | Home hub: next match, board, funds, group table, Play button |
| `02_select_nation` | New career: pick a team (club picker equivalent) |
| `03_tactics`, `f_03_squad` | Line-up on a pitch with ratings and condition |
| `04_tactics_instructions` | Team instructions with sliders and style chips |
| `05_match_preview`, `n_p05_match_preview` | Match preview: head to head, venue, attendance, weather |
| `06/07_match_2d_*`, `n_p06_*`, `n_2x_match_2d_*`, `f_24/25_*` | Live 2D match: goals, penalties, rain, snow, storm, fog, haze |
| `f_05_match_full_time`, `n_n06_full_time_summary` | Full time: score, player of the match, stats, share |
| `08_youth` | Youth pyramid U-13 to U-21 |
| `09_world_championship` | Tournament page with bracket |
| `10_my_career_trophies` | Career summary and trophy cabinet |
| `11_records`, `12_achievements` | Record book, achievements and challenges |
| `13_ranking` | World ranking with history chart |
| `14_world_managers`, `f_13_world_manager` | Other managers and a manager profile |
| `15_competitions`, `n_n03_competitions` | Competitions overview |
| `f_14_share_sheet` | Share card (square / story image) |
| `n_n04_player` | Player page: attributes, recent form, tabs |
| `n_n08/n09_finances_*` | Finances in tabs: overview, departments, facilities |
| `n_r01_campaigns`, `n_r02/r04_campaign_brief_*` | Campaigns (hand-built challenges) |
| `n_r03_national_stadiums` | Stadium map |
| `n_11_editor_clubs`, `n_15_editor_home`, `n_n11_editor_merge_review` | In-app database editor |

## iPad (`ipad/`)

The same key screens in the tablet layout (rail navigation, two panes, 2D match with side columns).

## Docs

- `../../docs/FNM_GAME_OVERVIEW.md`: every FNM system and what it becomes in a club manager.
- `../../docs/FNM_MATCH_ENGINE.md`: how a match is decided and drawn in 2D, and what can be reused.
