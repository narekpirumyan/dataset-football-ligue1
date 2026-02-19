# Page 5 — Fans, Stadium & Executive Summary
## "What Does It Mean for the Club?" (Strategic Wrap-Up)

**Message:** What is the overall impact of this season — competitively, financially, and commercially?

**Data Source:** Power BI Data Model (Star Schema)
- **Primary Tables:** `FactAttendance`, `FactTransfer`, `FactClubMatch`, `DimMatch`, `DimPlayer`, `DimClub`, `DimDate`
- **Relationships:** FactAttendance → DimClub (ClubKey), FactAttendance → DimDate (DateKey), FactAttendance → DimMatch (MatchKey), FactTransfer → DimClub (ArrivalClubKey, DepartureClubKey)

---

## 5.1 Attendance Trend

**Visual Type:** Line Chart or Area Chart

**Description:**
- Attendance over time (matchday or date) — per club or total

**Power BI Implementation:**

**Data Source:**
- `FactAttendance[Attendance]` - Attendance numbers
- `FactAttendance[Matchday]` - Matchday number
- `FactAttendance[DateKey]` - Date for time-based analysis
- `DimClub[ClubName]` - For filtering by club
- `DimDate[Date]` - For date-based timeline

**DAX Measures:**
```dax
Total Attendance = SUM(FactAttendance[Attendance])
Average Attendance = AVERAGE(FactAttendance[Attendance])
```

**Visual Setup - Option 1 (Line Chart by Matchday):**
1. Create line chart visual
2. X-axis: `FactAttendance[Matchday]` (sort 1-38)
3. Y-axis: `[Total Attendance]`
4. Legend: `DimClub[ClubName]` (for multiple clubs)
5. Add club slicer
6. Format: Use smooth lines, add data labels

**Visual Setup - Option 2 (Area Chart by Date):**
1. Create area chart visual
2. X-axis: `DimDate[Date]` (from FactAttendance via DateKey)
3. Y-axis: `[Total Attendance]`
4. Legend: `DimClub[ClubName]`
5. Add club slicer
6. Format: Use gradient fill, add trend line

**Visual Setup - Option 3 (HTML Content Interactive Chart):**
1. Create HTML Content visual
2. Use Chart.js for interactive line/area chart:
```html
<div class="attendance-trend">
  <h3>Attendance Trend</h3>
  <canvas id="attendance-chart"></canvas>
  <div class="controls">
    <button id="toggle-club">Show/Hide Clubs</button>
    <button id="toggle-trend">Show Trend Line</button>
  </div>
</div>
<script>
  // Chart.js implementation
  // Add interactivity: hover for details, click to filter
  // Add trend line calculation
</script>
```
3. Add interactive features: hover tooltips, click to drill down
4. Add trend line showing overall attendance trend
5. Add comparison to previous season (if data available)

---

## 5.2 Revenue per Match

**Visual Type:** KPI Card or Bar Chart

**Description:**
- Revenue (€) per match — e.g., ticket revenue or total matchday revenue

**Power BI Implementation:**

**Data Source:**
- **Note:** Revenue data is NOT available in the dataset
- `FactAttendance[Attendance]` - Can be used as proxy with estimated ticket price
- `FactAttendance[Matchday]` - Match identification
- `DimClub[ClubName]` - For filtering by club

**DAX Measures:**
```dax
Estimated Revenue per Match = 
// Proxy calculation if average ticket price is known
// [Total Attendance] * [Average Ticket Price]

// If no ticket price data:
// Document as "Not Available"
```

**Visual Setup:**
1. **If revenue data is available:**
   - Create bar chart or KPI card
   - Show revenue per matchday
   - Add club slicer
   - Format as currency

2. **If revenue data is NOT available (current case):**
   - Create informational text box or HTML Content visual
   - Display: "Revenue data not available in dataset"
   - Optionally show attendance as proxy indicator
   - Document as data gap

**Alternative - HTML Content Proxy Visualization:**
1. Create HTML Content visual
2. Show attendance as proxy for revenue potential:
```html
<div class="revenue-proxy">
  <h3>Matchday Revenue Potential</h3>
  <p class="note">Note: Actual revenue data not available. Showing attendance as proxy.</p>
  <div class="attendance-metric">
    <span class="label">Average Attendance</span>
    <span class="value">{{AverageAttendance}}</span>
  </div>
  <div class="estimated-revenue">
    <p>Estimated revenue (at €X per ticket):</p>
    <span class="value">{{EstimatedRevenue}} €</span>
  </div>
</div>
```
3. Allow user to input estimated ticket price for calculation
4. Style with CSS

---

## 5.3 Capacity Utilization %

**Visual Type:** Gauge, Bar Chart, or Line Chart

**Description:**
- Fill rate (or attendance/capacity) by matchday or by club

**Power BI Implementation:**

**Data Source:**
- `FactAttendance[FillRatePct]` - Fill rate percentage
- `FactAttendance[Attendance]` - Attendance numbers
- `FactAttendance[StadiumCapacity]` - Stadium capacity
- `FactAttendance[Matchday]` - Matchday number
- `DimClub[ClubName]` - For filtering by club

**DAX Measures:**
```dax
Average Fill Rate = AVERAGE(FactAttendance[FillRatePct])
Fill Rate = DIVIDE(SUM(FactAttendance[Attendance]), SUM(FactAttendance[StadiumCapacity]), 0) * 100
```

**Visual Setup - Option 1 (Gauge):**
1. Create gauge visual
2. Value: `[Average Fill Rate]`
3. Min: 0, Max: 100
4. Target: 80% (or league average)
5. Format: Green zone (80-100%), Yellow zone (60-80%), Red zone (<60%)
6. Add club slicer

**Visual Setup - Option 2 (Line Chart):**
1. Create line chart
2. X-axis: `FactAttendance[Matchday]` (sort 1-38)
3. Y-axis: `FactAttendance[FillRatePct]`
4. Legend: `DimClub[ClubName]`
5. Add club slicer
6. Format: Add reference line at 100% capacity

**Visual Setup - Option 3 (Bar Chart by Club):**
1. Create bar chart
2. Y-axis: `DimClub[ClubName]`
3. X-axis: `[Average Fill Rate]`
4. Format: Use conditional formatting (green for high, red for low)
5. Sort by fill rate descending

**Advanced - HTML Content Capacity Dashboard:**
1. Create HTML Content visual
2. Show comprehensive capacity analysis:
```html
<div class="capacity-dashboard">
  <div class="gauge-container">
    <h3>Average Capacity Utilization</h3>
    <div class="gauge" data-value="{{AverageFillRate}}">
      <!-- CSS/JS gauge visualization -->
    </div>
  </div>
  <div class="trend-chart">
    <canvas id="fill-rate-chart"></canvas>
  </div>
  <div class="club-comparison">
    <h3>Capacity Utilization by Club</h3>
    <table>
      <!-- Club comparison table -->
    </table>
  </div>
</div>
```
3. Use Chart.js for trend chart
4. Create custom gauge using CSS/JavaScript
5. Add club comparison table

---

## 5.4 Squad Value Rank vs Final Rank

**Visual Type:** Scatter Chart or Comparison Table

**Description:**
- Compare "squad value" rank with final league position
- Use **both**: (1) **squad market value** (DimPlayer) — valuation rank; (2) **transfer spend** (FactTransfer) — actual investment rank

**Power BI Implementation:**

**Data Source:**
- `DimPlayer[MarketValue]` - For squad market value
- `FactTransfer[AmountME]` - For transfer spend
- `FactTransfer[ArrivalClubKey]`, `FactTransfer[DepartureClubKey]` - Transfer direction
- `FactClubMatch[Points]` - For final position calculation
- `DimClub[ClubName]` - Club identification

**DAX Measures:**
```dax
Squad Market Value = SUM(DimPlayer[MarketValue])
Transfer Spend (Arrivals) = 
CALCULATE(
    SUM(FactTransfer[AmountME]),
    FactTransfer[ArrivalClubKey] = MAX(DimClub[ClubKey])
)
Total Points = SUM(FactClubMatch[Points])

Final Position = 
VAR TotalPoints = [Total Points]
VAR RankedClubs = 
    ADDCOLUMNS(
        ALL(DimClub[ClubKey], DimClub[ClubName]),
        "@Points", CALCULATE([Total Points], ALLSELECTED(DimMatch))
    )
RETURN
    RANKX(RankedClubs, [@Points], TotalPoints, DESC, Dense)

Squad Value Rank = 
VAR SquadValue = [Squad Market Value]
VAR RankedClubs = 
    ADDCOLUMNS(
        ALL(DimClub[ClubKey], DimClub[ClubName]),
        "@Value", CALCULATE([Squad Market Value], ALLSELECTED())
    )
RETURN
    RANKX(RankedClubs, [@Value], SquadValue, DESC, Dense)

Transfer Spend Rank = 
VAR TransferSpend = [Transfer Spend (Arrivals)]
VAR RankedClubs = 
    ADDCOLUMNS(
        ALL(DimClub[ClubKey], DimClub[ClubName]),
        "@Spend", CALCULATE([Transfer Spend (Arrivals)], ALLSELECTED())
    )
RETURN
    RANKX(RankedClubs, [@Spend], TransferSpend, DESC, Dense)

Rank Difference (Market Value) = [Final Position] - [Squad Value Rank]
Rank Difference (Transfer Spend) = [Final Position] - [Transfer Spend Rank]
```

**Visual Setup - Market Value Rank vs Final Rank:**
1. Create scatter chart
2. X-axis: `[Squad Value Rank]`
3. Y-axis: `[Final Position]`
4. Legend: `DimClub[ClubName]`
5. Add diagonal line (y=x) to show expected performance
6. Points above line = underperformed vs value, below = overperformed
7. Tooltips: Club name, market value, final position, rank difference

**Visual Setup - Transfer Spend Rank vs Final Rank:**
1. Create scatter chart
2. X-axis: `[Transfer Spend Rank]`
3. Y-axis: `[Final Position]`
4. Legend: `DimClub[ClubName]`
5. Add diagonal line (y=x)
6. Format: Color-code by rank difference

**Visual Setup - Comparison Table:**
1. Create matrix table
2. Rows: `DimClub[ClubName]`
3. Columns: Final Position, Squad Value Rank, Transfer Spend Rank, Rank Difference (Market Value), Rank Difference (Transfer Spend)
4. Format: Use conditional formatting for rank differences (green for positive = overperformed, red for negative = underperformed)
5. Sort by final position

**Advanced - HTML Content Rank Comparison Dashboard:**
1. Create HTML Content visual
2. Show comprehensive rank comparison:
```html
<div class="rank-comparison-dashboard">
  <h3>Squad Value vs Performance</h3>
  <div class="scatter-container">
    <canvas id="value-rank-chart"></canvas>
  </div>
  <div class="spend-scatter-container">
    <canvas id="spend-rank-chart"></canvas>
  </div>
  <div class="comparison-table">
    <table>
      <thead>
        <tr>
          <th>Club</th>
          <th>Final Position</th>
          <th>Value Rank</th>
          <th>Spend Rank</th>
          <th>Value vs Position</th>
          <th>Spend vs Position</th>
        </tr>
      </thead>
      <tbody>
        <!-- Club comparison data -->
      </tbody>
    </table>
  </div>
</div>
```
3. Use Chart.js for scatter plots
4. Add quadrant analysis (High Value/High Position, etc.)
5. Add interactive tooltips and filtering

---

## 5.5 Season Summary Panel (Success Drivers + Risks)

**Visual Type:** Multi-Card Layout or HTML Content Dashboard

**Description:**
- KPIs and short narrative: key success drivers (e.g., home form, top scorers, recruitment) and risks (injuries, discipline, financial)
- Data: Synthesised from all pages; use calculated KPIs and tables from existing visuals

**Power BI Implementation:**

**Data Source:**
- All tables from previous pages
- Calculated measures and KPIs from Pages 1-4

**DAX Measures (Summary KPIs):**
```dax
// From Page 1
Final Position = [Final Position]
Total Points = [Total Points]
Home Win Rate = [Home Win Rate]

// From Page 2
Top Scorer Goals = 
CALCULATE(
    MAX(FactPlayerSeason[Goals]),
    TOPN(1, ALL(DimPlayer[PlayerKey]), [Total Goals], DESC)
)
Top Scorer Name = 
CALCULATE(
    MAX(DimPlayer[FullName]),
    TOPN(1, ALL(DimPlayer[PlayerKey]), [Total Goals], DESC)
)

// From Page 3
Squad Value per Point = [Squad Value per Point]
Transfer Spend per Point = [Transfer Spend per Point]

// From Page 4
Net Transfer Spend = [Net Transfer Spend]
Key Players Availability = 
DIVIDE(
    SUM(FactPlayerSeason[MatchesPlayed]),
    COUNTROWS(DimPlayer) * 38,
    0
) * 100

// From Page 5
Average Fill Rate = [Average Fill Rate]
```

**Visual Setup - Option 1 (Multi-Card Layout):**
1. Create multiple KPI cards arranged in a grid:
   - **Competitive Performance:** Final Position, Total Points, Home Win Rate
   - **Player Impact:** Top Scorer (Name + Goals), Top Assister
   - **Financial Efficiency:** Squad Value per Point, Transfer Spend per Point
   - **Risk Factors:** Key Players Availability %, Total Suspensions
   - **Commercial:** Average Fill Rate, Average Attendance
2. Add club slicer
3. Format: Use color coding (green for positive, red for negative)
4. Add trend indicators if comparing to targets

**Visual Setup - Option 2 (HTML Content Executive Dashboard):**
1. Create HTML Content visual
2. Build comprehensive executive summary:
```html
<div class="executive-summary">
  <div class="summary-header">
    <h2>Season 2023-24 Executive Summary</h2>
    <div class="club-selector">
      <select id="club-filter">
        <!-- Club options -->
      </select>
    </div>
  </div>
  
  <div class="summary-sections">
    <section class="competitive-performance">
      <h3>Competitive Performance</h3>
      <div class="kpi-grid">
        <div class="kpi-card">
          <span class="label">Final Position</span>
          <span class="value">{{FinalPosition}}</span>
        </div>
        <div class="kpi-card">
          <span class="label">Total Points</span>
          <span class="value">{{TotalPoints}}</span>
        </div>
        <!-- More KPIs -->
      </div>
    </section>
    
    <section class="success-drivers">
      <h3>Key Success Drivers</h3>
      <ul>
        <li>Top Scorer: {{TopScorerName}} ({{TopScorerGoals}} goals)</li>
        <li>Home Form: {{HomeWinRate}}% win rate</li>
        <li>New Signings: Contributed {{NewSigningsContribution}}% of goals</li>
      </ul>
    </section>
    
    <section class="risks">
      <h3>Risk Factors</h3>
      <ul>
        <li>Key Players Availability: {{KeyPlayersAvailability}}%</li>
        <li>Discipline: {{TotalRedCards}} red cards, {{TotalSuspensions}} suspensions</li>
        <li>Financial: {{TransferSpendPerPoint}} M€ per point</li>
      </ul>
    </section>
    
    <section class="commercial">
      <h3>Commercial Performance</h3>
      <div class="metrics">
        <div class="metric">
          <span class="label">Average Attendance</span>
          <span class="value">{{AverageAttendance}}</span>
        </div>
        <div class="metric">
          <span class="label">Capacity Utilization</span>
          <span class="value">{{AverageFillRate}}%</span>
        </div>
      </div>
    </section>
  </div>
  
  <div class="visualizations">
    <div class="chart-container">
      <canvas id="performance-trend-chart"></canvas>
    </div>
  </div>
</div>
```
3. Use Chart.js for trend visualizations
4. Add interactive filtering by club
5. Style with professional CSS
6. Add print-friendly styling

**Visual Setup - Option 3 (Power BI Report Page):**
1. Create a dedicated summary page in Power BI
2. Arrange visuals from all pages in a single view
3. Add text boxes with narrative summaries
4. Use bookmarks to show/hide detailed sections
5. Add navigation buttons to other pages

---

## Page 5 Summary

**Tables Used:**
- `FactAttendance` - Attendance and capacity data
- `FactTransfer` - Transfer spend for rank comparison
- `FactClubMatch` - Final position and points
- `FactPlayerSeason` - Player statistics for success drivers
- `FactSanction` - Discipline metrics for risks
- `DimPlayer` - Squad value and player names
- `DimClub` - Club filtering and context
- `DimDate` - Time-based analysis
- `DimMatch` - Match context

**Key Measures:**
- Total Attendance, Average Attendance
- Average Fill Rate, Capacity Utilization %
- Final Position, Total Points
- Squad Market Value, Transfer Spend
- Squad Value Rank, Transfer Spend Rank
- Rank Difference (Market Value), Rank Difference (Transfer Spend)
- Top Scorer Goals, Top Scorer Name
- Key Players Availability %
- Summary KPIs from all pages

**Visual Types:**
- Line Charts, Area Charts (attendance trends)
- Gauges (capacity utilization)
- Scatter Charts (rank comparisons)
- KPI Cards (summary metrics)
- Tables (comparisons)
- HTML Content (comprehensive executive dashboards)

**Filtering:**
- Club slicer for single-club or league-wide view
- Date range slicer for time-based analysis
- Matchday slicer for specific periods

**Power BI Tips:**
1. Create a dedicated "Summary" measures table for easy access to KPIs
2. Use bookmarks to create interactive executive summary
3. Add tooltips to summary cards showing detailed breakdowns
4. Use conditional formatting to highlight positive/negative indicators
5. For HTML Content visuals, use Chart.js for quick implementation
6. Create print-friendly layouts for executive reports
7. Add navigation between summary and detailed pages

**HTML Content Implementation Notes:**
- Use DAX to prepare comprehensive summary data from all pages
- Create professional executive dashboard layout
- Use Chart.js for trend visualizations
- Add interactive filtering and drill-down capabilities
- Ensure responsive design for different screen sizes
- Add print-friendly CSS for reports
- Include narrative summaries alongside metrics
- Use color coding for success drivers (green) and risks (red)

**Data Quality Considerations:**
- Revenue data is not available — document as data gap or use attendance as proxy
- Ensure fill rate percentages are calculated correctly (0-100%)
- Handle NULL values in attendance data
- Market value may not be available for all players — handle in rank calculations
- Transfer spend may include zero values for free transfers — handle appropriately

**Executive Summary Best Practices:**
1. **Start with the big picture:** Final position, total points
2. **Highlight success drivers:** Top performers, key achievements
3. **Identify risks:** Availability, discipline, financial efficiency
4. **Show commercial impact:** Attendance, capacity utilization
5. **Provide context:** Compare to league averages or targets
6. **Use visual hierarchy:** Most important metrics at the top
7. **Keep it concise:** Executive summary should be scannable
8. **Enable drill-down:** Link to detailed pages for deeper analysis

---

This page serves as the strategic wrap-up, synthesizing insights from all previous pages into a comprehensive executive summary that answers: "What does this season mean for the club?"
