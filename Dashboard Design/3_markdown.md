# Page 3 — Financial Efficiency
## "Did We Spend Smart?" (Business Performance)

**Message:** Did financial investment translate into performance?

**Data Source:** Power BI Data Model (Star Schema)
- **Primary Tables:** `FactTransfer`, `FactClubMatch`, `DimPlayer`, `DimClub`, `DimDate`
- **Relationships:** FactTransfer → DimClub (ArrivalClubKey, DepartureClubKey), FactTransfer → DimPlayer (PlayerKey), FactClubMatch → DimClub (ClubKey)

**Note:** Market value represents squad valuation (what the squad is worth), not actual expenses. Transfer data provides actual money spent/received. Both are used together for comprehensive financial analysis.

---

## 3.1 Squad Market Value vs League Position (+ Transfer Spend vs Position)

**Visual Type:** Scatter Chart or Clustered Bar Chart

**Description:**
- **Market Value Visual:** Total market value per club vs final league position (or points) — valuation vs results
- **Transfer Spend Visual:** Transfer spend (net spend or spend on arrivals) per club vs final position — actual investment vs results

**Power BI Implementation:**

**Data Source:**
- `DimPlayer[MarketValue]` - Player market values
- `FactTransfer[AmountME]` - Transfer fees
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

Transfer Revenue (Departures) = 
CALCULATE(
    SUM(FactTransfer[AmountME]),
    FactTransfer[DepartureClubKey] = MAX(DimClub[ClubKey])
)

Net Transfer Spend = [Transfer Spend (Arrivals)] - [Transfer Revenue (Departures)]

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
```

**Visual Setup - Market Value vs Position:**
1. Create scatter chart
2. X-axis: `[Squad Market Value]`
3. Y-axis: `[Final Position]` (invert if needed so position 1 is at top)
4. Legend: `DimClub[ClubName]`
5. Tooltips: Club name, market value, points, position
6. Format: Size bubbles by total points

**Visual Setup - Transfer Spend vs Position:**
1. Create scatter chart
2. X-axis: `[Net Transfer Spend]` or `[Transfer Spend (Arrivals)]`
3. Y-axis: `[Final Position]`
4. Legend: `DimClub[ClubName]`
5. Tooltips: Club name, transfer spend, net spend, position
6. Format: Use different colors for positive vs negative net spend

**Alternative - Combined HTML Content Visual:**
1. Create HTML Content visual
2. Use DAX to prepare data with both market value and transfer spend
3. Create dual-axis scatter plot:
```html
<canvas id="financial-performance-chart"></canvas>
<script>
  // Chart.js scatter plot with two datasets
  // Market Value vs Position (blue dots)
  // Transfer Spend vs Position (red dots)
  // Add trend lines
  // Add quadrant analysis
</script>
```
4. Add quadrant lines: High Investment/High Performance, Low Investment/High Performance, etc.
5. Add interactive tooltips and click events

---

## 3.2 Squad Value per Point & Transfer Spend per Point (KPIs)

**Visual Type:** KPI Cards or Gauge

**Description:**
- **Squad Value per Point:** Sum(market value per club) / total points — value efficiency (squad worth per point; not actual cost)
- **Transfer Spend per Point:** Sum(transfer spend) / total points — actual cost per point (use transfer data)
- Scope: Per club; show both when data allows

**Power BI Implementation:**

**Data Source:**
- `DimPlayer[MarketValue]` - For squad value
- `FactTransfer[AmountME]` - For transfer spend
- `FactClubMatch[Points]` - For total points
- `DimClub[ClubName]` - For filtering by club

**DAX Measures:**
```dax
Squad Market Value = SUM(DimPlayer[MarketValue])
Transfer Spend (Arrivals) = 
CALCULATE(
    SUM(FactTransfer[AmountME]),
    FactTransfer[ArrivalClubKey] = MAX(DimClub[ClubKey])
)
Total Points = SUM(FactClubMatch[Points])

Squad Value per Point = 
DIVIDE([Squad Market Value], [Total Points], BLANK())

Transfer Spend per Point = 
DIVIDE([Transfer Spend (Arrivals)], [Total Points], BLANK())
```

**Visual Setup:**
1. Create two KPI card visuals side by side
2. First KPI: `[Squad Value per Point]`
   - Format as currency (M€ per point)
   - Add subtitle: "Squad Value Efficiency"
   - Add trend indicator if comparing to average
3. Second KPI: `[Transfer Spend per Point]`
   - Format as currency (M€ per point)
   - Add subtitle: "Transfer Cost Efficiency"
   - Add trend indicator
4. Add club slicer for filtering
5. Format: Use conditional formatting (green for low cost per point = efficient, red for high cost)

**Alternative - Gauge Visual:**
1. Create gauge visual
2. Value: `[Transfer Spend per Point]`
3. Min/Max: Set based on league range
4. Target: League average
5. Format: Green zone for efficient, red zone for inefficient

**Advanced - HTML Content Efficiency Dashboard:**
1. Create HTML Content visual
2. Display both metrics with visual indicators:
```html
<div class="efficiency-dashboard">
  <div class="metric">
    <h3>Squad Value per Point</h3>
    <div class="value">{{SquadValuePerPoint}} M€</div>
    <div class="efficiency-bar">
      <div class="bar" style="width: {{EfficiencyPercentage}}%;"></div>
    </div>
  </div>
  <div class="metric">
    <h3>Transfer Spend per Point</h3>
    <div class="value">{{TransferSpendPerPoint}} M€</div>
    <div class="efficiency-bar">
      <div class="bar" style="width: {{EfficiencyPercentage}}%;"></div>
    </div>
  </div>
</div>
```
3. Add comparison to league average
4. Use color coding for efficiency levels

---

## 3.3 Market Value vs Contribution Scatter

**Visual Type:** Scatter Chart

**Description:**
- X-axis: Market value
- Y-axis: Goals+assists or other contribution metric
- Scope: Per player — valuation vs output (not pay vs output; salary not in dataset)

**Power BI Implementation:**

**Data Source:**
- `DimPlayer[MarketValue]` - Player market value
- `FactPlayerSeason[Goals]`, `FactPlayerSeason[Assists]` - Contribution metrics
- `DimPlayer[FullName]` - Player identification
- `DimClub[ClubName]` - For filtering by club

**DAX Measures:**
```dax
Market Value = SUM(DimPlayer[MarketValue])
Total Goals = SUM(FactPlayerSeason[Goals])
Total Assists = SUM(FactPlayerSeason[Assists])
Goals + Assists = [Total Goals] + [Total Assists]
```

**Visual Setup:**
1. Create scatter chart visual
2. X-axis: `[Market Value]`
3. Y-axis: `[Goals + Assists]` or `[Total Goals]`
4. Legend: `DimClub[ClubName]` (optional)
5. Tooltips: `DimPlayer[FullName]`, `[Market Value]`, `[Total Goals]`, `[Total Assists]`
6. Add club slicer
7. Format: Size bubbles by minutes played
8. Add trend line to show correlation

**Advanced - HTML Content Scatter with Quadrants:**
1. Create HTML Content visual
2. Use Chart.js or D3.js for interactive scatter plot
3. Add quadrant lines:
   - High Value / High Performance (top right)
   - Low Value / High Performance (top left) — "Bargains"
   - High Value / Low Performance (bottom right) — "Overpaid"
   - Low Value / Low Performance (bottom left)
4. Add player labels on hover
5. Add filtering by club or position

---

## 3.4 Market Value vs Actual Production (+ Fee Paid vs Market Value)

**Visual Type:** Scatter Chart or Comparison Table

**Description:**
- Compare market value (or transfer fee as proxy) to output (goals, assists, minutes)
- **Also (transfer data):** For transferred players: fee paid (at transfer) vs current market value — value creation since signing

**Power BI Implementation:**

**Data Source:**
- `DimPlayer[MarketValue]` - Current market value
- `FactTransfer[AmountME]` - Transfer fee paid
- `FactTransfer[ArrivalClubKey]` - To identify transferred players
- `FactPlayerSeason[Goals]`, `FactPlayerSeason[Assists]`, `FactPlayerSeason[MinutesPlayed]` - Production metrics
- `DimPlayer[FullName]` - Player identification

**DAX Measures:**
```dax
Market Value = SUM(DimPlayer[MarketValue])

Transfer Fee Paid = 
CALCULATE(
    SUM(FactTransfer[AmountME]),
    FactTransfer[ArrivalClubKey] = MAX(DimClub[ClubKey])
)

Value Creation = [Market Value] - [Transfer Fee Paid]

Value Creation % = 
DIVIDE([Value Creation], [Transfer Fee Paid], 0)

Production Score = [Total Goals] * 2 + [Total Assists] + ([Total Minutes] / 100)
```

**Visual Setup - Market Value vs Production:**
1. Create scatter chart
2. X-axis: `[Market Value]`
3. Y-axis: `[Production Score]` or `[Goals + Assists]`
4. Tooltips: Player name, market value, goals, assists, minutes
5. Add club slicer
6. Format: Add trend line

**Visual Setup - Fee Paid vs Market Value (Transferred Players Only):**
1. Create scatter chart
2. Filter to players with transfers (FactTransfer has rows)
3. X-axis: `[Transfer Fee Paid]`
4. Y-axis: `[Market Value]`
5. Add diagonal line (y=x) to show break-even
6. Points above line = value creation, below = value loss
7. Tooltips: Player name, fee paid, current value, value creation

**Advanced - HTML Content Comparison Dashboard:**
1. Create HTML Content visual
2. Show side-by-side comparisons:
```html
<div class="value-analysis">
  <div class="chart-container">
    <h3>Market Value vs Production</h3>
    <canvas id="value-production-chart"></canvas>
  </div>
  <div class="chart-container">
    <h3>Fee Paid vs Current Value</h3>
    <canvas id="fee-value-chart"></canvas>
  </div>
  <div class="top-value-creators">
    <h3>Top Value Creators</h3>
    <table>
      <!-- Players with highest Value Creation -->
    </table>
  </div>
</div>
```
3. Use Chart.js for both scatter plots
4. Add interactive filtering and tooltips

---

## 3.5 Most Value-Efficient Players (Market Value or Transfer Fee)

**Visual Type:** Bar Chart or Table

**Description:**
- Contribution per unit of value — e.g., (goals+assists) / market_value or (goals+assists) / transfer_fee
- Use market value for valuation-based efficiency; use transfer fee (from transfers file) for spend-based efficiency where applicable

**Power BI Implementation:**

**Data Source:**
- `DimPlayer[MarketValue]` - For market value efficiency
- `FactTransfer[AmountME]` - For transfer fee efficiency
- `FactPlayerSeason[Goals]`, `FactPlayerSeason[Assists]` - Contribution metrics
- `DimPlayer[FullName]` - Player identification
- `DimClub[ClubName]` - For filtering by club

**DAX Measures:**
```dax
Market Value = SUM(DimPlayer[MarketValue])
Total Goals = SUM(FactPlayerSeason[Goals])
Total Assists = SUM(FactPlayerSeason[Assists])
Goals + Assists = [Total Goals] + [Total Assists]

Efficiency vs Market Value = 
DIVIDE([Goals + Assists], [Market Value], BLANK())

Transfer Fee Paid = 
CALCULATE(
    SUM(FactTransfer[AmountME]),
    FactTransfer[ArrivalClubKey] = MAX(DimClub[ClubKey])
)

Efficiency vs Transfer Fee = 
DIVIDE([Goals + Assists], [Transfer Fee Paid], BLANK())
```

**Visual Setup - Market Value Efficiency:**
1. Create bar chart or table
2. Y-axis/Rows: `DimPlayer[FullName]` (top N)
3. X-axis/Values: `[Efficiency vs Market Value]`
4. Sort descending
5. Add club slicer
6. Format: Use conditional formatting (green for high efficiency)
7. Filter out players with zero market value or zero contributions

**Visual Setup - Transfer Fee Efficiency:**
1. Create bar chart or table
2. Filter to players with transfers (FactTransfer has rows)
3. Y-axis/Rows: `DimPlayer[FullName]` (top N)
4. X-axis/Values: `[Efficiency vs Transfer Fee]`
5. Sort descending
6. Add club slicer
7. Format: Highlight "bargain" players

**Advanced - HTML Content Efficiency Leaderboard:**
1. Create HTML Content visual
2. Create dual leaderboard:
```html
<div class="efficiency-leaderboard">
  <div class="market-value-efficiency">
    <h3>Most Efficient by Market Value</h3>
    <table>
      <thead>
        <tr>
          <th>Rank</th>
          <th>Player</th>
          <th>Goals+Assists</th>
          <th>Market Value</th>
          <th>Efficiency</th>
        </tr>
      </thead>
      <tbody>
        <!-- Top N players -->
      </tbody>
    </table>
  </div>
  <div class="transfer-fee-efficiency">
    <h3>Most Efficient by Transfer Fee</h3>
    <table>
      <!-- Similar structure -->
    </table>
  </div>
</div>
```
3. Style with CSS
4. Add hover effects and color coding
5. Add comparison to league average

---

## Page 3 Summary

**Tables Used:**
- `FactTransfer` - Transfer fees and transfer events
- `FactClubMatch` - Points and league position
- `FactPlayerSeason` - Player production metrics
- `DimPlayer` - Market value and player attributes
- `DimClub` - Club filtering and context
- `DimDate` - Transfer date filtering (optional)

**Key Measures:**
- Squad Market Value, Transfer Spend (Arrivals), Transfer Revenue (Departures)
- Net Transfer Spend
- Squad Value per Point, Transfer Spend per Point
- Market Value, Transfer Fee Paid
- Value Creation, Value Creation %
- Efficiency vs Market Value, Efficiency vs Transfer Fee
- Production Score, Goals + Assists

**Visual Types:**
- Scatter Charts (market value vs position, value vs production)
- KPI Cards, Gauges (efficiency metrics)
- Bar Charts, Tables (efficiency leaderboards)
- HTML Content (custom financial dashboards, dual visualizations)

**Filtering:**
- Club slicer for single-club or league-wide view
- Transfer window slicer (summer/winter) using DimDate
- Transfer type slicer (Purchase, Loan, Free transfer)

**Power BI Tips:**
1. Handle NULL values in transfer fees (free transfers, loans)
2. Use DIVIDE() function to avoid division by zero errors
3. Create calculated columns for transfer windows (summer/winter) if needed
4. For HTML Content visuals, use Chart.js for quick implementation
5. Add trend lines to scatter plots to show correlation
6. Use conditional formatting to highlight efficient vs inefficient clubs/players
7. Consider creating a "Transfer Window" calculated column on FactTransfer for filtering
8. For value creation analysis, filter to players who have both transfer fees and current market values

**HTML Content Implementation Notes:**
- Use DAX to prepare comprehensive financial data tables
- Create dual visualizations (market value + transfer spend) for comparison
- Add quadrant analysis in scatter plots (High Investment/High Performance, etc.)
- Use Chart.js or D3.js for interactive financial charts
- Add tooltips showing detailed financial metrics
- Create efficiency leaderboards with conditional styling
- Ensure currency formatting (M€) is consistent

**Data Quality Considerations:**
- Market value may not be available for all players — handle NULLs appropriately
- Transfer fees may be zero for free transfers or loans — distinguish in analysis
- Some transfers may be from/to external clubs — filter or handle separately
- Transfer dates may span multiple seasons — filter to 2023-24 season window
