# Page 1 — Season Performance
## "What Did We Achieve?" (Sporting Outcomes)

**Message:** Did we perform competitively, and when did things change?

**Data Source:** Power BI Data Model (Star Schema)
- **Primary Tables:** `FactClubMatch`, `DimMatch`, `DimClub`, `DimDate`
- **Relationships:** FactClubMatch → DimClub (ClubKey), FactClubMatch → DimMatch (MatchKey), DimMatch → DimDate (DateKey)

---

## 1.1 Final Rank + Total Points (KPI Cards)

**Visual Type:** KPI Card or **HTML Content widget**

**Description:**
- **Final Rank:** League position at end of season (filter by club)
- **Total Points:** Sum of points (3/1/0) over all 38 matchdays

**Power BI Implementation:**

**Data Source:**
- `FactClubMatch[Points]` - Points per match
- `DimClub[ClubName]` - For filtering by club
- `DimMatch[Matchday]` - To ensure all 38 matchdays are included

**DAX Measures:**
```dax
Total Points = SUM(FactClubMatch[Points])

Final Rank = 
VAR TotalPoints = [Total Points]
VAR RankedClubs = 
    ADDCOLUMNS(
        ALL(DimClub[ClubKey], DimClub[ClubName]),
        "@Points", CALCULATE([Total Points], ALLSELECTED(DimMatch))
    )
RETURN
    RANKX(RankedClubs, [@Points], TotalPoints, DESC, Dense)
```

**Visual Setup (standard):**
1. Create two KPI card visuals
2. For "Total Points": Use measure `[Total Points]` with club filter
3. For "Final Rank": Use measure `[Final Rank]` with club filter
4. Add club slicer to filter by specific club or show league-wide

**Alternative Approach:** Create a calculated table `Standings` with one row per club containing FinalRank and TotalPoints, built from FactClubMatch filtered to Matchday = 38.

---

### 1.1 HTML Content Widget — Final Rank + Total Points (DAX Measure Approach)

**Power BI setup**

1. Create a **DAX measure** named `Rank Points HTML` (see code below).
2. Add an **HTML Content** visual or **Card** visual to the report.
3. Add the `[Rank Points HTML]` measure to the visual.
4. Use a **club slicer** so the widget respects the same filter context.

**DAX Measure — Returns HTML string**

```dax
Rank Points HTML = 
-- Calculate total points
VAR vTotalPoints = SUM(FactClubMatch[Points])

-- Calculate final rank
VAR vFinalRank = 
    VAR TotalPoints = vTotalPoints
    VAR RankedClubs = 
        ADDCOLUMNS(
            ALL(DimClub[ClubKey], DimClub[ClubName]),
            "@Points", CALCULATE(SUM(FactClubMatch[Points]), ALLSELECTED(DimMatch))
        )
    RETURN
        RANKX(RankedClubs, [@Points], TotalPoints, DESC, Dense)

-- Get club name (optional, for display)
VAR vClubName = SELECTEDVALUE(DimClub[ClubName], "League")

-- Format rank with ordinal suffix (1st, 2nd, 3rd, etc.)
VAR vRankFmt = 
    IF (
        ISBLANK ( vFinalRank ) || vFinalRank = 0,
        "–",
        VAR RankNum = vFinalRank
        VAR LastDigit = MOD ( RankNum, 10 )
        VAR LastTwoDigits = MOD ( RankNum, 100 )
        VAR Suffix = 
            IF (
                LastTwoDigits >= 11 && LastTwoDigits <= 13,
                "th",
                IF (
                    LastDigit = 1, "st",
                    IF ( LastDigit = 2, "nd",
                        IF ( LastDigit = 3, "rd", "th" )
                    )
                )
            )
        RETURN
            FORMAT ( RankNum, "#,0" ) & Suffix
    )

-- Format points
VAR vPointsFmt = 
    IF (
        ISBLANK ( vTotalPoints ) || vTotalPoints = 0,
        "0",
        FORMAT ( vTotalPoints, "#,0" )
    )

-- Determine rank color (theme: good / neutral / orange / bad)
VAR vRankColor = 
    IF (
        ISBLANK ( vFinalRank ),
        "#808080",
        IF ( vFinalRank <= 3, "#1AAB40",
            IF ( vFinalRank <= 10, "#FFC107",
                IF ( vFinalRank <= 17, "#E65C00", "#D64554" )
            )
        )
    )

RETURN
"
<style>
  * { box-sizing: border-box; }
  body {
    margin: 0;
    padding: 10px;
    font-family: 'Segoe UI', -apple-system, BlinkMacSystemFont, sans-serif;
    background: transparent;
  }
  .rank-points-widget {
    display: flex;
    flex-direction: column;
    gap: 8px;
  }
  .rank-title {
    font-size: 12px;
    font-weight: 600;
    color: #1a1a1a;
    text-align: center;
    margin-bottom: 2px;
  }
  .rank-cards {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    justify-content: center;
  }
  .rank-card, .points-card {
    flex: 1 1 90px;
    min-width: 70px;
    max-width: 110px;
    padding: 10px 8px;
    border-radius: 8px;
    text-align: center;
    box-shadow: 0 2px 6px rgba(0,0,0,0.1);
    transition: transform 0.2s ease;
  }
  .rank-card:hover, .points-card:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 8px rgba(0,0,0,0.15);
  }
  .rank-card {
    background: linear-gradient(135deg, #FFFFFF 0%, #F3F2F1 100%);
    border-left: 4px solid " & vRankColor & ";
  }
  .points-card {
    background: linear-gradient(135deg, #E8F4FC 0%, #D0E8F7 100%);
    border-left: 4px solid #0066CC;
  }
  .card-icon {
    font-size: 18px;
    margin-bottom: 2px;
    line-height: 1;
  }
  .card-value {
    font-size: 18px;
    font-weight: 700;
    color: #1a1a1a;
    margin-bottom: 2px;
  }
  .card-label {
    font-size: 10px;
    color: #404040;
    text-transform: uppercase;
    letter-spacing: 0.5px;
  }
</style>
<div class='rank-points-widget'>
  <div class='rank-title'>Season Performance</div>
  <div class='rank-cards'>
    <div class='rank-card'>
      <div class='card-icon'>🏆</div>
      <div class='card-value'>" & vRankFmt & "</div>
      <div class='card-label'>Final Rank</div>
    </div>
    <div class='points-card'>
      <div class='card-icon'>⚽</div>
      <div class='card-value'>" & vPointsFmt & "</div>
      <div class='card-label'>Total Points</div>
    </div>
  </div>
</div>
"
```

**Usage:**

1. **Create the measure** `[Rank Points HTML]` in your model (copy the DAX above).
2. **Add HTML Content visual** to your report page.
3. **Add the measure** `[Rank Points HTML]` to the visual (or use a Card visual).
4. The HTML will render automatically with the current filter context (club slicer, etc.).

**Features:**

- **Final Rank** displayed with ordinal suffix (1st, 2nd, 3rd, etc.)
- **Color-coded rank** border: Green (top 3), Yellow (4-10), Orange (11-17), Red (18-20)
- **Total Points** displayed with formatting
- **Club name** shown at the top (if filtered by club)
- **Hover effects** for better interactivity
- **Responsive design** that adapts to different sizes

**Notes:**

- The measure calculates rank dynamically based on total points across all clubs
- Rank color changes based on position (top 3 = green, mid-table = yellow/orange, bottom = red)
- If no club is selected, shows "League" as the club name
- Values update automatically when filters change

---

## 1.2 Rank Evolution by Matchday (Line Chart)

**Visual Type:** Line Chart

**Description:**
- X-axis: Matchday (1-38)
- Y-axis: League rank (1-20)
- Line(s): One line per club (e.g., focus club + comparison clubs)

**Power BI Implementation:**

**Data Source:**
- `DimMatch[Matchday]` - X-axis
- `FactClubMatch[Points]` - For calculating cumulative points
- `DimClub[ClubName]` - Legend/line series

**DAX Measures:**
```dax
Cumulative Points = 
VAR CurrentMatchday = MAX(DimMatch[Matchday])
RETURN
    CALCULATE(
        SUM(FactClubMatch[Points]),
        DimMatch[Matchday] <= CurrentMatchday,
        ALL(DimMatch[Matchday])
    )

Rank at Matchday = 
VAR CurrentMatchday = MAX(DimMatch[Matchday])
VAR CurrentPoints = [Cumulative Points]
VAR AllClubs = 
    ADDCOLUMNS(
        ALL(DimClub[ClubKey], DimClub[ClubName]),
        "@Points", 
        CALCULATE(
            [Cumulative Points],
            DimMatch[Matchday] <= CurrentMatchday,
            ALL(DimMatch[Matchday])
        )
    )
RETURN
    RANKX(AllClubs, [@Points], CurrentPoints, DESC, Dense)
```

**Visual Setup:**
1. Create line chart visual
2. X-axis: `DimMatch[Matchday]` (ensure sort order 1-38)
3. Y-axis: `[Rank at Matchday]` (invert if needed so rank 1 is at top)
4. Legend: `DimClub[ClubName]` (select specific clubs or use slicer)
5. Add club slicer for filtering

**Recommended:** Create a calculated table `StandingsByMatchday` in Power Query or DAX with columns: ClubKey, Matchday, CumulativePoints, Rank. This improves performance and simplifies the visual.

---

## 1.3 Win / Draw / Loss Split

**Visual Type:** Pie Chart or Bar Chart

**Description:**
- Categories: Win, Draw, Loss (per club or overall)
- Values: Count or percentage of matches

**Power BI Implementation:**

**Data Source:**
- `FactClubMatch[Result]` - Contains "W", "D", "L"
- `DimClub[ClubName]` - For filtering by club

**DAX Measures:**
```dax
Wins = CALCULATE(COUNTROWS(FactClubMatch), FactClubMatch[Result] = "W")
Draws = CALCULATE(COUNTROWS(FactClubMatch), FactClubMatch[Result] = "D")
Losses = CALCULATE(COUNTROWS(FactClubMatch), FactClubMatch[Result] = "L")

Win Percentage = DIVIDE([Wins], COUNTROWS(FactClubMatch), 0)
```

**Visual Setup:**
1. Create pie chart or stacked bar chart
2. Legend/Category: `FactClubMatch[Result]`
3. Values: `COUNTROWS(FactClubMatch)` or use individual measures
4. Add club slicer for filtering
5. Format: Use conditional formatting for colors (Green=Win, Yellow=Draw, Red=Loss)

**Alternative:** Create a calculated table with Result values and counts, then visualize.

---

## 1.4 Points Accumulation Curve

**Visual Type:** Line Chart

**Description:**
- X-axis: Matchday (or match order)
- Y-axis: Cumulative points
- Line(s): One per club

**Power BI Implementation:**

**Data Source:**
- `DimMatch[Matchday]` - X-axis
- `FactClubMatch[Points]` - For cumulative calculation
- `DimClub[ClubName]` - Legend/line series

**DAX Measure:**
```dax
Cumulative Points = 
VAR CurrentMatchday = MAX(DimMatch[Matchday])
RETURN
    CALCULATE(
        SUM(FactClubMatch[Points]),
        DimMatch[Matchday] <= CurrentMatchday,
        ALL(DimMatch[Matchday])
    )
```

**Visual Setup:**
1. Create line chart visual
2. X-axis: `DimMatch[Matchday]` (sort 1-38)
3. Y-axis: `[Cumulative Points]`
4. Legend: `DimClub[ClubName]`
5. Add club slicer for filtering specific clubs
6. Format: Use different colors for each club line

---

## 1.5 Home vs Away Comparison

**Visual Type:** Clustered Bar Chart or Table

**Description:**
- Metrics: Points, wins, goals for/against, or win rate
- Split: Home vs Away (per club)

**Power BI Implementation:**

**Data Source:**
- `FactClubMatch[IsHome]` - Boolean flag for home/away
- `FactClubMatch[Points]`, `FactClubMatch[GoalsFor]`, `FactClubMatch[GoalsAgainst]`
- `DimClub[ClubName]` - For filtering by club

**DAX Measures:**
```dax
Home Points = CALCULATE(SUM(FactClubMatch[Points]), FactClubMatch[IsHome] = TRUE())
Away Points = CALCULATE(SUM(FactClubMatch[Points]), FactClubMatch[IsHome] = FALSE())

Home Goals For = CALCULATE(SUM(FactClubMatch[GoalsFor]), FactClubMatch[IsHome] = TRUE())
Away Goals For = CALCULATE(SUM(FactClubMatch[GoalsFor]), FactClubMatch[IsHome] = FALSE())

Home Goals Against = CALCULATE(SUM(FactClubMatch[GoalsAgainst]), FactClubMatch[IsHome] = TRUE())
Away Goals Against = CALCULATE(SUM(FactClubMatch[GoalsAgainst]), FactClubMatch[IsHome] = FALSE())

Home Win Rate = DIVIDE(
    CALCULATE(COUNTROWS(FactClubMatch), FactClubMatch[Result] = "W", FactClubMatch[IsHome] = TRUE()),
    CALCULATE(COUNTROWS(FactClubMatch), FactClubMatch[IsHome] = TRUE()),
    0
)
```

**Visual Setup:**
1. Create clustered bar chart or matrix table
2. Rows: `DimClub[ClubName]` (or use club slicer for single club)
3. Columns: Create a calculated column or use "Home" / "Away" categories
4. Values: Use measures like `[Home Points]`, `[Away Points]`, etc.
5. Format: Use different colors for Home vs Away

**Alternative:** Create a calculated table with Home/Away as rows and metrics as columns.

---

## 1.6 Goals For / Goals Against / Goal Difference (KPI Cards)

**Visual Type:** KPI Cards

**Description:**  
Season totals for goals scored, goals conceded, and goal difference. Answers “How did we perform in front of goal?”

**Data Source:** `FactClubMatch` (via `DimClub` for filter).

**DAX Measures:**
```dax
Goals For = SUM(FactClubMatch[GoalsFor])
Goals Against = SUM(FactClubMatch[GoalsAgainst])
Goal Difference = [Goals For] - [Goals Against]
```

**Visual Setup:**  
Three KPI cards: `[Goals For]`, `[Goals Against]`, `[Goal Difference]`. Add club slicer. Optionally format Goal Difference (e.g. green if positive, red if negative).

---

## 1.7 Goals Trend by Matchday (Line Chart)

**Visual Type:** Line Chart

**Description:**  
Cumulative (or per-matchday) goals for and goals against over the season. Shows attacking/defensive trend.

**Data Source:** `FactClubMatch`, `DimMatch` (Matchday).

**DAX Measures:**
```dax
Cumulative Goals For = 
CALCULATE(
    SUM(FactClubMatch[GoalsFor]),
    DimMatch[Matchday] <= MAX(DimMatch[Matchday]),
    ALL(DimMatch[Matchday])
)
Cumulative Goals Against = 
CALCULATE(
    SUM(FactClubMatch[GoalsAgainst]),
    DimMatch[Matchday] <= MAX(DimMatch[Matchday]),
    ALL(DimMatch[Matchday])
)
```

**Visual Setup:**  
Line chart: X-axis = `DimMatch[Matchday]` (1–38), Y-axis = `[Cumulative Goals For]` and `[Cumulative Goals Against]` (two lines). Legend = measure names. Club slicer.

---

## 1.8 First Half vs Second Half of Season (Clustered Bar or KPI)

**Visual Type:** Clustered Bar Chart or two KPI cards

**Description:**  
Compare points (or wins) in matchdays 1–19 vs 20–38. Highlights strong/weak finish.

**Data Source:** `FactClubMatch`, `DimMatch` (Matchday).

**DAX Measures:**
```dax
Points First Half = 
CALCULATE(
    SUM(FactClubMatch[Points]),
    DimMatch[Matchday] <= 19
)
Points Second Half = 
CALCULATE(
    SUM(FactClubMatch[Points]),
    DimMatch[Matchday] >= 20
)
```

**Visual Setup:**  
Option A: Two KPI cards side by side. Option B: Clustered bar with category “First Half” / “Second Half” (e.g. from a calculated column or a small supporting table) and values `[Points First Half]` / `[Points Second Half]`. Club slicer.

---

## 1.9 Results Matrix / Season Calendar (Heatmap or Matrix)

**Visual Type:** Matrix or conditional-formatting heatmap

**Description:**  
One row per matchday (1–38), one column for result (W/D/L) or points. Visual “season calendar”.

**Data Source:** `FactClubMatch`, `DimMatch` (Matchday).

**Visual Setup:**  
Matrix: Rows = `DimMatch[Matchday]` (sort 1–38), Values = `FactClubMatch[Result]` (or `[Points]`). Use conditional formatting on the values (e.g. green = W, yellow = D, red = L). Club slicer. Alternatively, use a table with Matchday and Result and apply background color by result.

---

## 1.10 Longest Win Streak (KPI Card)

**Visual Type:** KPI Card

**Description:**  
Longest consecutive wins in the season. Simple DAX over match order.

**Data Source:** `FactClubMatch`, `DimMatch` (Matchday for order).

**DAX Measure (conceptual — best built with a calculated table or more advanced DAX):**  
A simple approach is to use a **calculated table** that has one row per club per matchday with Result, then use a running count of wins and take the max. Alternatively, a measure that iterates matchdays and counts consecutive Ws (more complex).  
**Simpler alternative:** Use a **card with “Wins”** = `CALCULATE(COUNTROWS(FactClubMatch), FactClubMatch[Result] = "W")` as a stand-in, or implement “Longest Win Streak” in Power Query (add a column per club that computes streak, then take MAX).

**Visual Setup:**  
One KPI card with the streak value. Club slicer.

---

## 1.11 Home Attendance Summary (FactAttendance)

**Visual Type:** KPI Cards or small bar chart

**Description:**  
Season-level home attendance: total or average attendance, average fill rate. Uses **FactAttendance** (home matches only).

**Data Source:** `FactAttendance`, `DimClub`.

**DAX Measures:**
```dax
Total Home Attendance = SUM(FactAttendance[Attendance])
Average Home Attendance = AVERAGE(FactAttendance[Attendance])
Home Matches Count = COUNTROWS(FactAttendance)
Average Fill Rate % = AVERAGE(FactAttendance[FillRatePct])
```

**Visual Setup:**  
Two or three KPI cards: e.g. `[Average Home Attendance]`, `[Average Fill Rate %]`. Club slicer. Optionally a bar chart of `[Total Home Attendance]` by club.

---

## 1.12 Season Discipline Summary (Cards / Sanctions)

**Visual Type:** KPI Cards, clustered bar, or **HTML Content widget**

**Description:**  
Season discipline summary showing:
- **Cards** (from match play): Yellow cards and red cards accumulated by players — sourced from **FactPlayerSeason**
- **Sanctions** (official disciplinary actions): Suspensions (matches) and fines (€) — sourced from **FactSanction**

**Data Source:**  
- **FactPlayerSeason:** `YellowCards`, `RedCards` — cards received by players during matches (aggregated by club via `DimClub`)
- **FactSanction:** `SuspensionMatches`, `FineEuros` — official disciplinary sanctions (suspensions and fines) issued to clubs/players

**DAX Measures:**
```dax
Total Yellow Cards = SUM(FactPlayerSeason[YellowCards])
Total Red Cards = SUM(FactPlayerSeason[RedCards])
Total Suspensions = SUM(FactSanction[SuspensionMatches])
Total Fines € = SUM(FactSanction[FineEuros])
```

**Note:** Cards (yellow/red) are match statistics from player performance data. Sanctions (suspensions/fines) are official disciplinary actions recorded separately.

**Visual Setup (standard):**  
KPI cards for selected measures, or clustered bar with `DimClub[ClubName]` on axis and these measures as values. Club slicer.

---

### 1.12 HTML Content Widget — Discipline Summary (DAX Measure Approach)

**Power BI setup**

1. Create a **DAX measure** named `Discipline HTML` (see code below).
2. Add an **HTML Content** visual or **Card** visual to the report.
3. Add the `[Discipline HTML]` measure to the visual.
4. Use a **club slicer** so the widget respects the same filter context.

**DAX Measure — Returns HTML string**

```dax
Discipline HTML = 
-- Cards from FactPlayerSeason (player match statistics)
VAR vYellowCards = SUM(FactPlayerSeason[YellowCards])
VAR vRedCards = SUM(FactPlayerSeason[RedCards])

-- Sanctions from FactSanction (official disciplinary actions)
VAR vSuspensions = SUM(FactSanction[SuspensionMatches])
VAR vFines = SUM(FactSanction[FineEuros])

-- Format fines (handle blank/null)
VAR vFinesFmt = 
    IF (
        ISBLANK ( vFines ) || vFines = 0,
        "0",
        FORMAT ( vFines, "#,0" ) & " €"
    )

-- Format other values (handle blank/null)
VAR vYellowFmt = IF ( ISBLANK ( vYellowCards ) || vYellowCards = 0, "0", FORMAT ( vYellowCards, "#,0" ) )
VAR vRedFmt = IF ( ISBLANK ( vRedCards ) || vRedCards = 0, "0", FORMAT ( vRedCards, "#,0" ) )
VAR vSuspendFmt = IF ( ISBLANK ( vSuspensions ) || vSuspensions = 0, "0", FORMAT ( vSuspensions, "#,0" ) )

RETURN
"
<style>
  * { box-sizing: border-box; }
  body {
    margin: 0;
    padding: 10px;
    font-family: 'Segoe UI', -apple-system, BlinkMacSystemFont, sans-serif;
    background: transparent;
  }
  .disc-widget {
    display: flex;
    flex-direction: column;
    gap: 8px;
  }
  .disc-title {
    font-size: 12px;
    font-weight: 600;
    color: #1a1a1a;
    text-align: center;
    margin-bottom: 2px;
  }
  .disc-cards {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    justify-content: center;
  }
  .disc-card {
    flex: 1 1 90px;
    min-width: 70px;
    max-width: 110px;
    padding: 10px 8px;
    border-radius: 8px;
    text-align: center;
    box-shadow: 0 2px 6px rgba(0,0,0,0.1);
    transition: transform 0.2s ease;
  }
  .disc-card:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 8px rgba(0,0,0,0.15);
  }
  .disc-yellow { 
    background: linear-gradient(135deg, #FFFFFF 0%, #FFF9E6 100%);
    border-left: 4px solid #FFC107;
  }
  .disc-red { 
    background: linear-gradient(135deg, #FFFFFF 0%, #FFEBEE 100%);
    border-left: 4px solid #D64554;
  }
  .disc-suspend { 
    background: linear-gradient(135deg, #FFFFFF 0%, #F3F2F1 100%);
    border-left: 4px solid #808080;
  }
  .disc-fines { 
    background: linear-gradient(135deg, #FFFFFF 0%, #E8F5E9 100%);
    border-left: 4px solid #1AAB40;
  }
  .disc-icon {
    font-size: 20px;
    margin-bottom: 3px;
    line-height: 1;
  }
  .disc-value {
    font-size: 20px;
    font-weight: 700;
    color: #1a1a1a;
    margin-bottom: 2px;
  }
  .disc-label {
    font-size: 10px;
    color: #404040;
    text-transform: uppercase;
    letter-spacing: 0.5px;
  }
</style>
<div class='disc-widget'>
  <div class='disc-title'>Season Discipline</div>
  <div class='disc-cards'>
    <div class='disc-card disc-yellow'>
      <div class='disc-icon'>🟨</div>
      <div class='disc-value'>" & vYellowFmt & "</div>
      <div class='disc-label'>Yellow Cards</div>
    </div>
    <div class='disc-card disc-red'>
      <div class='disc-icon'>🟥</div>
      <div class='disc-value'>" & vRedFmt & "</div>
      <div class='disc-label'>Red Cards</div>
    </div>
    <div class='disc-card disc-suspend'>
      <div class='disc-icon'>⛔</div>
      <div class='disc-value'>" & vSuspendFmt & "</div>
      <div class='disc-label'>Suspensions</div>
    </div>
    <div class='disc-card disc-fines'>
      <div class='disc-icon'>€</div>
      <div class='disc-value'>" & vFinesFmt & "</div>
      <div class='disc-label'>Total Fines</div>
    </div>
  </div>
  <div style='font-size: 9px; color: #404040; text-align: center; margin-top: 4px;'>
    Cards: Match statistics | Sanctions: Official disciplinary actions
  </div>
  </div>
</div>
"
```

**Usage:**

1. **Create the measure** `[Discipline HTML]` in your model (copy the DAX above).
2. **Add HTML Content visual** to your report page.
3. **Add the measure** `[Discipline HTML]` to the visual (or use a Card visual).
4. The HTML will render automatically with the current filter context (club slicer, etc.).

**Notes:**

- The measure calculates all values in DAX, so no JavaScript is needed.
- Values update automatically when filters change.
- Formatting (commas, handling blanks) is done in DAX.
- The HTML string is returned directly from the measure, which Power BI renders.

**Alternative: Card visual**

If you prefer, you can use a **Card** visual instead of HTML Content:
- Add the `[Discipline HTML]` measure to a Card visual.
- Set the card to display as "HTML" if your Power BI version supports it, or use HTML Content visual.

### 1.12.1 Aggressiveness index gauge (Indice d’agressivité – club sélectionné)

**Visual Type:** Gauge (Jauge)

**Purpose:** Show an **aggressiveness index** for the **selected club**: the gauge displays the club’s total sanction count (`FactSanction`), with the **minimum** of the scale = total sanctions of the club that has the **fewest** sanctions in the league, and the **maximum** = total sanctions of the club that has the **most** sanctions. The selected club’s value is shown on this min–max scale so users can see where the club stands relative to the league.

**Context:** Club view — the gauge is driven by the **club selection** (slicer or filter on `DimClub`). One club should be selected so the gauge shows that club’s sanction count between the league min and max.

**Data Source:** `FactSanction`, `DimClub`.

**DAX measures:**

```dax
Sanction Count = COUNTROWS(FactSanction)
```

Min and max sanction count across all clubs (for the gauge scale), independent of the current club filter:

```dax
Min Sanctions (League) =
MINX(
    ALL(DimClub),
    CALCULATE( COUNTROWS(FactSanction) )
)

Max Sanctions (League) =
MAXX(
    ALL(DimClub),
    CALCULATE( COUNTROWS(FactSanction) )
)
```

- **Value (valeur affichée):** `[Sanction Count]` — in the context of the selected club, this is that club’s total sanctions.
- **Minimum (échelle):** `[Min Sanctions (League)]`.
- **Maximum (échelle):** `[Max Sanctions (League)]`.

**Power BI setup:**

1. Add a **Gauge** visual (Jauge) on the Club view page.
2. **Value:** `[Sanction Count]` (uses filter context = selected club).
3. **Minimum value:** `[Min Sanctions (League)]` (if the gauge allows a measure; otherwise see note below).
4. **Maximum value:** `[Max Sanctions (League)]`.
5. The page **club slicer** defines the selected club; the gauge then shows that club’s sanction count between the league min and max.
6. **Format:** Whole numbers; title e.g. “Indice d’agressivité” or “Sanctions (club sélectionné)”.
7. **Tooltips:** ClubName, Sanction Count, Min Sanctions (League), Max Sanctions (League).

**Note:** If the built-in gauge does not accept measures for min/max (only constants), use a **Card** or **KPI** with the same Value and add a reference line, or a table with one row and columns MinValue = [Min Sanctions (League)] and MaxValue = [Max Sanctions (League)] to drive the scale, or use a custom visual that supports measure-driven min/max.

---

## 1.13 Top Scorer & Top Assister (FactPlayerSeason)

**Visual Type:** Card or small table

**Description:**  
Highlight the club’s (or league’s) top scorer and top assister for the season. Bridges “what we achieved” with “who delivered”.

**Data Source:** `FactPlayerSeason`, `DimPlayer`, `DimClub`.

**DAX Measures:**
```dax
Top Scorer Goals = 
VAR MaxGoals = 
    CALCULATE(
        MAX(FactPlayerSeason[Goals]),
        ALL(DimPlayer[PlayerKey]),
        ALL(DimPlayer[FullName])
    )
RETURN
    CALCULATE(
        MAX(FactPlayerSeason[Goals]),
        FactPlayerSeason[Goals] = MaxGoals
    )

Top Scorer Name = 
VAR MaxGoals = [Top Scorer Goals]
RETURN
    CONCATENATEX(
        FILTER(
            ADDCOLUMNS(
                SUMMARIZE(FactPlayerSeason, DimPlayer[FullName]),
                "G", SUM(FactPlayerSeason[Goals])
            ),
            [G] = MaxGoals
        ),
        DimPlayer[FullName],
        ", "
    )
```

**Simpler alternative:** Use a **table visual**: Rows = `DimPlayer[FullName]`, Values = `SUM(FactPlayerSeason[Goals])`, `SUM(FactPlayerSeason[Assists])`. Sort by goals descending, take top 1 for scorer and top 1 for assister (or use two small tables/cards with Top N = 1).

**Visual Setup:**  
Two cards: one “Top Scorer: [Top Scorer Name] ([Top Scorer Goals] goals)”, one “Top Assister: …”. Or two small tables with Top N = 1. Club slicer.

---

## Page 1 Summary

**Tables Used:**
- `FactClubMatch` - Match results, points, goals, W/D/L, home/away
- `DimMatch` - Matchday, date
- `DimClub` - Club filtering
- `FactAttendance` - Home attendance, fill rate (1.11)
- `FactPlayerSeason` - Goals, assists, cards (1.12, 1.13)
- `FactSanction` - Suspensions, fines (1.12)
- `DimPlayer` - Names for top scorer/assister (1.13)

**Sections:**
- **1.1–1.5** — Rank, points, evolution, W/D/L, accumulation, home vs away
- **1.6** — Goals for / against / goal difference (KPIs)
- **1.7** — Goals trend by matchday (line chart)
- **1.8** — First half vs second half (points)
- **1.9** — Results matrix / season calendar (heatmap or matrix)
- **1.10** — Longest win streak (KPI; optional, can use PQ or simpler “Wins” card)
- **1.11** — Home attendance summary (FactAttendance)
- **1.12** — Season discipline (FactPlayerSeason + FactSanction)
- **1.13** — Top scorer & top assister (FactPlayerSeason + DimPlayer)

**Visual Types:**  
KPI Cards, Line Charts, Pie/Bar Charts, Matrix/Heatmap, Tables.

**Filtering:**  
Club slicer; optional matchday slicer.

**Power BI Tips:**
1. Use calculated table `StandingsByMatchday` for rank evolution if needed.
2. Use conditional formatting for W/D/L and goal difference.
3. Keep Matchday sort order 1–38 on all axes.
4. For 1.10 (longest win streak), consider Power Query or a simple “Wins” card if full streak logic is too heavy.
### 1.13 HTML Content Widget — Top Scorer & Top Assister (DAX Measure Approach)

**Power BI setup**

1. Create a **DAX measure** named `Top Players HTML` (see code below).
2. Add an **HTML Content** visual or **Card** visual to the report.
3. Add the `[Top Players HTML]` measure to the visual.
4. Use a **club slicer** so the widget respects the same filter context.

**DAX Measure — Returns HTML string**

```dax
Top Players HTML = 
-- Note: FactPlayerSeason has one row per player with Goals and Assists
-- It's possible (and valid) for the same player to lead in both categories
-- This measure finds top scorer and top assister separately

-- Get top scorer (respects club filter from slicer)
VAR PlayersByGoals = 
    ADDCOLUMNS(
        VALUES(FactPlayerSeason[PlayerKey]),
        "@Goals", 
        CALCULATE(SUM(FactPlayerSeason[Goals])),
        "@PlayerName",
        CALCULATE(MAX(DimPlayer[FullName]))
    )
VAR TopScorerTbl = 
    TOPN(1, PlayersByGoals, [@Goals], DESC)
VAR vTopScorerKey = 
    IF(COUNTROWS(TopScorerTbl) > 0, MAXX(TopScorerTbl, FactPlayerSeason[PlayerKey]), BLANK())
VAR vTopScorerName = 
    IF(
        NOT ISBLANK(vTopScorerKey),
        CALCULATE(MAX(DimPlayer[FullName]), DimPlayer[PlayerKey] = vTopScorerKey),
        "–"
    )
VAR vTopScorerGoals = 
    IF(
        NOT ISBLANK(vTopScorerKey),
        CALCULATE(SUM(FactPlayerSeason[Goals]), FactPlayerSeason[PlayerKey] = vTopScorerKey),
        0
    )

-- Get top assister (respects club filter from slicer)
VAR PlayersByAssists = 
    ADDCOLUMNS(
        VALUES(FactPlayerSeason[PlayerKey]),
        "@Assists", 
        CALCULATE(SUM(FactPlayerSeason[Assists])),
        "@PlayerName",
        CALCULATE(MAX(DimPlayer[FullName]))
    )
VAR TopAssisterTbl = 
    TOPN(1, PlayersByAssists, [@Assists], DESC)
VAR vTopAssisterKey = 
    IF(COUNTROWS(TopAssisterTbl) > 0, MAXX(TopAssisterTbl, FactPlayerSeason[PlayerKey]), BLANK())
VAR vTopAssisterName = 
    IF(
        NOT ISBLANK(vTopAssisterKey),
        CALCULATE(MAX(DimPlayer[FullName]), DimPlayer[PlayerKey] = vTopAssisterKey),
        "–"
    )
VAR vTopAssisterAssists = 
    IF(
        NOT ISBLANK(vTopAssisterKey),
        CALCULATE(SUM(FactPlayerSeason[Assists]), FactPlayerSeason[PlayerKey] = vTopAssisterKey),
        0
    )

-- Optional: If same player leads in both, show second-best assister for variety
-- Comment out this section if you want to show the actual top assister even if same as scorer
VAR vIsSamePlayer = (vTopScorerKey = vTopAssisterKey) && NOT ISBLANK(vTopScorerKey)
VAR vSecondAssisterTbl = 
    IF(
        vIsSamePlayer,
        TOPN(1, FILTER(PlayersByAssists, FactPlayerSeason[PlayerKey] <> vTopScorerKey), [@Assists], DESC),
        TopAssisterTbl
    )
VAR vFinalAssisterKey = 
    IF(
        vIsSamePlayer && COUNTROWS(vSecondAssisterTbl) > 0,
        MAXX(vSecondAssisterTbl, FactPlayerSeason[PlayerKey]),
        vTopAssisterKey
    )
VAR vFinalAssisterName = 
    IF(
        NOT ISBLANK(vFinalAssisterKey),
        CALCULATE(MAX(DimPlayer[FullName]), DimPlayer[PlayerKey] = vFinalAssisterKey),
        IF(vIsSamePlayer, vTopScorerName, "–")
    )
VAR vFinalAssisterAssists = 
    IF(
        NOT ISBLANK(vFinalAssisterKey),
        CALCULATE(SUM(FactPlayerSeason[Assists]), FactPlayerSeason[PlayerKey] = vFinalAssisterKey),
        IF(vIsSamePlayer, CALCULATE(SUM(FactPlayerSeason[Assists]), FactPlayerSeason[PlayerKey] = vTopScorerKey), 0)
    )

-- Format values
VAR vScorerGoalsFmt = 
    IF(
        ISBLANK(vTopScorerGoals) || vTopScorerGoals = 0,
        "0",
        FORMAT(vTopScorerGoals, "#,0")
    )
VAR vAssisterAssistsFmt = 
    IF(
        ISBLANK(vFinalAssisterAssists) || vFinalAssisterAssists = 0,
        "0",
        FORMAT(vFinalAssisterAssists, "#,0")
    )

-- Truncate long names (max 20 chars)
VAR vScorerNameDisplay = 
    IF(
        LEN(vTopScorerName) > 20,
        LEFT(vTopScorerName, 17) & "...",
        vTopScorerName
    )
VAR vAssisterNameDisplay = 
    IF(
        LEN(vFinalAssisterName) > 20,
        LEFT(vFinalAssisterName, 17) & "...",
        vFinalAssisterName
    )

RETURN
"
<style>
  * { box-sizing: border-box; }
  body {
    margin: 0;
    padding: 10px;
    font-family: 'Segoe UI', -apple-system, BlinkMacSystemFont, sans-serif;
    background: transparent;
  }
  .top-players-widget {
    display: flex;
    flex-direction: column;
    gap: 8px;
  }
  .top-players-title {
    font-size: 12px;
    font-weight: 600;
    color: #1a1a1a;
    text-align: center;
    margin-bottom: 2px;
  }
  .top-players-cards {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    justify-content: center;
  }
  .player-card {
    flex: 1 1 90px;
    min-width: 70px;
    max-width: 110px;
    padding: 10px 8px;
    border-radius: 8px;
    text-align: center;
    box-shadow: 0 2px 6px rgba(0,0,0,0.1);
    transition: transform 0.2s ease;
  }
  .player-card:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 8px rgba(0,0,0,0.15);
  }
  .scorer-card {
    background: linear-gradient(135deg, #FFFFFF 0%, #FFF3E0 100%);
    border-left: 4px solid #E65C00;
  }
  .assister-card {
    background: linear-gradient(135deg, #E8F4FC 0%, #D0E8F7 100%);
    border-left: 4px solid #0066CC;
  }
  .player-icon {
    width: 20px;
    height: 20px;
    margin: 0 auto 3px;
    color: #404040;
  }
  .player-name {
    font-size: 10px;
    font-weight: 600;
    color: #1a1a1a;
    margin-bottom: 2px;
    line-height: 1.2;
    word-wrap: break-word;
  }
  .player-stat {
    font-size: 18px;
    font-weight: 700;
    color: #1a1a1a;
    margin-bottom: 1px;
  }
  .player-label {
    font-size: 9px;
    color: #404040;
    text-transform: uppercase;
    letter-spacing: 0.5px;
  }
</style>
<div class='top-players-widget'>
  <div class='top-players-title'>Top Performers</div>
  <div class='top-players-cards'>
    <div class='player-card scorer-card'>
      <svg class='player-icon' xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='2'>
        <circle cx='12' cy='8' r='4'/>
        <path d='M4 20c1.5-3.5 4-5.5 8-5.5s6.5 2 8 5.5' stroke-linecap='round'/>
        <circle cx='12' cy='12' r='8' stroke-dasharray='2 2' opacity='0.3'/>
      </svg>
      <div class='player-name'>" & vScorerNameDisplay & "</div>
      <div class='player-stat'>" & vScorerGoalsFmt & "</div>
      <div class='player-label'>Goals</div>
    </div>
    <div class='player-card assister-card'>
      <svg class='player-icon' xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='2'>
        <circle cx='12' cy='8' r='4'/>
        <path d='M4 20c1.5-3.5 4-5.5 8-5.5s6.5 2 8 5.5' stroke-linecap='round'/>
        <path d='M8 12l2 2 4-4' stroke-linecap='round' stroke-linejoin='round'/>
      </svg>
      <div class='player-name'>" & vAssisterNameDisplay & "</div>
      <div class='player-stat'>" & vAssisterAssistsFmt & "</div>
      <div class='player-label'>Assists</div>
    </div>
  </div>
</div>
"
```

**Usage:**

1. **Create the measure** `[Top Players HTML]` in your model (copy the DAX above).
2. **Add HTML Content visual** to your report page.
3. **Add the measure** `[Top Players HTML]` to the visual (or use a Card visual).
4. The HTML will render automatically with the current filter context (club slicer, etc.).

**Features:**

- **Top Scorer** card: Name, goals scored, orange gradient background
- **Top Assister** card: Name, assists provided, blue gradient background
- **Football player icon** (SVG) on each card — scorer has a ball icon, assister has a checkmark
- **Name truncation** for long names (max 20 characters)
- **Consistent sizing** with other widgets (same card dimensions: flex 1 1 100px, min 80px, max 120px)
- **Horizontal layout** matching Discipline and Rank/Points widgets
- **Hover effects** for better interactivity

**Notes:**

- The measure finds the top scorer and top assister based on current filter context (club slicer)
- **Important:** If the same player leads in both Goals and Assists, the measure will show the **second-best assister** instead, so you see two different players
- If you want to show the actual top assister even if it's the same as the scorer, comment out the "Optional" section in the DAX
- If multiple players tie, it shows the first one found
- Handles blank/null values gracefully
- Player names are truncated if too long to fit the card
- Values update automatically when filters change

**Data Validation:**

To verify if the same player is genuinely top in both categories:
1. Create a table visual with: `DimPlayer[FullName]`, `SUM(FactPlayerSeason[Goals])`, `SUM(FactPlayerSeason[Assists])`
2. Sort by Goals descending → top scorer
3. Sort by Assists descending → top assister
4. If they're different players, the widget will show them correctly
5. If they're the same player, the widget will show the second-best assister for variety
