# Ligue-Wide View — Performance

**Category:** League-level performance metrics (goal difference, win rate, points per match, ranking).

**Data source:** Power BI data model — `FactClubMatch`, `DimClub`, `DimMatch` (and `DimDate` where applicable for matchday/rank).

---

## Visuals on this page

The Performance page includes the following visuals:

1. **Goal difference by Club** — Grouped bar chart showing each club’s total or average goal difference (goals for minus goals against).
2. **Win Rate by Club** — Stacked bar chart showing the percentage of matches won per club (wins / matches played).
3. **Points/Match by Club** — Stacked bar chart showing average points per match per club (total points / matches played).
4. **Rank at Matchday** — Line chart showing the league rank at each matchday for the **top 5 clubs** (e.g. clubs ranked 1–5 at the end of the season, or the 5 best by total points).

---

## 1. Goal difference by Club

**Visual type:** Grouped bar chart

**Purpose:** Compare clubs by goal difference (Goals For − Goals Against) over the season.

| Role        | Field / measure                    | Notes |
|------------|------------------------------------|-------|
| **Category** | Club (`DimClub[ClubName]`)        | One bar per club. |
| **Value**    | Goal difference                    | Use `FactClubMatch[Goal difference]` (calculated column) summed, or a measure: `SUM(FactClubMatch[GoalsFor]) - SUM(FactClubMatch[GoalsAgainst])`. |

**DAX (if using a measure):**

```dax
Goal Difference Total = SUM(FactClubMatch[GoalsFor]) - SUM(FactClubMatch[GoalsAgainst])
```

---

## 2. Win Rate by Club

**Visual type:** Stacked bar chart

**Purpose:** Show the proportion of matches won per club (wins / matches played), as a percentage.

| Role        | Field / measure                    | Notes |
|------------|------------------------------------|-------|
| **Category** | Club (`DimClub[ClubName]`)        | One bar per club. |
| **Value**    | Win rate (%)                       | Wins / Matches played × 100. |

**DAX:**

```dax
Matches Played = COUNTROWS(FactClubMatch)
Wins = CALCULATE(COUNTROWS(FactClubMatch), FactClubMatch[Result] = "W")
Win Rate Pct = DIVIDE([Wins], [Matches Played], BLANK()) * 100
```

Use **Win Rate Pct** (or format as percentage) as the value; category = Club.

---

## 3. Points/Match by Club

**Visual type:** Stacked bar chart

**Purpose:** Show average points per match per club (total points ÷ matches played).

| Role        | Field / measure                    | Notes |
|------------|------------------------------------|-------|
| **Category** | Club (`DimClub[ClubName]`)        | One bar per club. |
| **Value**    | Points per match                   | Total points / matches played. |

**DAX:**

```dax
Total Points = SUM(FactClubMatch[Points])
Matches Played = COUNTROWS(FactClubMatch)
Points per Match = DIVIDE([Total Points], [Matches Played], BLANK())
```

Use **Points per Match** as the value; category = Club.

---

## 4. Rank at Matchday (top 5 clubs)

**Visual type:** Line chart (rank on Y-axis, matchday on X-axis)

**Purpose:** Display how league rank evolves over matchdays for the five best-ranked clubs (e.g. the top 5 by final standings or by total points).

| Role        | Field / measure                    | Notes |
|------------|------------------------------------|-------|
| **Rows / Legend** | Club (`DimClub[ClubName]`)      | Filter or restrict to top 5 clubs (e.g. by final rank or total points). |
| **X-axis**  | Matchday (`DimMatch[Matchday]`)    | Or use `DimDate` if rank is computed per date. |
| **Y-axis / Values** | Rank at matchday              | Running rank per matchday (e.g. rank by cumulative points at that matchday). |

**Implementation notes:**

- **Top 5 filter:** Use a filter on Club so that only the 5 clubs with the highest total points (or final rank) are shown — e.g. a Top N filter on `DimClub` by a measure like `Total Points` or a “Final Rank” measure, with N = 5.
- **Rank at matchday:** Requires a measure that computes rank at each matchday (e.g. rank by cumulative points up to that matchday). This can be done with a measure that uses `ALLSELECTED(DimMatch[Matchday])` and cumulative logic, or a calculated table/column that stores “points after matchday M” and then rank; exact DAX depends on the model’s date/matchday structure.

---

## Summary

| Visual                  | Chart type           | Main metric / axis              | Category      |
|-------------------------|----------------------|----------------------------------|---------------|
| Goal difference by Club | Grouped bar chart    | Goal difference (total or avg)   | Club          |
| Win Rate by Club        | Stacked bar chart    | Win rate (%)                     | Club          |
| Points/Match by Club    | Stacked bar chart    | Points per match                 | Club          |
| Rank at Matchday        | Line chart           | Rank at each matchday            | Matchday × Top 5 clubs |

All visuals use `FactClubMatch` and `DimClub`; Rank at Matchday additionally uses `DimMatch[Matchday]` (and possibly `DimDate`) and a top-5 filter on clubs.
