# Ligue-Wide View — Offensive / Defensive

**Category:** Attack and defence metrics at league level (all clubs).

**Data source:** Power BI data model — `FactClubMatch`, `DimClub`, `DimMatch`.

---

## A. Attack–Defence map (efficiency frontier)

**Visual type:** Scatter chart

**Purpose:** Compare clubs by attacking output and defensive solidity to identify profiles:
- **Dominant:** high goals for, low goals against (top-right if Y = “fewer is better”, or top-left if Y inverted).
- **Fragile:** high goals for but high goals against (attack-minded, leaky).
- **Pragmatic:** low goals for, low goals against (tight, low-scoring).
- **Weak:** low goals for, high goals against.

**Axes and size:**

| Role   | Field / measure                         | Notes |
|--------|----------------------------------------|-------|
| **X-axis** | Goals for per match                    | Average goals scored per match (GoalsFor / Match). |
| **Y-axis** | Goals against per match (inverted)     | Average goals conceded per match. Invert axis or use a “defence score” so **high = better** (e.g. Y = −GoalsAgainst/Match, or invert the Y-axis in the visual so top = fewer goals against). |
| **Size**   | Points                                 | Bubble size = total points (or points per match). |
| **Legend / category** | Club (DimClub[ClubName])     | One point per club; use **club_colors.json** for consistent colours. |

**DAX measures (league-wide, one value per club):**

```dax
Matches Played = COUNTROWS(FactClubMatch)

Goals For = SUM(FactClubMatch[GoalsFor])
Goals Against = SUM(FactClubMatch[GoalsAgainst])

Goals For per Match = DIVIDE([Goals For], [Matches Played], BLANK())
Goals Against per Match = DIVIDE([Goals Against], [Matches Played], BLANK())

Total Points = SUM(FactClubMatch[Points])
```

**Optional (for “high = better” on Y without inverting axis):**

```dax
Defence Score = - [Goals Against per Match]
```
Use `[Defence Score]` as Y so that higher = fewer goals conceded. Otherwise use `[Goals Against per Match]` as Y and **invert the Y-axis** in the scatter format.

**Power BI setup:**

1. Add a **Scatter chart** visual.
2. **X-axis:** `[Goals For per Match]`.
3. **Y-axis:** `[Goals Against per Match]` (then Format → Y-axis → Reverse scale) **or** `[Defence Score]`.
4. **Size:** `[Total Points]`.
5. **Legend / Details:** `DimClub[ClubName]` (one point per club).
6. Apply the **Visual Charter** club colours (Format by field value from the Club Colors table loaded from `club_colors.json`).
7. Add tooltips: ClubName, Goals For per Match, Goals Against per Match, Total Points (and optionally Goals For, Goals Against).

**Interpretation:** Top-left = few goals conceded (strong defence); right = many goals scored (strong attack). Dominant clubs sit top-right (attack + defence); fragile clubs right and down; pragmatic clubs left and up; weak clubs bottom-left.
