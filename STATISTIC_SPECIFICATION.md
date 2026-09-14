# EVA — Elo Value Added
### Statistic Specification, v1.0
Source dataset: `NHL_ELO_Odyssey.csv` (66,118 games, 1917–18 through 2022–23) and
`NHL_ELO_Odyssey_latest.csv` (2022–23 season extract with extended game-context fields)

---

## 1. Definition

**EVA** measures whether a team earned more or fewer standings points than its pregame
Elo rating implied it should, given the actual game outcome distribution the rating
system assigns to it.

> EVA answers: *"Is this team winning more than its rating says it deserves to?"*
> Elo answers: *"How good is this team?"*

They are deliberately different questions. A team can have a mediocre Elo and a highly
positive EVA (a team that keeps stealing points it "shouldn't"), or an elite Elo and a
negative EVA (a team living dangerously close to its rating, or worse, riding past
form more than current performance).

---

## 2. Inputs (all sourced directly from the dataset — nothing invented)

| Field used | Role |
|---|---|
| `home_team_winprob` / `away_team_winprob` | Pregame win probability implied by Elo |
| `overtime_prob` | Pregame probability the game is not decided in regulation |
| `home_team_expected_points` / `away_team_expected_points` | Dataset's precomputed expected standings points (see §3) |
| `home_team_score` / `away_team_score` | Final score, used to derive actual result |
| `ot` | Overtime/shootout flag and length, used to derive actual points |
| `season`, `date`, `playoff`, `neutral` | Context for aggregation and adjustment |
| `home_team_pregame_rating` / `away_team_pregame_rating` | Elo entering the game, used for the "underrated" cross-cut, not for EVA itself |

No column outside this list is required. `game_importance_rating` and
`game_overall_rating` are **not** used in EVA v1.0's core formula because they only
exist for the 2022–23 season (see Data Dictionary, §"Coverage gaps"); they are used
optionally in a single-season Clutch EVA variant (§6).

---

## 3. Reverse-engineered structure of `expected_points`

Fitting the provided `expected_points` columns against `winprob` and `overtime_prob`
recovers, to 5+ significant figures:

```
E[points_team] = 2 · P(team wins)  +  P(game reaches OT/SO) · P(opponent wins)
```

This is the standard "3-point game" convention (regulation win = 2, OT/SO win = 2,
OT/SO loss = 1, regulation loss = 0) — **applied uniformly to every season back to
1917-18**, even though that point system did not exist for most of NHL history
(ties were on the books through 2003-04; the OT-loss point did not exist before
1999-2000). This is a known simplification in Elo-rating projects that keep ratings
comparable across rule eras, but it means **raw actual-minus-expected is not directly
comparable across eras** — see §5.

---

## 4. Actual points (derived, following real historical NHL rules)

Actual points are computed from `home/away_team_score` and `ot` **using the point
system that actually applied in that season**, not the modern convention baked into
`expected_points`:

| Result | Seasons 1917–1999 | Seasons 2000–2004 | Seasons 2006–present |
|---|---|---|---|
| Regulation win | 2 | 2 | 2 |
| Regulation loss | 0 | 0 | 0 |
| Tie (score level, no OT winner) | 1 – 1 | 1 – 1 | n/a (ties abolished 2005) |
| OT/SO win | 2 | 2 | 2 |
| OT/SO loss | 0 (no loser point existed yet) | 1 | 1 |

(1918–1919 season data omitted from table for brevity; falls under the 1917–1999 rule column. No 2005 season exists — 2004-05 lockout.)

---

## 5. Core formula

**Game-level:**
```
EVA_game = Actual_Points − Expected_Points
```
computed once per team per game (so every game contributes one EVA value to each of
the two teams, and the two values are not required to sum to zero, since actual and
expected points use different point-system conventions — see §3–4).

**Season-level bias correction → EVA+ (era-adjusted):**

Because §3's uniform 3-point assumption creates a real, measurable era bias
(mean `EVA_game` ≈ **−0.20** in the pre-1983 no-OT era vs. ≈ **0.00** in the
2006+ era, on the actual data), raw EVA cannot be compared across eras without
correction. EVA+ removes this by centering every game on its own season's
league-wide mean:

```
season_bias(season) = mean(EVA_game) across all team-game rows in that season
EVA+_game = EVA_game − season_bias(season)
```

This uses only data already in the file (it's a within-dataset normalization, not an
external adjustment), and it is the metric that should be used for **any
cross-era comparison, leaderboard, or "who over/underperforms" claim.**

**Team-season EVA+:**
```
EVA+_season(team) = Σ EVA+_game  over all games that team played that season
```

---

## 6. Derived metrics

| Metric | Formula | Use |
|---|---|---|
| **EVA** | Σ EVA_game | Single-season, single-era analysis only |
| **EVA+** | Σ EVA+_game | Default metric for all cross-era leaderboards |
| **Playoff EVA+** (historical Clutch proxy) | Σ EVA+_game where `playoff == 1` | Clutch performance, all 106 seasons |
| **Clutch EVA (importance-weighted)** | Σ (EVA+_game × game_importance_rating) restricted to rows where `game_importance_rating` is non-null | Only computable for the 2022–23 season with current data; documented here so the formula is fixed in advance for when future seasons add this field |
| **Expected Wins** | Σ win_prob | Compare against actual wins for a "luck vs. skill" framing |
| **EVA+ per game** | EVA+_season / games | Normalizes across strike-shortened / lockout seasons for career comparisons |

---

## 7. Treatment of special cases

- **Overtime / shootout**: handled entirely inside the actual-points derivation (§4); no separate adjustment needed since `expected_points` already prices in `overtime_prob`.
- **Playoffs vs. regular season**: not weighted differently within EVA/EVA+ itself (a playoff win is still 2 points); playoff performance is isolated via the separate Playoff EVA+ metric, not by re-weighting the core stat, so EVA+ retains one consistent definition of "a point."
- **Neutral-site games** (`neutral == 1`, 291 games, mostly outdoor/international games): included in EVA+ as normal. Flagged in the data so they can be excluded by any downstream analysis that wants to isolate true home-ice effects — not excluded by default because the Elo model itself already accounts for the neutral flag when it set `winprob`.
- **Franchise continuity**: the dataset backfills relocated/renamed franchises under their current identity (e.g., 1979–1995 Quebec Nordiques seasons are labeled "Colorado Avalanche"/`COL` throughout). Career EVA+ therefore reflects franchise lineage, not city/brand identity. This is documented so it isn't mistaken for a data error.
- **Minimum sample size**: team-season leaderboards use no minimum; cross-season "career" or "most underrated" leaderboards use a **1,000-game minimum** (~12 seasons) to avoid small-sample noise from short-lived or recently-expanded franchises.

## 8. Interpretation

- **EVA+ > 0**: team is taking more standings points out of its games than its Elo-implied probabilities predicted — either clutch, well-coached in close games, or riding variance.
- **EVA+ < 0**: team is leaving points on the table relative to its rating — either unlucky, weak in close games, or a rating that hasn't caught down to reality yet.
- EVA+ is a *performance-vs-expectation* stat, not a *quality* stat. A bad team with a highly positive EVA+ is not necessarily a good team — it is a team outperforming its own (possibly low) bar.

## 9. Known limitations

1. `expected_points`'s uniform 3-point-game assumption (§3) is a property of the source Elo model, not of EVA — EVA+ corrects for its *average* effect per season but cannot correct for any within-season heterogeneity the same assumption might introduce (e.g., if OT rates vary a lot within a season).
2. Elo pregame ratings themselves already incorporate some mean-reversion, so a team with a strongly positive multi-season EVA+ will, mechanically, also tend to see its Elo rise over time — the two are not fully independent over long windows.
3. Full importance-weighted Clutch EVA (§6) is presently a single-season metric due to data coverage; treat any historical "clutch" claim as the Playoff EVA+ proxy, not the importance-weighted version, unless stated otherwise.
4. This spec defines v1.0. Any change to the actual-points table (§4), the season-bias correction (§5), or the minimum-sample rule (§7) should increment the version number rather than silently changing the numbers behind an existing chart or claim.
