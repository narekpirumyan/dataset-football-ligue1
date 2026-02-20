# Page 4 — Transfers, Availability & Risk
## "What Influenced the Season?" (Change & Risk)

**Message:** What external factors shaped performance — recruitment or disruptions?

**Data Source:** Power BI Data Model (Star Schema)
- **Primary Tables:** `FactTransfer`, `FactPlayerSeason`, `FactSanction`, `DimPlayer`, `DimClub`, `DimDate`
- **Relationships:** FactTransfer → DimClub (ArrivalClubKey, DepartureClubKey), FactTransfer → DimPlayer (PlayerKey), FactSanction → DimClub (ClubKey), FactSanction → DimPlayer (PlayerKey)

---

## 4.1 Transfers In/Out Summary (+ Net Spend, Spend by Window)

**Visual Type:** Clustered Bar Chart or Matrix Table

**Description:**
- Count and/or total fee: transfers IN vs OUT per club
- **Also (transfer data):** Net transfer spend per club (spend on arrivals − fees received from departures); total spend (sum of AmountME for arrivals); spend by window (summer 2023 vs winter 2024)

**Power BI Implementation:**

**Data Source:**
- `FactTransfer[AmountME]` - Transfer fees
- `FactTransfer[ArrivalClubKey]` - Transfers in
- `FactTransfer[DepartureClubKey]` - Transfers out
- `FactTransfer[TransferType]` - Transfer type
- `FactTransfer[TransferDateKey]` - For window filtering
- `DimClub[ClubName]` - Club identification
- `DimDate[Month]`, `DimDate[Year]` - For window classification

**DAX Measures:**
```dax
Transfers In Count = 
CALCULATE(
    COUNTROWS(FactTransfer),
    FactTransfer[ArrivalClubKey] = MAX(DimClub[ClubKey])
)

Transfers Out Count = 
CALCULATE(
    COUNTROWS(FactTransfer),
    FactTransfer[DepartureClubKey] = MAX(DimClub[ClubKey])
)

Transfer Spend (Arrivals) = 
CALCULATE(
    SUM(FactTransfer[AmountME]),
    FactTransfer[ArrivalClubKey] = MAX(DimClub[ClubKey])
)

Transfer Revenue (Departures) = 
CALCULATE(
    SUM(FactTransfer[AmountME]),
    FactTransfer[DepartureClubKey] = MAX(DimClub[ClubKey])
)

Net Transfer Spend = [Transfer Spend (Arrivals)] - [Transfer Revenue (Departures)]

Summer Window Spend = 
CALCULATE(
    SUM(FactTransfer[AmountME]),
    FactTransfer[ArrivalClubKey] = MAX(DimClub[ClubKey]),
    DimDate[Month] IN {7, 8, 9},  // July, August, September
    DimDate[Year] = 2023
)

Winter Window Spend = 
CALCULATE(
    SUM(FactTransfer[AmountME]),
    FactTransfer[ArrivalClubKey] = MAX(DimClub[ClubKey]),
    DimDate[Month] IN {1, 2},  // January, February
    DimDate[Year] = 2024
)
```

**Visual Setup - Transfers In/Out Count:**
1. Create clustered bar chart
2. Y-axis: `DimClub[ClubName]`
3. X-axis: `[Transfers In Count]`, `[Transfers Out Count]`
4. Format: Use different colors for In vs Out
5. Add club slicer

**Visual Setup - Net Spend:**
1. Create bar chart
2. Y-axis: `DimClub[ClubName]`
3. X-axis: `[Net Transfer Spend]`
4. Format: Use conditional formatting (green for positive net spend, red for negative)
5. Add club slicer

**Visual Setup - Spend by Window:**
1. Create stacked bar chart
2. Y-axis: `DimClub[ClubName]`
3. X-axis: `[Summer Window Spend]`, `[Winter Window Spend]`
4. Format: Use different colors for summer vs winter
5. Add club slicer

**Advanced - HTML Content Transfer Dashboard:**
1. Create HTML Content visual
2. Create comprehensive transfer summary:
```html
<div class="transfer-dashboard">
  <div class="transfer-summary">
    <h3>Transfers Summary</h3>
    <div class="metrics">
      <div class="metric">
        <span class="label">Transfers In</span>
        <span class="value">{{TransfersInCount}}</span>
      </div>
      <div class="metric">
        <span class="label">Transfers Out</span>
        <span class="value">{{TransfersOutCount}}</span>
      </div>
      <div class="metric">
        <span class="label">Net Spend</span>
        <span class="value">{{NetSpend}} M€</span>
      </div>
    </div>
  </div>
  <div class="window-comparison">
    <canvas id="window-spend-chart"></canvas>
  </div>
</div>
```
3. Use Chart.js for window comparison chart
4. Add interactive tooltips

---

## 4.2 New Signings Contribution

**Visual Type:** Bar Chart or Table

**Description:**
- Goals/assists/minutes from players who "joined" during the season (or in summer 2023)

**Power BI Implementation:**

**Data Source:**
- `FactTransfer[ArrivalClubKey]` - To identify new signings
- `FactTransfer[TransferDateKey]` - To filter by transfer window
- `FactPlayerSeason[Goals]`, `FactPlayerSeason[Assists]`, `FactPlayerSeason[MinutesPlayed]` - Contribution metrics
- `DimPlayer[FullName]` - Player identification
- `DimClub[ClubName]` - For filtering by club

**DAX Measures:**
```dax
New Signings Goals = 
VAR NewSignings = 
    FILTER(
        ALL(DimPlayer[PlayerKey]),
        CALCULATE(
            COUNTROWS(FactTransfer),
            FactTransfer[ArrivalClubKey] = MAX(DimClub[ClubKey]),
            DimDate[Year] = 2023,
            DimDate[Month] IN {7, 8, 9}  // Summer window
        ) > 0
    )
RETURN
    CALCULATE(
        SUM(FactPlayerSeason[Goals]),
        NewSignings
    )

New Signings Assists = 
// Similar logic as above, using Assists

New Signings Minutes = 
// Similar logic as above, using MinutesPlayed

New Signings Contribution % = 
DIVIDE(
    [New Signings Goals] + [New Signings Assists],
    [Team Total Goals] + [Team Total Assists],
    0
)
```

**Alternative Approach - Calculated Column:**
Create a calculated column `IsNewSigning` on `FactPlayerSeason` or `DimPlayer`:
```dax
IsNewSigning = 
CALCULATE(
    COUNTROWS(FactTransfer),
    FactTransfer[PlayerKey] = DimPlayer[PlayerKey],
    FactTransfer[ArrivalClubKey] = FactPlayerSeason[ClubKey],
    DimDate[Year] = 2023,
    DimDate[Month] IN {7, 8, 9}
) > 0
```

Then use:
```dax
New Signings Goals = 
CALCULATE(
    SUM(FactPlayerSeason[Goals]),
    DimPlayer[IsNewSigning] = TRUE()
)
```

**Visual Setup:**
1. Create bar chart or table
2. Rows: `DimPlayer[FullName]` (filtered to new signings)
3. Values: `[New Signings Goals]`, `[New Signings Assists]`, `[New Signings Minutes]`
4. Add club slicer
5. Format: Highlight top contributors

**Advanced - HTML Content New Signings Analysis:**
1. Create HTML Content visual
2. Show new signings with their contributions:
```html
<div class="new-signings-analysis">
  <h3>New Signings Contribution</h3>
  <div class="summary">
    <p>New signings contributed {{NewSigningsContribution}}% of team goals+assists</p>
  </div>
  <table class="signings-table">
    <thead>
      <tr>
        <th>Player</th>
        <th>Transfer Fee</th>
        <th>Goals</th>
        <th>Assists</th>
        <th>Minutes</th>
        <th>ROI</th>
      </tr>
    </thead>
    <tbody>
      <!-- New signings data -->
    </tbody>
  </table>
</div>
```
3. Calculate ROI (Return on Investment) = (Goals + Assists) / Transfer Fee
4. Style with CSS
5. Add sorting and filtering capabilities

---

## 4.3 Injury Timeline vs Results

**Visual Type:** Line Chart or HTML Content Timeline

**Description:**
- Timeline of "unavailable" players (injuries) and overlay with match results or points

**Power BI Implementation:**

**Data Source:**
- `FactPlayerSeason[MatchesMissed]` - Proxy for unavailability
- `FactPlayerSeason[MatchesPlayed]` - To calculate matches missed
- `FactClubMatch[Points]` - Match results
- `DimMatch[Matchday]`, `DimMatch[MatchDate]` - Timeline
- `DimPlayer[FullName]` - Player identification
- `DimClub[ClubName]` - For filtering by club

**Note:** True injury data (dates, injury types) is not available in the dataset. Use `MatchesMissed` as a proxy for player unavailability.

**DAX Measures:**
```dax
Key Players Unavailable = 
VAR KeyPlayers = 
    TOPN(
        5,
        ALL(DimPlayer[PlayerKey]),
        [Total Goals] + [Total Assists],
        DESC
    )
RETURN
    CALCULATE(
        SUM(FactPlayerSeason[MatchesMissed]),
        KeyPlayers
    )

Points per Matchday = 
CALCULATE(
    SUM(FactClubMatch[Points]),
    VALUES(DimMatch[Matchday])
)
```

**Visual Setup - Option 1 (Line Chart):**
1. Create line chart with dual Y-axis
2. X-axis: `DimMatch[Matchday]`
3. Y-axis 1: `[Key Players Unavailable]` (left axis)
4. Y-axis 2: `[Points per Matchday]` (right axis)
5. Add club slicer
6. Format: Use different colors and line styles

**Visual Setup - Option 2 (HTML Content Timeline):**
1. Create HTML Content visual
2. Create interactive timeline:
```html
<div class="injury-timeline">
  <div class="timeline-container">
    <div class="timeline-line"></div>
    <div class="timeline-events">
      <!-- Matchday events with unavailable players -->
      <div class="event" style="left: {{MatchdayPercentage}}%;">
        <div class="event-marker"></div>
        <div class="event-details">
          <span class="matchday">MD {{Matchday}}</span>
          <span class="unavailable">{{UnavailablePlayers}} key players unavailable</span>
          <span class="points">Points: {{Points}}</span>
        </div>
      </div>
    </div>
  </div>
</div>
```
3. Use D3.js or Chart.js for timeline visualization
4. Add hover tooltips showing player names
5. Overlay match results

**Note:** This is a proxy visualization. Document that true injury data is not available.

---

## 4.4 Matches Missed by Key Players

**Visual Type:** Bar Chart or Table

**Description:**
- For key players (e.g., top by goals/assists), show matches_played vs 38 → "matches missed"

**Power BI Implementation:**

**Data Source:**
- `FactPlayerSeason[MatchesPlayed]` - Matches played
- `FactPlayerSeason[MatchesMissed]` - Matches missed (calculated as 38 - MatchesPlayed)
- `FactPlayerSeason[Goals]`, `FactPlayerSeason[Assists]` - To identify key players
- `DimPlayer[FullName]` - Player identification
- `DimClub[ClubName]` - For filtering by club

**DAX Measures:**
```dax
Total Goals = SUM(FactPlayerSeason[Goals])
Total Assists = SUM(FactPlayerSeason[Assists])
Matches Missed = SUM(FactPlayerSeason[MatchesMissed])
Matches Played = SUM(FactPlayerSeason[MatchesPlayed])
```

**Visual Setup:**
1. Create bar chart or table
2. Filter to top N players by goals+assists
3. Y-axis/Rows: `DimPlayer[FullName]`
4. X-axis/Values: `[Matches Missed]`, `[Matches Played]`
5. Use stacked bar to show both
6. Add club slicer
7. Format: Use conditional formatting (red for high matches missed)

**Alternative - Waterfall Chart:**
1. Create waterfall chart
2. Starting value: 38 (total matches)
3. Negative values: `[Matches Missed]` per player
4. Final value: `[Matches Played]`
5. Show key players only

**Advanced - HTML Content Availability Dashboard:**
1. Create HTML Content visual
2. Show availability matrix:
```html
<div class="availability-dashboard">
  <h3>Key Players Availability</h3>
  <table class="availability-matrix">
    <thead>
      <tr>
        <th>Player</th>
        <th>Matches Played</th>
        <th>Matches Missed</th>
        <th>Availability %</th>
        <th>Impact</th>
      </tr>
    </thead>
    <tbody>
      <!-- Key players data -->
    </tbody>
  </table>
  <div class="availability-chart">
    <canvas id="availability-chart"></canvas>
  </div>
</div>
```
3. Calculate Availability % = (Matches Played / 38) * 100
4. Use Chart.js for visual representation
5. Add color coding: Green for high availability, Red for low

---

## 4.5 Cards & Impact on Lost Points

**Visual Type:** Bar Chart or Scatter Chart

**Description:**
- Discipline: red cards / suspensions and correlation with results (e.g., points in matches with red, or after suspension)

**Power BI Implementation:**

**Data Source:**
- `FactSanction[SuspensionMatches]` - Suspension duration
- `FactSanction[SanctionType]` - Type of sanction
- `FactSanction[SanctionDateKey]` - Sanction date
- `FactPlayerSeason[RedCards]`, `FactPlayerSeason[YellowCards]` - Card counts
- `FactClubMatch[Points]` - Match results
- `DimMatch[Matchday]` - Match context
- `DimPlayer[FullName]` - Player identification
- `DimClub[ClubName]` - For filtering by club

**DAX Measures:**
```dax
Total Red Cards = SUM(FactPlayerSeason[RedCards])
Total Yellow Cards = SUM(FactPlayerSeason[YellowCards])
Total Suspensions = SUM(FactSanction[SuspensionMatches])

Points After Red Card = 
// Requires match-level red card data (may need calculated table)
// Approximate: average points in matches where player received red card

Total Fines = SUM(FactSanction[FineEuros])

Discipline Score = 
[Total Red Cards] * 3 + [Total Yellow Cards] * 1  // Weighted score
```

**Visual Setup - Cards Summary:**
1. Create clustered bar chart
2. Y-axis: `DimClub[ClubName]`
3. X-axis: `[Total Red Cards]`, `[Total Yellow Cards]`
4. Format: Use red for red cards, yellow for yellow cards
5. Add club slicer

**Visual Setup - Suspensions:**
1. Create bar chart
2. Y-axis: `DimClub[ClubName]` or `DimPlayer[FullName]`
3. X-axis: `[Total Suspensions]`
4. Format: Use conditional formatting
5. Add club/player slicer

**Visual Setup - Fines:**
1. Create bar chart
2. Y-axis: `DimClub[ClubName]`
3. X-axis: `[Total Fines]`
4. Format: Show as currency
5. Add club slicer

**Advanced - HTML Content Discipline Analysis:**
1. Create HTML Content visual
2. Show comprehensive discipline analysis:
```html
<div class="discipline-dashboard">
  <div class="summary-cards">
    <div class="card">
      <h4>Total Red Cards</h4>
      <div class="value">{{TotalRedCards}}</div>
    </div>
    <div class="card">
      <h4>Total Suspensions</h4>
      <div class="value">{{TotalSuspensions}}</div>
    </div>
    <div class="card">
      <h4>Total Fines</h4>
      <div class="value">{{TotalFines}} €</div>
    </div>
  </div>
  <div class="discipline-chart">
    <canvas id="discipline-chart"></canvas>
  </div>
  <div class="suspension-timeline">
    <h4>Suspension Timeline</h4>
    <!-- Timeline of suspensions -->
  </div>
</div>
```
3. Use Chart.js for visualizations
4. Create timeline of suspensions
5. Add correlation analysis with match results

**Note:** Direct correlation between suspensions and "lost points" requires match-level data on which players were suspended. Use aggregate analysis as proxy.

---

## Page 4 Summary

**Tables Used:**
- `FactTransfer` - Transfer events, fees, windows
- `FactPlayerSeason` - Player availability (matches missed/played)
- `FactSanction` - Disciplinary sanctions, suspensions, fines
- `FactClubMatch` - Match results and points
- `DimPlayer` - Player identification
- `DimClub` - Club filtering and context
- `DimDate` - Transfer windows, sanction dates
- `DimMatch` - Match context for timeline analysis

**Key Measures:**
- Transfers In/Out Count, Transfer Spend, Transfer Revenue
- Net Transfer Spend, Summer/Winter Window Spend
- New Signings Goals/Assists/Minutes, New Signings Contribution %
- Key Players Unavailable, Matches Missed
- Total Red Cards, Total Yellow Cards, Total Suspensions
- Total Fines, Discipline Score

**Visual Types:**
- Clustered Bar Charts (transfers in/out, cards)
- Stacked Bar Charts (spend by window, matches played/missed)
- Line Charts (timeline analysis)
- Tables (detailed breakdowns)
- HTML Content (comprehensive dashboards, timelines)

**Filtering:**
- Club slicer for single-club or league-wide view
- Transfer window slicer (summer/winter) using DimDate
- Transfer type slicer (Purchase, Loan, Free transfer)
- Player slicer for individual player analysis

**Power BI Tips:**
1. Create calculated columns for transfer windows (summer/winter) if needed
2. Use TOPN() to identify key players for availability analysis
3. Handle NULL values in suspension matches and fines
4. For HTML Content visuals, use Chart.js or D3.js for timelines
5. Create calculated tables for match-level analysis if needed
6. Use proxy metrics (matches missed) when true injury data is unavailable
7. Document data limitations (e.g., no true injury timeline)

**HTML Content Implementation Notes:**
- Use DAX to prepare comprehensive transfer and availability data
- Create interactive timelines for injury/availability analysis
- Use Chart.js for standard charts (bar, line, pie)
- Use D3.js for custom timeline visualizations
- Add tooltips showing detailed player and match information
- Create summary cards for key metrics
- Ensure responsive design for different screen sizes

**Data Quality Considerations:**
- Transfer dates may span multiple seasons — filter to 2023-24 season
- Some transfers may be loans or free transfers — handle zero fees appropriately
- Matches missed is calculated (38 - MatchesPlayed) — may not reflect true injuries
- Suspension data may have NULL values — handle in measures
- True injury timeline data is not available — use matches missed as proxy
