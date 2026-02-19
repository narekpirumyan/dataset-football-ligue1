# Page 2 — Squad Impact
## "Who Delivered the Results?" (Human Performance)

**Message:** Which players drove success, and how balanced was the squad?

**Data Source:** Power BI Data Model (Star Schema)
- **Primary Tables:** `FactPlayerSeason`, `DimPlayer`, `DimClub`
- **Relationships:** FactPlayerSeason → DimPlayer (PlayerKey), FactPlayerSeason → DimClub (ClubKey)

---

## 2.1 Goals + Assists Leaders

**Visual Type:** Bar Chart or Table

**Description:**
- Bar chart or table: top N players by goals, and by assists (or combined)
- Scope: Per club or league-wide

**Power BI Implementation:**

**Data Source:**
- `FactPlayerSeason[Goals]` - Goals scored
- `FactPlayerSeason[Assists]` - Assists provided
- `DimPlayer[FullName]` - Player names
- `DimClub[ClubName]` - For filtering by club

**DAX Measures:**
```dax
Total Goals = SUM(FactPlayerSeason[Goals])
Total Assists = SUM(FactPlayerSeason[Assists])
Goals + Assists = [Total Goals] + [Total Assists]
```

**Visual Setup - Option 1 (Bar Chart):**
1. Create bar chart visual
2. Y-axis: `DimPlayer[FullName]` (top N)
3. X-axis: `[Total Goals]` or `[Total Assists]` or `[Goals + Assists]`
4. Add club slicer for filtering
5. Sort descending to show top performers
6. Use "Top N" filter to limit to top 10/15/20 players

**Visual Setup - Option 2 (Table):**
1. Create table visual
2. Columns: `DimPlayer[FullName]`, `DimClub[ClubName]`, `[Total Goals]`, `[Total Assists]`, `[Goals + Assists]`
3. Sort by Goals + Assists descending
4. Add club slicer
5. Format: Highlight top performers with conditional formatting

**Visual Setup - Option 3 (HTML Content - Custom Leaderboard):**
1. Create HTML Content visual
2. Use DAX to create a table with top N players
3. Create custom HTML table with styling:
```html
<table class="leaderboard">
  <thead>
    <tr>
      <th>Rank</th>
      <th>Player</th>
      <th>Goals</th>
      <th>Assists</th>
      <th>Total</th>
    </tr>
  </thead>
  <tbody>
    <!-- DAX data rendered here -->
  </tbody>
</table>
```
4. Style with CSS for professional appearance
5. Add hover effects and color coding

---

## 2.2 Contribution % by Top Players

**Visual Type:** KPI Card or Gauge

**Description:**
- Percentage of team goals (or goals+assists) from top 5/10 players
- Calculation: Sum(goals) top N / Sum(goals) team

**Power BI Implementation:**

**Data Source:**
- `FactPlayerSeason[Goals]`, `FactPlayerSeason[Assists]`
- `DimClub[ClubName]` - For team context
- `DimPlayer[FullName]` - To identify top players

**DAX Measures:**
```dax
Team Total Goals = SUM(FactPlayerSeason[Goals])
Team Total Assists = SUM(FactPlayerSeason[Assists])
Team Total Contributions = [Team Total Goals] + [Team Total Assists]

Top 5 Goals = 
VAR Top5Players = 
    TOPN(
        5,
        ALL(DimPlayer[PlayerKey], DimPlayer[FullName]),
        [Total Goals],
        DESC
    )
RETURN
    CALCULATE(
        [Team Total Goals],
        Top5Players
    )

Top 5 Contribution % = DIVIDE([Top 5 Goals], [Team Total Goals], 0)

Top 10 Contribution % = 
VAR Top10Goals = 
    CALCULATE(
        [Team Total Goals],
        TOPN(
            10,
            ALL(DimPlayer[PlayerKey]),
            [Total Goals],
            DESC
        )
    )
RETURN
    DIVIDE(Top10Goals, [Team Total Goals], 0)
```

**Visual Setup:**
1. Create KPI card or gauge visual
2. Value: `[Top 5 Contribution %]` or `[Top 10 Contribution %]`
3. Format as percentage
4. Add club slicer for team-specific view
5. Add subtitle: "Top 5 players contribute X% of team goals"

**Alternative Visual - Donut Chart:**
1. Create donut chart
2. Values: Top 5 Goals, Rest of Team Goals
3. Show percentage labels
4. Format with team colors

---

## 2.3 Minutes Played Distribution

**Visual Type:** Histogram, Box Plot, or Bar Chart

**Description:**
- Distribution of minutes_played per player (per club)
- Purpose: Squad rotation and usage

**Power BI Implementation:**

**Data Source:**
- `FactPlayerSeason[MinutesPlayed]` - Minutes played per player
- `DimPlayer[FullName]` - Player identification
- `DimClub[ClubName]` - For filtering by club

**DAX Measures:**
```dax
Total Minutes = SUM(FactPlayerSeason[MinutesPlayed])
Average Minutes = AVERAGE(FactPlayerSeason[MinutesPlayed])
```

**Visual Setup - Option 1 (Histogram):**
1. Create column chart visual
2. X-axis: Create bins for MinutesPlayed (e.g., 0-500, 500-1000, 1000-1500, etc.)
3. Y-axis: Count of players in each bin
4. Add club slicer
5. Format: Use gradient colors

**Visual Setup - Option 2 (Box Plot - using HTML Content):**
1. Create HTML Content visual
2. Use DAX to calculate: Min, Q1, Median, Q3, Max minutes
3. Create custom box plot visualization:
```html
<div class="box-plot">
  <div class="box" style="left: {{Q1}}%; width: {{Q3-Q1}}%;">
    <div class="median" style="left: {{Median}}%;"></div>
  </div>
  <div class="whisker left" style="width: {{Q1-Min}}%;"></div>
  <div class="whisker right" style="width: {{Max-Q3}}%;"></div>
</div>
```
4. Use Chart.js or D3.js for professional box plots
5. Add tooltips showing exact values

**Visual Setup - Option 3 (Bar Chart):**
1. Create bar chart
2. Y-axis: `DimPlayer[FullName]`
3. X-axis: `[Total Minutes]`
4. Sort by minutes descending
5. Add club slicer
6. Use conditional formatting: Green for high usage, Yellow for medium, Red for low

---

## 2.4 Age / Nationality Structure

**Visual Type:** Histogram and Pie/Bar Chart

**Description:**
- Age distribution (histogram)
- Nationality breakdown (pie/bar chart)
- Purpose: Squad composition

**Power BI Implementation:**

**Data Source:**
- `DimPlayer[Age]` - Player age
- `DimPlayer[Nationality]` - Player nationality
- `DimClub[ClubName]` - For filtering by club

**DAX Measures:**
```dax
Average Age = AVERAGE(DimPlayer[Age])
Player Count = COUNTROWS(DimPlayer)
```

**Visual Setup - Age Distribution:**
1. Create column chart (histogram)
2. X-axis: Create age bins (e.g., 18-22, 23-27, 28-32, 33+)
3. Y-axis: Count of players (`[Player Count]`)
4. Add club slicer
5. Format: Use age-appropriate color gradient

**Visual Setup - Nationality Breakdown:**
1. Create pie chart or bar chart
2. Legend/Category: `DimPlayer[Nationality]`
3. Values: `[Player Count]`
4. Add club slicer
5. Format: Use distinct colors for each nationality
6. Show percentage labels

**Alternative - Combined HTML Content Visual:**
1. Create HTML Content visual
2. Use DAX to prepare age and nationality data
3. Create dual visualization:
```html
<div class="squad-composition">
  <div class="age-chart">
    <h3>Age Distribution</h3>
    <!-- Chart.js or D3.js histogram -->
  </div>
  <div class="nationality-chart">
    <h3>Nationality Breakdown</h3>
    <!-- Chart.js or D3.js pie chart -->
  </div>
</div>
```
4. Use Chart.js for easy implementation
5. Add interactive tooltips

---

## 2.5 Position Depth Chart

**Visual Type:** Stacked Bar Chart or Matrix

**Description:**
- Count or minutes by position (and optionally by club)
- Purpose: Depth and balance by role

**Power BI Implementation:**

**Data Source:**
- `DimPlayer[Position]` - Player position
- `FactPlayerSeason[MinutesPlayed]` or `FactPlayerSeason[MatchesPlayed]` - Depth metric
- `DimClub[ClubName]` - For filtering by club

**DAX Measures:**
```dax
Players by Position = COUNTROWS(DimPlayer)
Total Minutes by Position = SUM(FactPlayerSeason[MinutesPlayed])
```

**Visual Setup - Option 1 (Stacked Bar Chart):**
1. Create stacked bar chart
2. Y-axis: `DimPlayer[Position]`
3. X-axis: `[Total Minutes by Position]` or `[Players by Position]`
4. Add club slicer
5. Format: Use position-specific colors (e.g., Green for Goalkeeper, Blue for Defender, etc.)

**Visual Setup - Option 2 (Matrix Table):**
1. Create matrix visual
2. Rows: `DimPlayer[Position]`
3. Columns: `DimClub[ClubName]` (if showing league-wide)
4. Values: `[Players by Position]`, `[Total Minutes by Position]`
5. Format: Use conditional formatting for depth visualization

**Visual Setup - Option 3 (HTML Content - Custom Depth Chart):**
1. Create HTML Content visual
2. Use DAX to create position depth data
3. Create custom visualization:
```html
<div class="depth-chart">
  <div class="position-row">
    <span class="position-label">Goalkeeper</span>
    <div class="depth-bar" style="width: {{GoalkeeperCount}}%;">
      {{GoalkeeperCount}} players
    </div>
  </div>
  <!-- Repeat for each position -->
</div>
```
4. Style with CSS for professional appearance
5. Add hover tooltips showing player names

---

## 2.6 Player Value vs Performance Scatter

**Visual Type:** Scatter Chart

**Description:**
- X-axis: Market value
- Y-axis: Performance (goals, assists, or composite)
- Purpose: Identify high-impact vs high-cost players

**Power BI Implementation:**

**Data Source:**
- `DimPlayer[MarketValue]` - Player market value
- `FactPlayerSeason[Goals]`, `FactPlayerSeason[Assists]` - Performance metrics
- `DimPlayer[FullName]` - Player identification
- `DimClub[ClubName]` - For filtering by club

**DAX Measures:**
```dax
Market Value = SUM(DimPlayer[MarketValue])
Performance Score = [Total Goals] * 2 + [Total Assists]  // Weighted composite
```

**Visual Setup:**
1. Create scatter chart visual
2. X-axis: `[Market Value]`
3. Y-axis: `[Performance Score]` or `[Total Goals]` or `[Goals + Assists]`
4. Legend: `DimClub[ClubName]` (optional)
5. Tooltips: `DimPlayer[FullName]`, `[Total Goals]`, `[Total Assists]`, `[Market Value]`
6. Add club slicer
7. Format: Size bubbles by minutes played or matches played

**Advanced - HTML Content Scatter Plot:**
1. Create HTML Content visual
2. Use DAX to prepare player data with MarketValue and Performance
3. Create interactive scatter plot using Chart.js or D3.js:
```html
<canvas id="value-performance-chart"></canvas>
<script>
  // Chart.js scatter plot
  // Add interactivity: hover for player details
  // Add quadrant lines: High Value/High Performance, etc.
</script>
```
4. Add quadrant analysis (High Value/High Performance, Low Value/High Performance, etc.)
5. Add click events to drill into player details

---

## Page 2 Summary

**Tables Used:**
- `FactPlayerSeason` - Primary fact table for player statistics
- `DimPlayer` - Player attributes (name, position, age, nationality, market value)
- `DimClub` - Club filtering and context

**Key Measures:**
- Total Goals, Total Assists, Goals + Assists
- Team Total Goals/Assists
- Top N Contribution Percentage
- Total Minutes, Average Minutes
- Average Age, Player Count
- Market Value, Performance Score

**Visual Types:**
- Bar Charts, Tables, KPI Cards, Gauges
- Histograms, Box Plots
- Pie Charts, Stacked Bar Charts
- Scatter Charts
- HTML Content (for custom visualizations)

**Filtering:**
- Club slicer for single-club or league-wide view
- Position slicer for position-specific analysis
- Top N filter for leaderboards

**Power BI Tips:**
1. Use TOPN() function for top N calculations
2. Create calculated columns for age bins if needed
3. Use conditional formatting for performance indicators
4. For HTML Content visuals, use Chart.js or D3.js for professional charts
5. Consider creating a "Performance Score" calculated column combining goals and assists
6. Use tooltips to show additional player information
7. For market value analysis, ensure MarketValue is properly aggregated (use SUM or MAX depending on data model)

**HTML Content Implementation Notes:**
- Use DAX to prepare data tables with necessary columns
- Pass data to HTML Content visual via Power BI's data binding
- Use Chart.js for quick implementation of standard charts
- Use D3.js for custom, highly interactive visualizations
- Ensure responsive design for different screen sizes
- Add tooltips and hover effects for better user experience
