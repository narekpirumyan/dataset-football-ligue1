


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

**Visual type:** Line chart (Graphique en lignes)

**Purpose:** Show the **cumulative** number of sanctions over the season: X = matchday (1–38), Y = total sanctions counted up to that matchday. Highlights how discipline incidents accumulate across the season (league-wide or per club if filtered).

**Axes:**

| Role | Field | Notes |
|------|--------|--------|
| **X-axis** | Matchday (1–38) | One point per matchday. Use **`DimMatch[Matchday]`** on the axis; Power BI shows one category per distinct value (38 points). No need for a separate Matchday table. |
| **Y-axis** | Cumulative sanction count | Number of sanctions with `SanctionDateKey` ≤ end of that matchday. |

**Data source:** `FactSanction` (SanctionDateKey → DimDate), `DimMatch` (Matchday, DateKey → DimDate). There is no direct relationship between FactSanction and DimMatch; use the date of the match(es) on that matchday as cutoff. Per 0_markdown.md, DimMatch has **DateKey** (FK to DimDate). To get "end of matchday N", use `MAX(DimMatch[DateKey])` for that Matchday.

**Option A — Use DimMatch for the axis (recommended)**

Put **`DimMatch[Matchday]`** on the X-axis. Power BI groups by distinct Matchday, so you get 38 points. For each point, the measure gets the end date of that matchday from `DimMatch` (DateKey is the match date per schema), then counts sanctions up to that date:

```dax
Cumulative Sanction Count =
VAR MatchdayVal = MAX(DimMatch[Matchday])
VAR EndDate =
    CALCULATE(
        MAX(DimMatch[DateKey]),
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

**X-axis:** `DimMatch[Matchday]`. **Y-axis:** `[Cumulative Sanction Count]`. No extra table needed.

*Optional:* If you prefer a dedicated dimension with exactly one row per matchday (1–38), create a small table **DimMatchday** with column **Matchday**, use it on the X-axis, and in the measure use `MAX(DimMatchday[Matchday])` instead of `MAX(DimMatch[Matchday])`; the rest of the measure stays the same.

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
2. **X-axis:** `DimMatch[Matchday]` (Option A) or Matchday from DimMatchday / FactSanction (Option B); ensure **one point per matchday** and sort by Matchday ascending.
3. **Y-axis:** `[Cumulative Sanction Count]`.
4. **Legend (optional):** Add `DimClub[ClubName]` if you want one line per club (cumulative per club); otherwise one line for the whole league.
5. **Format:** Y-axis = Whole number; no decimals. X-axis min/max 1–38 if needed.
6. **Tooltips:** Matchday, Cumulative Sanction Count (and ClubName if by club).

**Alternating between total and breakdown (by reason / by club):**

Allow users to switch between a single line (total cumulative sanctions) and multiple lines (cumulative by reason or by club) without changing the DAX measure.

- **Total view:** Same line chart with **no field in Legend** → one line for league-wide cumulative count.
- **Breakdown view:** Same line chart with **Legend** = `FactSanction[Reason]` (one line per reason) or **Legend** = `DimClub[ClubName]` (one line per club).

**Implementation — Buttons + bookmarks (recommended):**

1. Create the line chart in **total** mode (Legend well empty). Create a **bookmark** (e.g. "Cumul total"); in Bookmark pane, uncheck "Data" if you want the bookmark to only capture visual state.
2. Add **Reason** (or **ClubName**) to the chart **Legend**. Create a second **bookmark** (e.g. "Cumul par raison" or "Cumul par club").
3. Add **buttons** (Insert → Boutons) or a simple shape/segment: e.g. "Total" and "Par raison" (or "Par club"). Set each button's action to **Bookmark** → select the corresponding bookmark.
4. Optional: add a **slicer** on Reason (or Club) so that in breakdown view users can filter to specific reasons or clubs; the cumulative measure will respect filters.

No change to the cumulative measure is required; only the Legend configuration and bookmarks/buttons define the two (or more) display modes.

---

## C. Sanction count by reason (donut chart)

**Visual type:** Donut chart (Graphique en anneau)

**Purpose:** Show the **distribution** of sanctions by reason across the league: one segment per reason, value = number of sanctions for that reason. Highlights which types of incidents (e.g. red cards, accumulation of yellows, simulation, unsportsmanlike conduct) are most frequent overall.

**Roles:**

| Role | Field | Notes |
|------|--------|--------|
| **Legend** (or **Axis**) | `FactSanction[Reason]` | One segment per distinct reason. |
| **Values** | Sanction count | Number of rows in FactSanction per reason. |


**Data source:** `FactSanction`. Use the existing **Reason** column (from `disciplinary_sanctions.csv`).

**DAX measure:**

Reuse the same measure as in section A:

```dax
Sanction Count = COUNTROWS(FactSanction)
```

**Power BI setup:**

1. Add a **Donut chart** (Graphique en anneau).
2. **Legend** (or **Axis**): `FactSanction[Reason]`.
3. **Values:** `[Sanction Count]`.
4. **Sort:** Optionally sort by Sanction Count descending so the most frequent reasons appear first.
5. Use a **distinct palette** for reasons so each segment is readable (avoid club colours here; this is reason-based).
6. **Tooltips:** Reason, Sanction Count, and optionally percentage of total.
7. **Center label (optional):** Show total sanction count or "Sanctions par raison" in the center.

**Note:** If there are many distinct reasons, consider showing only the top N (e.g. filter or "Top 10" in the visual) or use a slicer on Reason so users can focus on a subset.

---

## D. Total sanctions by city (map with bubble size)

**Visual type:** Map (Carte) — bubbles sized by value

**Purpose:** Display **total sanctions per city** on a map: one point per city (from `DimClub[city]`), with **bubble size** proportional to the total number of sanctions in that city (all clubs in the city combined). Gives a league-wide geographic view of discipline by location.

**Data source (0_markdown.md):** `FactSanction` (via `ClubKey`), `DimClub` (`city`). No Latitude/Longitude in the schema — use **city** for geocoding (Power BI/Bing recognizes French city names).

**One point per city — recommended: summarized table**

To avoid duplicate points when several clubs share the same city, use a table with **one row per city** and total sanctions.

**Option A — Calculated table (DAX)**

Create a calculated table, e.g. `CitySanctions`:

```dax
CitySanctions =
SUMMARIZE(
    DimClub,
    DimClub[city],
    "TotalSanctions", CALCULATE( COUNTROWS(FactSanction) )
)
```

Result: columns `city`, `TotalSanctions`. Use this table as the source for the map (no relationship needed if you only need this visual; otherwise relate via `city` to a dimension if you use one).

**Option B — Power Query**

- Start from `DimClub` (or a query that joins DimClub and FactSanction).
- Group by `city`.
- Add column: total sanctions = count of related `FactSanction` rows (or sum of a count column). Output columns: `city`, `TotalSanctions`.
- Load as a table, e.g. `CitySanctions`.

**If you keep using DimClub directly:** Put `DimClub[city]` in Location and a measure in Size (see below). You may get one point per club; if two clubs share a city, you can get two overlapping points. Prefer the summarized table for one bubble per city.

**DAX measure (when using DimClub or for tooltips)**

```dax
Sanction Count = COUNTROWS(FactSanction)
```

When the visual groups by city, this gives total sanctions in that city. For the summarized table, use the column `TotalSanctions` in the **Size** well.

**Power BI setup**

1. Add a **Map** visual (Carte).
2. **Location:** `City` (from `CitySanctions`) or `DimClub[city]` — one value per city so Power BI geocodes the city name.
3. **Size:** `TotalSanctions` (from `CitySanctions`) or `[Sanction Count]` if using DimClub. Bubble size = total sanctions in that city.
4. **Legend (optional):** Add `DimClub[ClubName]` only if you need one point per club; for one point per city, leave legend empty or use a tooltip.
5. **Tooltips:** City, Total Sanctions (and optionally list of clubs in that city).
6. **Format:** Adjust size range (min/max) so bubbles are readable; optional labels.

**Note:** Geocoding uses the **city** name only (e.g. Paris, Lyon). For more precise stadium locations you would need Latitude/Longitude columns (not in the current schema).

---

## E. Sanctions at home vs away (bar chart)

**Visual type:** Bar chart (Graphique à barres) — two bars: one for sanctions at home (domicile), one for sanctions away (extérieur).

**Purpose:** Compare the **number of sanctions at home** vs **away** (league-wide or per club if filtered). One bar = total sanctions in “domicile” context, one bar = total in “extérieur” context.

**Data source:** `FactSanction`, `DimClub`. The current schema (0_markdown.md) does **not** include a home/away attribute on `FactSanction`. You need a way to associate each sanction with a match context (home or away for the sanctioned club).

**Enriching the model with Venue (Domicile / Extérieur)**

Add a column to `FactSanction` (in Power Query or as a calculated column) that indicates whether the sanction occurred in a **home** or **away** context for the club:

- **Option A — Power Query:** For each sanction, match `SanctionDateKey` and `ClubKey` to the match(es) where that club played on that date (via `DimMatch` + `FactClubMatch`: same date and club). If the club was home in that match, set `Venue = "Domicile"`; if away, `Venue = "Extérieur"`. If a club played twice on the same day, use matchday or a rule (e.g. first match). Load as `FactSanction[Venue]` (Text) or `FactSanction[IsHome]` (Boolean).
- **Option B — Calculated column (DAX):** If you can uniquely identify the match (e.g. you add `MatchKey` to FactSanction), then `Venue = IF(LOOKUPVALUE(FactClubMatch[IsHome], FactClubMatch[MatchKey], FactSanction[MatchKey], FactClubMatch[ClubKey], FactSanction[ClubKey]), "Domicile", "Extérieur")`. Otherwise prefer Power Query.

If you cannot link sanctions to a specific match, you can approximate: e.g. for each sanction date and club, take the **matchday** where that club played and get IsHome from `FactClubMatch` for that club and matchday (one row per club per match, so one IsHome value per club per match). Matching on (ClubKey, Date) to (FactClubMatch via DimMatch date) gives the match and thus IsHome.

**DAX measures**

Once `FactSanction` has a **Venue** column (e.g. `"Domicile"` / `"Extérieur"`):

```dax
Sanction Count = COUNTROWS(FactSanction)
```

Use this in the **Value** well; Power BI will slice by the field you put on the axis.

**Power BI setup**

1. Add a **Bar chart** (Graphique à barres) or **Clustered bar chart**.
2. **Axis (Category):** `FactSanction[Venue]` (or a table with two rows: "Domicile", "Extérieur") so you get exactly two bars.
3. **Value:** `[Sanction Count]` (or drag FactSanction with Count). You get one bar for Domicile and one for Extérieur.
4. **Sort:** Optionally sort by value descending.
5. **Format:** Whole numbers; optional data labels on the bars.
6. **Tooltips:** Venue, Sanction Count; optionally ClubName if filtered by club.
7. Use a **slicer** on Club to show home/away sanctions for a selected club, or leave unfiltered for league totals.

**Alternative axis:** If you prefer a small **dimension table** with two rows (e.g. `DimVenue`: VenueKey, VenueName = "Domicile" / "Extérieur"`) and a measure that filters by it, you can put `DimVenue[VenueName]` on the axis and use a measure that counts sanctions where `FactSanction[Venue] = MAX(DimVenue[VenueName])` (or relate FactSanction to DimVenue on Venue). Same result: two bars.
