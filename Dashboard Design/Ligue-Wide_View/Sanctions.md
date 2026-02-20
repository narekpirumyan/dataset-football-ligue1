


## A. Sanctions by club (stacked by reason)

**Visual type:** Stacked bar chart

**Purpose:** Show total number of sanctions per club (all clubs in the league) with the possibility to see the breakdown by **reason** (each bar is stacked by sanction reason). Compare discipline across clubs and identify which reasons are most frequent per club or overall.

**Axes and legend:**

| Role | Field | Notes |
|------|--------|--------|
| **Axis (category)** | `DimClub[ClubName]` | One bar per club (or use **Y-axis** for horizontal bars). |
| **Value** | Count of sanctions | Number of rows in FactSanction per club (and per reason for the stack). |
| **Legend** | `FactSanction[Reason]` | Stack segments by reason; each segment colour = one reason. |

**Data source:** `FactSanction`, `DimClub`. FactSanction must have a **Reason** (or similar) column from `disciplinary_sanctions.csv` (reason).

**DAX measure:**

```dax
Sanction Count = COUNTROWS(FactSanction)
```

Use **Sanction Count** in the **Value** well. Power BI will aggregate by Club (axis) and by Reason (legend), so you get one bar per club and segments per reason.

**Power BI setup:**

1. Add a **Stacked bar chart** (Barres empilées) visual.
2. **Axis** (Y for horizontal bars): `DimClub[ClubName]`.
3. **Value** (Length): `[Sanction Count]` (or drag `FactSanction` and set aggregation to Count of rows if you prefer).
4. **Legend**: `FactSanction[Reason]` (or the column that stores the sanction reason in your model). This creates the stack: each reason is one segment.
5. **Sort:** Optionally sort bars by total sanctions (descending) so the most sanctioned clubs appear first.
6. Apply **Visual Charter** colours for clubs if the chart uses club on axis; for the legend (reasons), use a distinct palette so each reason is readable. Optionally use a single colour for “total” view and distinct colours per reason for the stack.
7. **Tooltips:** Add ClubName, Reason, Sanction Count (and optionally SanctionType, SuspensionMatches, FineEuros) so users can see details on hover.

**Variants:**

- **100% stacked bar:** Format → set “Show as percentage of total” (or use a measure that divides count by total per club) so each bar shows the **share** of each reason per club instead of absolute counts.
- **Drill-down:** Enable drill-down on the axis (Club) so users can drill to a lower level if you add one later (e.g. player); or add a **slicer** on Reason to filter to one or several reasons.

---

## B. Cumulative sanction count by matchday (line chart)

**Visual type:** Line chart

**Purpose:** Show the **cumulative** number of sanctions over the season: X = matchday (1–38), Y = total sanctions counted up to that matchday. Highlights how discipline incidents accumulate across the season (league-wide or per club if filtered).

**Axes:**

| Role | Field | Notes |
|------|--------|--------|
| **X-axis** | Matchday (1–38) | One point per matchday. Use a **Matchday** dimension (e.g. table with values 1 to 38) or `DimMatch[Matchday]` with one row per matchday (see below). |
| **Y-axis** | Cumulative sanction count | Number of sanctions with `SanctionDateKey` ≤ end of that matchday. |

**Data source:** `FactSanction` (SanctionDateKey), `DimMatch` (Matchday, MatchDate). To get "end of matchday N", use the date of the match(es) on that matchday (e.g. `MAX(DimMatch[MatchDate])` for that Matchday).

**Option A — Matchday dimension + DAX measure**

1. **Matchday axis:** Create a small table **DimMatchday** with one column **Matchday** (values 1 to 38). Add it to the model; no relationship needed if you use the measure below.
2. **Cumulative measure:** For each matchday on the axis, compute the "cutoff date" (end of that matchday) from `DimMatch`, then count sanctions up to that date:

```dax
Cumulative Sanction Count =
VAR MatchdayVal = MAX(DimMatchday[Matchday])
VAR EndDate =
    CALCULATE(
        MAX(DimMatch[MatchDate]),
        ALL(DimMatch),
        DimMatch[Matchday] = MatchdayVal
    )
RETURN
    CALCULATE(
        COUNTROWS(FactSanction),
        FactSanction[SanctionDateKey] <= EndDate,
        ALL(FactSanction[SanctionDateKey])
    )
```

Use **DimMatchday[Matchday]** on the X-axis and **`[Cumulative Sanction Count]`** on the Y-axis. Ensure the visual shows one point per matchday (no duplicate matchdays).

**Option B — Assign sanction to matchday in the model**

In Power Query or with a calculated column, add **Matchday** to `FactSanction` by matching `SanctionDateKey` to the matchday whose match date is the first ≥ sanction date (or "end of matchday" logic). Then use a simple running total:

```dax
Cumulative Sanction Count = 
VAR MatchdayVal = MAX(FactSanction[Matchday])
RETURN
    CALCULATE(
        COUNTROWS(FactSanction),
        FactSanction[Matchday] <= MatchdayVal,
        ALLSELECTED(FactSanction[Matchday])
    )
```

X-axis: the Matchday column (from a table that has one row per matchday, e.g. DimMatchday or summarized FactSanction by Matchday). Y-axis: this measure.

**Power BI setup:**

1. Add a **Line chart** (Graphique en lignes).
2. **X-axis:** Matchday (1–38) from DimMatchday or equivalent; ensure **one point per matchday** and sort by Matchday ascending.
3. **Y-axis:** `[Cumulative Sanction Count]`.
4. **Legend (optional):** Add `DimClub[ClubName]` if you want one line per club (cumulative per club); otherwise one line for the whole league.
5. **Format:** Y-axis = Whole number; no decimals. X-axis min/max 1–38 if needed.
6. **Tooltips:** Matchday, Cumulative Sanction Count (and ClubName if by club).
