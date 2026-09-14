# Data Dictionary — NHL ELO Odyssey

Two files were supplied:

- **`NHL_ELO_Odyssey.csv`** — 66,118 games, 1917-12-19 through 2023-06-13 (season labels
  1918–2023), 40 franchise identities, 24 columns.
- **`NHL_ELO_Odyssey_latest.csv`** — 1,400 rows, all from the 2022-23 season only. Same
  24 columns; this is a season extract of the same underlying pipeline, not an
  additional/different schema.

**Note:** the brief for this project assumed a 48-column schema. The actual files have
**24 columns**. Everything below reflects the real schema — nothing here was inferred
or padded to reach a target column count.

| # | Column | Type | Null? | Meaning | Used in EVA? |
|---|---|---|---|---|---|
| 1 | `season` | int | never | Season label (e.g. `2023` = the 2022-23 season) | ✓ (grouping, era rules) |
| 2 | `date` | date | never | Game date | ✓ (ordering) |
| 3 | `playoff` | 0/1 | never | Playoff game flag | ✓ (Playoff EVA+) |
| 4 | `neutral` | 0/1 | never | Neutral-site game flag (291 games total) | flagged, not excluded |
| 5 | `status` | string | never | Always `"post"` in this file (completed games only) | not used |
| 6 | `ot` | string | 55,255 empty | Overtime/SO indicator: blank = regulation, else `OT`,`SO`,`2OT`…`6OT` | ✓ (actual-points derivation) |
| 7 | `home_team` | string | never | Home franchise, current name (see continuity note below) | ✓ |
| 8 | `away_team` | string | never | Away franchise, current name | ✓ |
| 9 | `home_team_abbr` | string | never | 3-letter code | ✓ (join key) |
| 10 | `away_team_abbr` | string | never | 3-letter code | ✓ (join key) |
| 11 | `home_team_pregame_rating` | float | never | Elo rating entering the game | ✓ (Elo cross-cut, not EVA formula itself) |
| 12 | `away_team_pregame_rating` | float | never | Elo rating entering the game | ✓ |
| 13 | `home_team_winprob` | float | never | Elo-implied win probability | ✓ (core EVA input) |
| 14 | `away_team_winprob` | float | never | `1 − home_team_winprob` always | ✓ |
| 15 | `overtime_prob` | float | never | Pregame probability game is not decided in regulation | ✓ (core EVA input) |
| 16 | `home_team_expected_points` | float | never | Precomputed E[points]; reverse-engineered formula in Spec §3 | ✓ (core EVA input) |
| 17 | `away_team_expected_points` | float | never | Same, away side | ✓ |
| 18 | `home_team_score` | int | never | Final score | ✓ (actual-points derivation) |
| 19 | `away_team_score` | int | never | Final score | ✓ |
| 20 | `home_team_postgame_rating` | float | never | Elo after the game | not used (could seed a future "Elo swing vs EVA" study) |
| 21 | `away_team_postgame_rating` | float | never | Elo after the game | not used |
| 22 | `game_quality_rating` | int | never (0–100 scale) | A 538-style "how good was this game to watch" index | not used in v1.0 |
| 23 | `game_importance_rating` | float | **64,718 / 66,118 empty** — only populated for season `2023` | Stakes/leverage of the game | optional, single-season Clutch EVA only |
| 24 | `game_overall_rating` | float | **64,718 / 66,118 empty** — only populated for season `2023` | Composite of quality × importance | not used in v1.0 |

## Coverage gaps (verified directly against the data, not assumed)

- `game_importance_rating` and `game_overall_rating` are populated for **exactly one
  season: 2022-23**. Every other season (1918–2022) has these blank. This is why
  Clutch EVA defaults to the `playoff` flag as a proxy rather than importance-weighting
  — the weighted version literally cannot be computed before 2022-23 with this data.
- `game_quality_rating` (col 22) is fully populated for all 106 seasons, unlike its two
  neighbors — it just isn't currently used by EVA v1.0.
- Regular ties (`home_team_score == away_team_score`) occur in 5,754 games, all in
  seasons 1918–2004. The shootout era (2006+) has zero.

## Franchise continuity convention

`home_team`/`away_team` use a franchise's **current** name/abbreviation retroactively.
Confirmed directly in the data: all Quebec Nordiques seasons (1979–1995) appear as
"Colorado Avalanche" / `COL`. Any team/season table using this file inherits that
convention — a "Colorado Avalanche 1990 season" in the outputs is the Quebec Nordiques.
