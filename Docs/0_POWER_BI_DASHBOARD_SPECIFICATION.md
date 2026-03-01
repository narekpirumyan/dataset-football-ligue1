# Power BI Dashboard Specification — Ligue 1 Season 2023–2024

**Purpose:** This document defines the Power BI dashboard structure, pages, and visuals as implemented and documented in **Dashboard Design**. It maps each view to the data model and to the detailed design documents.

**Context:** The dashboard supports single-club analysis (Club View), individual player analysis (Player View), and league-wide analysis (Ligue-Wide View), using a star-schema data model (see **Docs/0_markdown.md** and **Data_Model.bim**).

**Design reference:** All visuals and DAX are described in the markdown files under **Dashboard Design/** (Club_View, Player_View, Ligue-Wide_View). This specification summarises the structure; for implementation details, measures, and setup, refer to those documents.

---

## 1. Data model (summary)

The dashboard is built on a **star schema** implemented in Power BI (**Data_Model.bim** = source of truth). Main elements:

| Type | Tables |
|------|--------|
| **Dimensions** | `DimClub`, `DimDate`, `DimPlayer`, `DimMatch` |
| **Facts** | `FactClubMatch`, `FactPlayerSeason`, `FactAttendance`, `FactSanction`, `FactTransfer` |
| **Calculated table** | `RivalryPair` (one row per unordered pair of clubs; used in Rivalities) |

**Key relationships:** Fact tables link to dimensions via keys (e.g. ClubKey, PlayerKey, MatchKey, DateKey). DimMatch links to DimDate via DateKey. FactClubMatch has two rows per match (one per club). See **Docs/0_markdown.md** for full schema.

**Source data (ETL):** Match results, player stats, stadium attendance, disciplinary sanctions, and transfers are loaded and transformed via Power Query into the star schema; see ETL and measures documentation where applicable.

---

## 2. Dashboard structure overview

| View | Pages / sections | Design document(s) |
|------|-------------------|--------------------|
| **Club View** | 5 pages (Season Performance, Squad Impact, Financial Efficiency, Transfers & Risk, Fans & Summary) | `Dashboard Design/Club_View/1_Season_Performance.md`, `2_markdown.md`, `3_markdown.md`, `4_markdown.md`, `5_markdown.md` |
| **Player View** | 1 page (Player card: KPIs vs position average) | `Dashboard Design/Player_View/1_Player_KPI_vs_Position.md` |
| **Ligue-Wide View** | 4 sections (Performance, Offensive-Defensive, Rivalities, Sanctions) | `Dashboard Design/Ligue-Wide_View/Performance.md`, `Offensive-Defensive.md`, `Rivalities.md`, `Sanctions.md` |

**Themes and visuals:** `Dashboard Design/theme_arena_light.json`; visual charter in `Dashboard Design/6_Visual_Charter.md`. Club logos: `Dashboard Design/club_logos.json` or `DimClub[Club_URL]` (where used). No per-club colour mapping is used; visuals use the report theme palette.

---

## 3. Club View (single-club analysis)

**Scope:** Analyse one club (or compare a few) on sporting results, squad, finances, transfers/risk, and fans/stadium. Filter by `DimClub[ClubName]` (slicer).

### 3.1 Page 1 — Season Performance  
*“What did we achieve?” (sporting outcomes)*

**Message:** Did we perform competitively, and when did things change?

**Reference:** `Dashboard Design/Club_View/1_Season_Performance.md`

**Data source:** Power BI data model — `FactClubMatch`, `DimMatch`, `DimClub`, `DimDate`. Relationships: FactClubMatch → DimClub (ClubKey), FactClubMatch → DimMatch (MatchKey), DimMatch → DimDate (DateKey).

---

#### 3.1.1 Final rank + total points (KPI cards)

| Visual | Description |
|--------|-------------|
| Final rank | League position at end of season (e.g. filter by club) |
| Total points | Sum of points (3/1/0) over all 38 matchdays |

**Data required:**

| Field | Use | Notes |
|-------|-----|--------|
| `DimClub[ClubName]` | Identify club | Filter for “our” club |
| `FactClubMatch[Points]` | Result (W/D/L → 3/1/0) | Sum over matchdays |
| `DimMatch[Matchday]` | Season scope | 1–38 |

**Measures:** `Total Points = SUM(FactClubMatch[Points])`; `Final Rank` (e.g. RANKX over clubs by Total Points, DESC, Dense). Optional: HTML Content visual with measure `Rank Points HTML` (see design doc).

**Data quality checks:**
- [ ] All 38 matchdays present per club
- [ ] No duplicate MatchKey per club
- [ ] Consistent club keys across FactClubMatch and DimClub

---

#### 3.1.2 Rank evolution by matchday (line chart)

| Visual | Description |
|--------|-------------|
| X-axis | Matchday (1–38) |
| Y-axis | League rank (1–20) |
| Line(s) | One line per club (e.g. focus club + comparison) |

**Data required:**

| Field | Use |
|-------|-----|
| `DimMatch[MatchKey]`, `DimMatch[Matchday]`, `DimMatch[DateKey]` | Order and scope |
| `FactClubMatch[ClubKey]`, `FactClubMatch[Points]`, `FactClubMatch[MatchKey]` | Compute points per match, then cumulative points per club per matchday |
| — | **Derived:** cumulative points after each matchday → rank per matchday |

**Measures:** `Cumulative Points` (e.g. CALCULATE(SUM(Points), DimMatch[Matchday] <= CurrentMatchday)); `Rank at Matchday` (RANKX over clubs by cumulative points at that matchday).

**Data quality checks:**
- [ ] Matchdays sequential; no missing matchdays in date range
- [ ] Date formats consistent (DimDate / DateKey)

---

#### 3.1.3 Win / Draw / Loss split

| Visual | Description |
|--------|-------------|
| Chart type | Pie or bar (count or % of matches) |
| Categories | Win, Draw, Loss (per club or overall) |

**Data required:**

| Field | Use |
|-------|-----|
| `FactClubMatch[ClubKey]`, `FactClubMatch[Result]` | Result per row (W/D/L); filter by club |

**Measures:** Count or percentage of rows where Result = "W", "D", "L" in context of selected club.

**Data quality checks:**
- [ ] Result in { "W", "D", "L" }
- [ ] W + D + L = 38 per club (or matches played)

---

#### 3.1.4 Points accumulation curve

| Visual | Description |
|--------|-------------|
| X-axis | Matchday (or match order) |
| Y-axis | Cumulative points |
| Line(s) | One per club |

**Data required:** Same as 3.1.2 — `DimMatch[Matchday]`, `FactClubMatch[ClubKey]`, `FactClubMatch[Points]` to compute running total per club.

**Data quality checks:** Same as 3.1.2.

---

#### 3.1.5 Home vs away comparison

| Visual | Description |
|--------|-------------|
| Metrics | Points, wins, goals for/against, or win rate |
| Split | Home vs Away (per club) |

**Data required:**

| Field | Use |
|-------|-----|
| `FactClubMatch[IsHome]` | Tag match as home or away for the club |
| `FactClubMatch[Points]`, `FactClubMatch[GoalsFor]`, `FactClubMatch[GoalsAgainst]` | Result and goals for/against |

**Data quality checks:**
- [ ] Each club has 19 home and 19 away matches
- [ ] No duplicate matches (each MatchKey twice in FactClubMatch, once per club)

---

#### 3.1.6 Form timeline (last 5 matches trend)

| Visual | Description |
|--------|-------------|
| Display | Per matchday (or date), show result of “last 5” (e.g. W/W/D/L/W) or running form metric |
| Purpose | Momentum and streaks |

**Data required:**

| Field | Use |
|-------|-----|
| `DimMatch[Matchday]`, `DimMatch[DateKey]` | Order matches |
| `FactClubMatch[ClubKey]`, `FactClubMatch[Result]` | Result (W/D/L) per match for the club |

**Data quality checks:**
- [ ] Chronological order consistent (DateKey vs Matchday)
- [ ] No gaps so “last 5” is well-defined

---

#### 3.1.7 Performance vs strong/weak opponents (optional)

| Visual | Description |
|--------|-------------|
| Idea | Points or goals vs “top 6” vs “bottom 6” (by final rank) |

**Data required:** Match results + classification of opponent strength (e.g. from computed final table / Final Rank measure). Filter or segment matches by opponent rank band.

**Data quality:** Depends on correct standings and club key alignment.

---

#### Page 1 — Table & field summary

| Table | Fields used |
|-------|-------------|
| `FactClubMatch` | MatchKey, ClubKey, Points, GoalsFor, GoalsAgainst, Result, IsHome |
| `DimMatch` | MatchKey, Matchday, DateKey |
| `DimClub` | ClubKey, ClubName |
| `DimDate` | DateKey, Date (where needed) |

**Implementation details (DAX, HTML widgets):** See `Dashboard Design/Club_View/1_Season_Performance.md`.

---

### 3.2 Page 2 — Squad Impact  
*“Who delivered the results?” (human performance)*

**Message:** Which players drove success, and how balanced was the squad?

**Reference:** `Dashboard Design/Club_View/2_markdown.md`

**Data source:** Power BI data model — `FactPlayerSeason`, `DimPlayer`, `DimClub`. Relationships: FactPlayerSeason → DimPlayer (PlayerKey), FactPlayerSeason → DimClub (ClubKey).

---

#### 3.2.1 Goals + assists leaders

| Visual | Description |
|--------|-------------|
| Chart | Bar chart or table: top N players by goals, and by assists (or combined) |
| Scope | Per club or league-wide |

**Data required:**

| Field | Use |
|-------|-----|
| `DimPlayer[PlayerKey]`, `DimPlayer[FullName]`, `DimClub[ClubName]` | Identify player and filter by club |
| `FactPlayerSeason[Goals]` | Goals leaderboard |
| `FactPlayerSeason[Assists]` | Assists leaderboard |

**Measures:** `Total Goals = SUM(FactPlayerSeason[Goals])`, `Total Assists = SUM(FactPlayerSeason[Assists])`, `Goals + Assists = [Total Goals] + [Total Assists]`.

**Data quality checks:**
- [ ] Numeric type for goals/assists; no text in numeric fields
- [ ] Normalise any trailing spaces or comma decimals in source data

---

#### 3.2.2 Contribution % by top players

| Visual | Description |
|--------|-------------|
| Idea | % of team goals (or goals+assists) from top 5/10 players |
| Calculation | Sum(goals) top N / Sum(goals) team |

**Data required:** `DimClub[ClubName]`, `FactPlayerSeason[Goals]`, `FactPlayerSeason[Assists]` — aggregate by club, then by top players (e.g. TOPN or RANKX).

**Data quality checks:**
- [ ] Total team goals consistent with match results (optional cross-check with FactClubMatch)
- [ ] No negative goals/assists

---

#### 3.2.3 Minutes played distribution

| Visual | Description |
|--------|-------------|
| Chart | Distribution (histogram, box, or bar) of minutes_played per player (e.g. per club) |
| Purpose | Squad rotation and usage |

**Data required:**

| Field | Use |
|-------|-----|
| `FactPlayerSeason[MinutesPlayed]`, `DimClub[ClubName]`, `DimPlayer[PlayerKey]` | Distribution and filters |

**Data quality checks:**
- [ ] `MinutesPlayed` numeric; no mixed units in source
- [ ] No nulls where relevant; max minutes plausible (e.g. ≤ 38×90 ≈ 3420)

---

#### 3.2.4 Age / nationality structure

| Visual | Description |
|--------|-------------|
| Chart | Age distribution (e.g. histogram); nationality breakdown (pie/bar) |
| Purpose | Squad composition |

**Data required:** `DimPlayer`: `Age` (or `DateOfBirth`), `Nationality` (or country) per player. From model or extended player master.

**Data quality:** Add or use player master (PlayerKey, Age, Nationality) in model; align with FactPlayerSeason via PlayerKey.

---

#### 3.2.5 Position depth chart

| Visual | Description |
|--------|-------------|
| Chart | Count or minutes by position (and optionally by club) |
| Purpose | Depth and balance by role |

**Data required:**

| Field | Use |
|-------|-----|
| `DimPlayer[Position]`, `DimClub[ClubName]` | Group by position; filter by club |
| `FactPlayerSeason[MinutesPlayed]` or `FactPlayerSeason[MatchesPlayed]` | Depth (e.g. total minutes per position) |

**Data quality checks:**
- [ ] Position values consistent (controlled vocabulary); typo check
- [ ] All players have a position

---

#### 3.2.6 Player value vs performance scatter

| Visual | Description |
|--------|-------------|
| Axes | X: Market value, Y: performance (goals, assists, or composite) |
| Purpose | Identify high-impact vs high-cost players |

**Data required:** From `DimPlayer`: `MarketValue`; from `FactPlayerSeason`: `Goals`, `Assists` (and optionally `MinutesPlayed`). Join by PlayerKey.

**Data quality:** Ensure `MarketValue` is numeric and aligned with PlayerKey; same scope (e.g. season) for stats.

---

#### Page 2 — Table & field summary

| Table | Fields used |
|-------|-------------|
| `FactPlayerSeason` | PlayerKey, ClubKey, Goals, Assists, MinutesPlayed, MatchesPlayed, Starts, (YellowCards, RedCards, Shots, ShotsOnTarget, CleanSheets, Saves, SuccessfulDribbles, Interceptions, SuccessfulTackles as needed) |
| `DimPlayer` | PlayerKey, FullName, Position, (Age, Nationality, MarketValue) |
| `DimClub` | ClubKey, ClubName |

**Missing for full spec (if not in model):** Age, Nationality, MarketValue — add to DimPlayer or separate player master.

**Implementation details (DAX, HTML leaderboard):** See `Dashboard Design/Club_View/2_markdown.md`.

---

### 3.3 Page 3 — Financial Efficiency  
*“Did we spend smart?” (business performance)*

**Reference:** `Dashboard Design/Club_View/3_markdown.md`

| Visual | Type | Description |
|--------|------|-------------|
| Squad market value vs league position | Scatter or bar | Total market value per club vs final position (or points). |
| Transfer spend vs position | Scatter or bar | Transfer spend (e.g. net or arrivals) per club vs final position. |
| Squad value per point / transfer spend per point | KPI cards | Value efficiency and actual cost per point. |
| Market value vs contribution | Scatter | Per player: market value vs goals+assists (or other output). |
| Fee paid vs market value (transferred players) | Scatter or table | For arrivals: fee at transfer vs current market value. |
| Most value-efficient players | Table or bar | (Goals+assists) / market value or / transfer fee. |

**Key tables/measures:** `DimPlayer` (MarketValue), `FactTransfer`, `FactClubMatch`, `DimClub`. Measures: Squad Market Value, Transfer Spend (Arrivals), Transfer Revenue (Departures), Net Transfer Spend, Total Points, Final Position, value per point.

---

### 3.4 Page 4 — Transfers, Availability & Risk  
*“What influenced the season?” (change & risk)*

**Reference:** `Dashboard Design/Club_View/4_markdown.md`

| Visual | Type | Description |
|--------|------|-------------|
| Transfers in/out summary | Clustered bar or matrix | Count and/or total fee: IN vs OUT per club; net spend; optional by window (summer/winter). |
| New signings contribution | Table or bar | Goals/assists/minutes from players who joined (e.g. summer 2023). |
| Injury timeline vs results | Timeline (if data) | Unavailable players vs match results; **gap** if no injury dataset (proxy: matches missed). |
| Matches missed by key players | Table or bar | For key players: matches_played vs 38 → matches missed. |
| Cards & impact on lost points | Table or cards | Discipline (reds, suspensions) and link to results where possible. |

**Key tables/measures:** `FactTransfer`, `FactPlayerSeason`, `FactSanction`, `DimPlayer`, `DimClub`, `DimDate`. Measures: Transfers In/Out count, Transfer Spend/Revenue, Net Spend, window-based measures.

---

### 3.5 Page 5 — Fans, Stadium & Executive Summary  
*“What does it mean for the club?” (strategic wrap-up)*

**Reference:** `Dashboard Design/Club_View/5_markdown.md`

| Visual | Type | Description |
|--------|------|-------------|
| Attendance trend | Line or area chart | Attendance over time (matchday or date), per club or total. |
| Revenue per match | KPI or bar | **Gap** if no revenue data; document as N/A or proxy (e.g. attendance × ticket price if added). |
| Capacity utilization % | Line or bar | Fill rate (or attendance/capacity) by matchday or by club. |
| Squad value rank vs final rank | Table or scatter | Compare squad value rank (market value and/or transfer spend) with final league position. |
| Season summary panel | KPI + narrative | Key success drivers and risks; synthesised from other pages. |

**Key tables/measures:** `FactAttendance`, `DimMatch`, `DimClub`, `DimDate`; transfer and market value measures for value rank.

---

## 4. Player View (single-player analysis)

**Scope:** One selected player; compare their metrics to the league average or to the average of players in the same position.

**Reference:** `Dashboard Design/Player_View/1_Player_KPI_vs_Position.md`

### 4.1 Player card — 9 gauge charts (KPIs vs position average)

| Visual | Type | Description |
|--------|------|-------------|
| 9 gauge charts | Gauge | One gauge per metric: **Saves**, **Interceptions**, **Shots**, **Shots on target**, **Goals**, **Assists**, **Dribbles**, **Tackles**, **Red cards**. Each shows: **value** = sum for the selected player; **target** = average (all players, or same position if position filter applied); **min/max** = scale (global or same position). |

**Selection:** Slicer on `DimPlayer[FullName]` (single selection). Optional slicer on `DimPlayer[Position]`: when used, target and min/max switch to same-position.

**Key tables/measures:** `FactPlayerSeason`, `DimPlayer`. Measures: Player &lt;Metric&gt; (SUM), Avg &lt;Metric&gt; All, Avg &lt;Metric&gt; Same Position; Min/Max for gauge scale.

---

## 5. Ligue-Wide View (league-level analysis)

**Scope:** All clubs; performance, attack/defence, rivalries, and sanctions. No single-club filter unless stated.

### 5.1 Performance page

**Reference:** `Dashboard Design/Ligue-Wide_View/Performance.md`

| Visual | Type | Description |
|--------|------|-------------|
| Goal difference by club | **Grouped bar chart** | Goal difference (goals for − goals against) per club. |
| Win rate by club | **Stacked bar chart** | Win rate (%) per club (wins / matches played). |
| Points per match by club | **Stacked bar chart** | Average points per match per club. |
| Rank at matchday (top 5 clubs) | **Line chart** | X = Matchday, Y = rank; one line per club for the top 5 (e.g. by final points). |

**Key tables/measures:** `FactClubMatch`, `DimClub`, `DimMatch`. Measures: Goal Difference Total, Matches Played, Wins, Win Rate Pct, Total Points, Points per Match; rank at matchday (cumulative points then rank).

---

### 5.2 Offensive-Defensive page

**Reference:** `Dashboard Design/Ligue-Wide_View/Offensive-Defensive.md`

| Visual | Type | Description |
|--------|------|-------------|
| Attack–defence map (efficiency frontier) | **Scatter chart** | X = goals for per match, Y = goals against per match (inverted so high = better defence); size = points; one point per club. Optional quadrant divider (e.g. cross image). Profiles: Dominant, Fragile, Pragmatic, Weak. |

**Key tables/measures:** `FactClubMatch`, `DimClub`. Measures: Goals For, Goals Against, Matches Played, Goals For/Against per Match, Total Points. Colours: use report theme palette (no per-club colour mapping).

---

### 5.3 Rivalities (two pages)

**Reference:** `Dashboard Design/Ligue-Wide_View/Rivalities.md`

**Rivality = unordered pair of clubs** (all matches between the two). Table **RivalryPair** (calculated): ClubKey1, ClubKey2, PairLabel.

**Page 1 — Rivalities (overview: most anticipated)**

| Visual | Type | Description |
|--------|------|-------------|
| Table — list of PairLabel | **Table** (selector) | One row per rivalry; user selects a row to filter the report to that pair. |
| Average attendance by rivalry | **Stacked bar chart** | Y = PairLabel, value = average attendance for matches of that pair. |
| Average fill rate by rivalry | **Stacked bar chart** | Y = PairLabel, value = average fill rate (%) for matches of that pair. |

**Page 2 — Rivalities (2) (detail for one rivalry)**

| Visual | Type | Description |
|--------|------|-------------|
| HTML cards — head-to-head summary | **HTML Content** | Five cards: match count, wins club 1, wins club 2, draws, total goals (for the two selected clubs). |
| Sanctions per rivalry by reason | **Stacked bar chart** | Y = PairLabel, value = sanction count, **legend = Reason**. |
| Match history (chronological) | **Table / timeline** | List of matches between the two selected clubs: matchday, date, home/away, goals; filter by measure **Is Rivalry Match** = TRUE. |

**Key tables/measures:** `RivalryPair`, `DimClub`, `DimMatch`, `FactClubMatch`, `FactAttendance`, `FactSanction`. Measures: Avg Attendance in Rivalry, Avg Fill Rate in Rivalry, Rivalry Match Count, Rivalry Wins Club1/Club2, Rivalry Draws, Rivalry Goals Club1/Club2, Rivalry Head-to-Head Cards HTML, sanctions in rivalry; DimMatch: HomeClubName, AwayClubName, Goals Home, Goals Away, Is Rivalry Match.

---

### 5.4 Sanctions page

**Reference:** `Dashboard Design/Ligue-Wide_View/Sanctions.md`

**Purpose:** Explore factors influencing sanctions (e.g. matchday, position, home/away) and each club’s sanction behaviour.

| Visual | Type | Description |
|--------|------|-------------|
| Red cards by club | **Bar chart** | Red card count per club; **legend = Position** (DimPlayer). |
| Total sanctions: home vs away | **Bar chart** | Two bars: total sanctions at home vs away (all clubs). Sanctions attributed to last match of club on or before sanction date. |
| Cumulative sanctions by matchday | **Line chart** | X = Matchday (1–38), Y = cumulative sanction count (league-wide or per club). |
| Sanctions by club and reason | **Stacked bar chart** | Y = Club, value = sanction count, **legend = Reason**. |
| Distribution of sanctions by reason | **Donut chart** | One segment per reason; value = count. |

**Key tables/measures:** `FactSanction`, `DimClub`, `DimPlayer`, `DimMatch`, `DimDate`. Measures: Sanction Count (selected reasons optional), Cumulative Sanction Count, Sanctions Home, Sanctions Away; red-card count by club (filter by Reason or SanctionType).

---

## 6. Visual charter and themes

**Reference:** `Dashboard Design/6_Visual_Charter.md`

- **Club colours:** No per-club colour mapping is used. Charts and series use the default report theme palette (colours assigned in order by Power BI).
- **Themes:** `Dashboard Design/theme_arena_light.json` (and any dark variant) defines report theme (backgrounds, data colours, visuals). Apply in Power BI via **View → Themes**.
- **Club logos:** Where used (e.g. Club View, selectors), logo URLs can be stored in `DimClub` (e.g. `Club_URL`) or in `club_logos.json`; use in Image or HTML Content visuals with DAX or mapping.

---

## 7. Data quality and implementation notes

- **Data model:** The canonical definition is **Data_Model.bim**. Schema details (columns, calculated columns, measures) are in **Docs/0_markdown.md**.
- **Consistency:** Club names, player IDs, and date formats should be aligned across all source files and the model (ETL steps in Power Query).
- **Gaps:** Revenue per match, dedicated injury/availability data, and (if missing) player age/nationality or market value may be documented as N/A or proxy in the relevant Club View pages.
- **Implementation:** For each visual, use the referenced Dashboard Design document for DAX, field roles, filters, and step-by-step Power BI setup.

This specification is the single entry point for the dashboard structure; the **Dashboard Design** folder holds the detailed, implementable documentation for each view and page.
