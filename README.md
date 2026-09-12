# Road to the World Series — MLB Data, 2021–2025

Complete regular-season and postseason game logs, plus season-level player statistics, for **every team that reached the MLB World Series from 2021 through 2025** — ten team-seasons, one Champion and one Runner-Up per year.

This is the public data source behind the Tableau Public visualization **[All In The Wins | #VOTD](https://public.tableau.com/app/profile/john.johansson/viz/AllIntheWins/RoadtotheWorldSeries)** by John Johansson.

> The workbook is published as-is, exactly matching what the visualization consumes. No Tableau calculated fields, parameters or formatting are included here — just the underlying data.

---

## Contents

| File | Description |
|---|---|
| `Data_Baseball_Road_To_World_Series_2021-2025.xlsx` | The dataset — 6 worksheets, 2,909 data rows (~551 KB) |
| `Baseball Teams Logos.zip` | Team logo images for all 30 MLB clubs (~3.5 MB) |
| `DOCUMENTATION_Baseball_Road_To_World_Series_2021-2025.txt` | Full documentation: complete data dictionary for all 184 columns, join model, and verified data-quality notes |

Start here for a quick orientation; read the documentation file before building anything on the data.

---

## What's in the dataset

| Sheet | Rows | Cols | Grain | Role |
|---|---:|---:|---|---|
| `WS Teams` | 10 | 10 | team-season | **Hub / dimension** |
| `Season Games` | 1,619 | 37 | team × regular-season game | Fact |
| `PS Games` | 161 | 25 | team × postseason game | Fact |
| `Batting Stats` | 321 | 36 | player × team-season | Fact |
| `Pitching Stats` | 342 | 39 | player × team-season | Fact |
| `Fielding Stats` | 426 | 32 | player × team-season | Fact |
| `Team Name Opponite` | 30 | 5 | MLB club | Lookup |

### The ten team-seasons

| Year | Champion | Runner-Up |
|---|---|---|
| 2021 | Atlanta Braves | Houston Astros |
| 2022 | Houston Astros | Philadelphia Phillies |
| 2023 | Texas Rangers | Arizona Diamondbacks |
| 2024 | Los Angeles Dodgers | New York Yankees |
| 2025 | Los Angeles Dodgers | Toronto Blue Jays |

**This is a team-centric dataset, not a league-wide one.** Only the 10 World Series participants have rows. Opponents appear as codes and lookup attributes only — their own game logs and player stats are not included, so this file cannot produce league-wide averages or standings.

---

## How to join the sheets

`WS Teams` is the hub. Four fact tables attach to it on the single key **`Unique`**. The two game tables each attach an opponent lookup on **`Opp`**.

```
                                  ┌──────────────────┐
                              ┌──▶│  Batting Stats   │
                              │   └──────────────────┘
                              │   ┌──────────────────┐
                              │──▶│  Pitching Stats  │
   ┌──────────────┐           │   └──────────────────┘
   │              │           │   ┌──────────────┐      ┌──────────────────────┐
   │   WS Teams   │──[Unique]─┼──▶│   PS Games   │─────▶│  Team Name Opponite  │
   │    (hub)     │           │   └──────────────┘[Opp] │     (instance 1)     │
   │              │           │                         └──────────────────────┘
   └──────────────┘           │   ┌──────────────┐      ┌──────────────────────┐
                              │──▶│ Season Games │─────▶│  Team Name Opponite  │
                              │   └──────────────┘[Opp] │     (instance 2)     │
                              │                         └──────────────────────┘
                              │   ┌──────────────────┐
                              └──▶│  Fielding Stats  │  (not used by the viz)
                                  └──────────────────┘
```

| # | Left | Key | Right | Key | Cardinality |
|---|---|---|---|---|---|
| 1 | `WS Teams` | `Unique` | `Batting Stats` | `Unique` | one-to-many |
| 2 | `WS Teams` | `Unique` | `Pitching Stats` | `Unique` | one-to-many |
| 3 | `WS Teams` | `Unique` | `PS Games` | `Unique` | one-to-many |
| 4 | `WS Teams` | `Unique` | `Season Games` | `Unique` | one-to-many |
| 5 | `PS Games` | `Opp` | `Team Name Opponite` | `TeamID Opponite` | many-to-one |
| 6 | `Season Games` | `Opp` | `Team Name Opponite` | `TeamID Opponite` | many-to-one |
| 7 | `WS Teams` | `Unique` | `Fielding Stats` | `Unique` | one-to-many *(recommended — not in the published workbook)* |

**`Team Name Opponite` must be joined twice, as two independent instances** — once to `Season Games`, once to `PS Games`. Tableau names the second one `Team Name Opponite1`; in SQL, alias the table twice.

### The `Unique` key

`Unique` is the primary key of the whole model, built as:

```
Year + "_" + League Champion + "_" + Team Name Short      e.g.  2025_Champion_Dodgers
```

It has to be composite: the Astros appear in both 2021 and 2022, the Dodgers in both 2024 and 2025. **Always join on `Unique`** — never on year + team name, and never on the three-letter team code.

Verified: `WS Teams.Unique` is unique across all 10 rows, and every `Unique` value in every fact sheet resolves to it. **Zero orphan keys.**

### ⚠️ Use relationships, not physical joins

The fact tables sit at different grains that share only the team-season key. Physically joining them on `Unique` produces a Cartesian fan-out — for the 2024 Dodgers alone, `Season Games` (162 rows) × `Batting Stats` (23 rows) = **3,726 rows**, with team wins counted 23× and every player's home runs counted 162×.

Use Tableau **relationships** (the logical-layer "noodle"), which query each table at its own level of detail. In SQL or pandas, **aggregate each fact table to the `Unique` grain first, then join the aggregates.**

```sql
SELECT  t.[Unique], t.[Year], t.[Team Name],
        g.[Gm#], g.[Game Date], g.[W/L], g.[R], g.[RA],
        o.[Team Name Opponite]
FROM         [WS Teams]           AS t
INNER JOIN   [Season Games]       AS g  ON g.[Unique] = t.[Unique]
LEFT  JOIN   [Team Name Opponite] AS o  ON o.[TeamID Opponite] = g.[Opp];
-- LEFT JOIN on the lookup is deliberate: 20 rows don't match (see below).
```

```python
import pandas as pd

xl  = pd.ExcelFile("Data_Baseball_Road_To_World_Series_2021-2025.xlsx")
hub = xl.parse("WS Teams")
sg  = xl.parse("Season Games")
opp = xl.parse("Team Name Opponite")

season = (sg.merge(hub, on="Unique", how="left", suffixes=("", "_team"))
            .merge(opp, left_on="Opp", right_on="TeamID Opponite", how="left"))
```

---

## Before you use it — the short list

These are verified against the file. Full detail is in section 6 of the documentation.

| | Issue |
|---|---|
| 🔴 | **`PS Games` `W/L` is unreliable on 16 of 161 rows** — it contradicts `R` vs `RA`. Five rows show an impossible tied score; eleven have the result inverted (notably the 2023 World Series, where the Diamondbacks' five rows carry the Rangers' outcomes). Derive the result from `R > RA` instead. `Season Games` is clean on all 1,619 rows. |
| 🔴 | **Filter out `Team Totals` rows** before aggregating `Batting`, `Pitching` or `Fielding Stats`. Ten rows (eight in Fielding) have `Player = "Team Totals"` and a blank `Rk` — they are pre-aggregated team sums, which is why counting columns reach values like 6,306 plate appearances. |
| 🟠 | **20 opponent codes don't resolve** in the lookup: `ATH` (10 rows, lookup uses legacy `OAK`), `CWS` (4 rows, lookup uses `CHW`), `BRA` (6 rows, everything else uses `ATL`). Use a LEFT join or these rows vanish. |
| 🟠 | **`Fielding Stats` covers 2021–2024 only** — neither 2025 team has rows. |
| 🟠 | **`League Champion` in `Season Games` is wrong on 161 rows** — the 2025 Dodgers read "Runner-Up" on all but game 1. The hub is correct; source team attributes from `WS Teams`, not from the denormalized copies. |
| 🟡 | **Dates are inconsistent.** `Game Date` is *text* (`M/D/YYYY`) in `Season Games` but a *real date* in `PS Games`. `Date Calc` is a plotting helper with the year forced to 2025 for 2021–2024 and stored as year-less text (`"Apr-1"`) for 2025 — never use it as a date. |
| 🟡 | **`W/L` in `Season Games` has four values**, not two: `W`, `L`, `W-wo`, `L-wo` (walk-offs). Test with *starts with* `W`, or you'll miss 70 wins. |
| 🟡 | **`Z2` uses a blank to mean "home game"** (809 rows). That blank is data, not a missing value. |
| 🟡 | **`Batting Stats` has two columns named `Pos`** (simple position, and a detailed B-R position string). Tableau renames the second to `Pos 1`; pandas to `Pos.1`. |
| 🟡 | **`IP` (pitching) and `Inn` (fielding) use baseball thirds** — `140.1` means 140⅓ innings. Convert to outs before doing arithmetic. |
| 🟡 | **`Player-additional`** (the Baseball-Reference player ID) is missing for both 2025 teams and holds `-9999` on 2021–2024 Team Totals rows. Use `Unique` + `Player` as the player key instead. |

Section 7 of the documentation lists an 11-step cleanup if you want to publish a derived, analysis-ready version alongside the raw file.

---

## Team logos

`Baseball Teams Logos.zip` contains **30 transparent PNG logos, one for each MLB club**, sized to roughly 905 px on the longest edge (~3.5 MB unzipped). These are the shape images used by the Tableau visualization.

Filenames are not standardized — most are lowercase with underscores (`toronto_blue_jays.png`), a few use the nickname only (`Dodgers.png`), and one keeps its original source filename. There is no ID column in the dataset that maps to them directly, so expect to build a small filename-to-team lookup if you want to join them programmatically.

Logos are team trademarks, included for reference and non-commercial visualization use only. See attribution below.

---

## Provenance

- **Statistics:** [Baseball-Reference.com](https://www.baseball-reference.com/) (Sports Reference LLC) — team schedule/results pages and standard team batting, pitching and fielding tables. Some Baseball-Reference conventions survive in the column names (`cLI`, `OPS+`, `Rdrs/yr`, `Player-additional`), along with one scrape artifact (`Z1`, which contains only the word `boxscore`).
- **Logos:** [SportsLogos.net](https://www.sportslogos.net/)

The published Tableau workbook connects to **four separate Excel files**. For this public release those four were consolidated into one workbook and the sheet names were tidied. Nothing else was changed — columns, values and grain are identical to what the visualization consumes.

| Tableau object | Sheet in this workbook |
|---|---|
| `Team Groups` | `WS Teams` |
| `Games_Champions` | `Season Games` |
| `PS Games` | `PS Games` |
| `Batting` | `Batting Stats` |
| `Pitching` | `Pitching Stats` |
| `Team Name Opponite` / `Team Name Opponite1` | `Team Name Opponite` *(one sheet, used twice)* |
| — | `Fielding Stats` *(included here; not used by the viz)* |

> "Opponite" is the author's spelling of "opponent". It's preserved throughout so the data stays in sync with the published workbook.

---

## License and attribution

The statistics are the work of **Baseball-Reference.com (Sports Reference LLC)** and the logos are sourced from **SportsLogos.net**; both remain subject to their owners' terms of use. Team names and logos are trademarks of their respective clubs and of Major League Baseball.

This repository is a derived compilation assembled for a data-visualization project and is offered for **educational, analytical and portfolio use**. Please credit Baseball-Reference as the source of the statistics, SportsLogos.net for the logos, and this dataset as the compilation. Check Sports Reference's and SportsLogos.net's current terms before any commercial or large-scale redistribution.

**Suggested citation**

> Johansson, John. *Data_Baseball_Road_To_World_Series_2021–2025* (2026). Compiled from Baseball-Reference.com; logos from SportsLogos.net. Companion dataset to the Tableau Public visualization *All In The Wins*. https://public.tableau.com/app/profile/john.johansson/viz/AllIntheWins/RoadtotheWorldSeries
