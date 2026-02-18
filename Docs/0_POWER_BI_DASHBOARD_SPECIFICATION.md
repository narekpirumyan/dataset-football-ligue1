# Power BI Dashboard Specification — Ligue 1 Season 2023-2024

**Purpose:** This document defines the 5-page Power BI dashboard concept, maps each visual to source files and required data fields, and supports **data quality assessment** (gaps, consistency, completeness).

**Context:** Single-club or league-wide view using the dataset in the current folder (match results, player stats, transfers, stadium attendance, disciplinary sanctions). When provided: `players.xlsx` supplies player age, nationality, salary, and market value; `clubs.xlsx` supplies the club list and attributes (stadium, capacity) as the primary source for DimClub.

---

## Available Data Files (Summary)

| File | Format | Key content |
|------|--------|-------------|
| `match_results_2023_2024.csv` | CSV | match_id, matchday, match_date, home_team, away_team, home_score, away_score, stadium, referee, attendance |
| `player_stats_season.csv` | CSV | player_id, full_name, club, position, matches_played, starts, minutes_played, goals, assists, yellow_cards, red_cards, shots, shots_on_target, clean_sheets, saves, successful_dribbles, interceptions, successful_tackles |
| `transfers_2023_2024.txt` | TSV | transfer_id, player, player_id, departure_club, arrival_club, transfer_type, amount_ME, transfer_date, agent |
| `stadium_attendance.csv` | CSV (;) | club, matchday, date, opponent, attendance, stadium_capacity, fill_rate, weather, temperature_c |
| `disciplinary_sanctions.csv` | CSV | sanction_id, player, player_id, club, sanction_date, sanction_type, reason, suspension_matches, fine_euros |
| `players.xlsx` | Excel | player_id, age, nationality, salary, market_value (enriches DimPlayer for squad/financial visuals) |
| `clubs.xlsx` | Excel | club name, stadium, capacity (primary source for DimClub; single source of truth for club list and attributes) |
| `season_report_2023_2024.pdf` | PDF | Final standings, key season stats, narrative (reference only) |

---

# Page 1 — Season Performance  
## "What Did We Achieve?" (Sporting Outcomes)

**Message:** Did we perform competitively, and when did things change?

### 1.1 Final rank + total points (KPI cards)

| Visual | Description |
|--------|-------------|
| Final rank | League position at end of season (e.g. filter by club) |
| Total points | Sum of points (3/1/0) over all 38 matchdays |

**Source file:** `match_results_2023_2024.csv`  
**Data required:**

| Field | Use | Notes |
|-------|-----|--------|
| `home_team` | Identify club | Filter for "our" club |
| `away_team` | Identify club | Same |
| `home_score`, `away_score` | Result (W/D/L) | Derive points per match |
| `matchday` | Season scope | 1-38 |

**Alternative / validation:** `season_report_2023_2024.pdf` — final standings table (Pos, Club, Pts) for cross-check.

**Data quality checks:**  
- [ ] All 38 matchdays present per club  
- [ ] No duplicate match_id per club  
- [ ] Consistent club names across home_team / away_team  

---

### 1.2 Rank evolution by matchday (line chart)

| Visual | Description |
|--------|-------------|
| X-axis | Matchday (1-38) |
| Y-axis | League rank (1-20) |
| Line(s) | One line per club (e.g. focus club + comparison) |

**Source file:** `match_results_2023_2024.csv`  
**Data required:**

| Field | Use |
|-------|-----|
| `match_id`, `matchday`, `match_date` | Order and scope |
| `home_team`, `away_team`, `home_score`, `away_score` | Compute points per match, then cumulative points per club per matchday |
| — | **Derived:** cumulative points after each matchday → rank per matchday |

**Data quality checks:**  
- [ ] Matchdays sequential; no missing matchdays in date range  
- [ ] Date formats consistent (`match_date`) — file has mixed formats (2023-08-12, 19 Aug 2023, 18/08/2023) |

---

### 1.3 Win / Draw / Loss split

| Visual | Description |
|--------|-------------|
| Chart type | Pie or bar (count or % of matches) |
| Categories | Win, Draw, Loss (per club or overall) |

**Source file:** `match_results_2023_2024.csv`  
**Data required:**

| Field | Use |
|-------|-----|
| `home_team`, `away_team`, `home_score`, `away_score` | For each row, determine result for "our" club (W/D/L) |

**Data quality checks:**  
- [ ] No nulls in score columns  
- [ ] W + D + L = 38 per club |

---

### 1.4 Points accumulation curve

| Visual | Description |
|--------|-------------|
| X-axis | Matchday (or match order) |
| Y-axis | Cumulative points |
| Line(s) | One per club |

**Source file:** `match_results_2023_2024.csv`  
**Data required:** Same as 1.2 — `matchday`, `home_team`, `away_team`, `home_score`, `away_score` to compute points and running total.

**Data quality checks:** Same as 1.2.

---

### 1.5 Home vs Away comparison

| Visual | Description |
|--------|-------------|
| Metrics | Points, wins, goals for/against, or win rate |
| Split | Home vs Away (per club) |

**Source file:** `match_results_2023_2024.csv`  
**Data required:**

| Field | Use |
|-------|-----|
| `home_team`, `away_team` | Tag match as "home" or "away" for each club |
| `home_score`, `away_score` | Result and goals for/against |

**Data quality checks:**  
- [ ] Each club has 19 home and 19 away matches  
- [ ] No duplicate matches (each match_id once) |

---

### 1.6 Form timeline (last 5 matches trend)

| Visual | Description |
|--------|-------------|
| Display | Per matchday (or date), show result of "last 5" (e.g. W/W/D/L/W) or running form metric |
| Purpose | Momentum and streaks |

**Source file:** `match_results_2023_2024.csv`  
**Data required:**

| Field | Use |
|-------|-----|
| `matchday`, `match_date` | Order matches |
| `home_team`, `away_team`, `home_score`, `away_score` | Result (W/D/L) per match for the club |

**Data quality checks:**  
- [ ] Chronological order consistent (match_date vs matchday)  
- [ ] No gaps so "last 5" is well-defined |

---

### 1.7 Performance vs strong/weak opponents (optional)

| Visual | Description |
|--------|-------------|
| Idea | Points or goals vs "top 6" vs "bottom 6" (by final rank) |

**Source files:** `match_results_2023_2024.csv` + final standings (from same file or PDF).  
**Data required:** Match results + classification of opponent strength (e.g. from computed final table).  
**Data quality:** Depends on correct standings and club name alignment.

---

## Page 1 — File & field summary

| File | Fields used |
|------|-------------|
| `match_results_2023_2024.csv` | match_id, matchday, match_date, home_team, away_team, home_score, away_score, stadium, referee, attendance (attendance optional here) |
| `season_report_2023_2024.pdf` | Reference: final standings, aggregate stats |

**Known data quality issues (to analyse):**  
- Mixed date formats in `match_date`.  
- Some empty `attendance` values (e.g. row 66).  
- Need to confirm 380 matches and 20 clubs × 38 games.

---

# Page 2 — Squad Impact  
## "Who Delivered the Results?" (Human Performance)

**Message:** Which players drove success, and how balanced was the squad?

**Scope:** All registered players — **field players** (forwards, midfielders, defenders) and **goalkeepers**. The `position` field in player_stats (and DimPlayer[Position] in the model) distinguishes roles; visuals include both unless a metric is position-specific (e.g. goals leaders are typically field players; goalkeeper-specific metrics use clean_sheets, saves).

### 2.1 Goals + Assists leaders

| Visual | Description |
|--------|-------------|
| Chart | Bar chart or table: top N players by goals, and by assists (or combined) |
| Scope | Per club or league-wide; **all players** (field + goalkeepers). Goalkeepers will typically have 0 goals/assists; optional filter by position to show "Outfield only" if desired. |

**Source file:** `player_stats_season.csv`  
**Data required:**

| Field | Use |
|-------|-----|
| `player_id`, `full_name`, `club`, `position` | Identify player, filter by club; use `position` to include or exclude Goalkeeper |
| `goals` | Goals leaderboard |
| `assists` | Assists leaderboard |

**Data quality checks:**  
- [ ] Numeric type for goals/assists; no text in numeric fields  
- [ ] Some rows have trailing space in numbers (e.g. `8 ,2` in assists) — normalise |

---

### 2.2 Contribution % by top players

| Visual | Description |
|--------|-------------|
| Idea | % of team goals (or goals+assists) from top 5/10 players |
| Calculation | Sum(goals) top N / Sum(goals) team |
| Scope | **Field players** drive goals/assists; include all players for denominator (goalkeepers contribute 0), or restrict to outfield only for a cleaner "attacking contribution" view. |

**Source file:** `player_stats_season.csv`  
**Data required:** `club`, `goals`, `assists`, optionally `position` — aggregate by club, then by top players.

**Data quality checks:**  
- [ ] Total team goals consistent with match results (optional cross-check with match_results)  
- [ ] No negative goals/assists |

---

### 2.3 Minutes played distribution

| Visual | Description |
|--------|-------------|
| Chart | Distribution (histogram, box, or bar) of minutes_played per player (e.g. per club) |
| Purpose | Squad rotation and usage |
| Scope | All players (field + goalkeepers). Goalkeepers often have lower max minutes (e.g. backup); use `position` to split or filter if needed. |

**Source file:** `player_stats_season.csv`  
**Data required:**

| Field | Use |
|-------|-----|
| `minutes_played`, `club`, `player_id`, `position` | Distribution and filters; position to compare field vs goalkeeper usage |

**Data quality checks:**  
- [ ] `minutes_played` numeric (file has "289 min", "2754 min" — strip " min" or standardise)  
- [ ] No nulls where relevant; max minutes plausible (field ≤ 38×90 ≈ 3420; goalkeepers may be lower) |

---

### 2.4 Age / nationality structure

| Visual | Description |
|--------|-------------|
| Chart | Age distribution (e.g. histogram); nationality breakdown (pie/bar) |
| Purpose | Squad composition |
| Scope | **All players** (field + goalkeepers). Optional breakdown by `position` (e.g. age by role) or filter to outfield only. |

**Source file:** `players.xlsx` (loaded and merged into DimPlayer as Staging_Players).  
**Data required:** `player_id`, `age` (or date_of_birth), `nationality` (or country) per player. DimPlayer is enriched via Power Query merge so Age and Nationality are available for this visual.

**Data quality:** Ensure `player_id` in players.xlsx matches `player_stats_season` (trim, same format e.g. "J0001"); column names/types set in Staging_Players (Age Whole Number, Nationality Text). Data is **available** in `players.xlsx` (see Available Data Files).

---

### 2.5 Position depth chart

| Visual | Description |
|--------|-------------|
| Chart | Count or minutes by position (and optionally by club) |
| Purpose | Depth and balance by role |
| Scope | **All positions including Goalkeeper** and all field positions (e.g. Forward, Central midfielder, Defender, etc.). Ensures both field players and goalkeepers are represented in squad depth. |

**Source file:** `player_stats_season.csv`  
**Data required:**

| Field | Use |
|-------|-----|
| `position`, `club` | Group by position; filter by club. Must include "Goalkeeper" and all outfield positions. |
| `minutes_played` or `matches_played` | Depth (e.g. total minutes per position) |

**Data quality checks:**  
- [ ] Position values consistent (controlled vocabulary); typo check  
- [ ] All players have a position (field players and goalkeepers) |

---

### 2.6 Player value vs performance scatter

| Visual | Description |
|--------|-------------|
| Axes | X: "value" (salary or market value), Y: performance (goals, assists, or composite) |
| Purpose | Identify high-impact vs high-cost players |

**Source file:** `players.xlsx` (value) + `player_stats_season.csv` (performance).  
**Data required:** `salary` or `market_value`; for field players `goals`, `assists`; for goalkeepers `clean_sheets`, `saves`. Use `position` to distinguish. Scope: field players → goals/assists as performance; goalkeepers → clean_sheets/saves as performance (or separate visual/filter).

**Data quality:** **Available** in `players.xlsx` (salary, market_value) via Staging_Players → DimPlayer. Use `position` for field (goals/assists) vs GK (clean_sheets/saves). Transfer fee from `transfers_2023_2024.txt` can proxy value for transferred players if needed.

---

## Page 2 — File & field summary

| File | Fields used |
|------|-------------|
| `player_stats_season.csv` | player_id, full_name, club, position, matches_played, starts, minutes_played, goals, assists, (yellow_cards, red_cards, shots, shots_on_target, clean_sheets, saves, successful_dribbles, interceptions, successful_tackles as needed) |

**Scope (Page 2):** All players — **field players** and **goalkeepers**. The `position` column identifies role; use it to include both in squad visuals and to apply position-specific metrics (e.g. goals/assists for outfield, clean_sheets/saves for goalkeepers).

**Missing for full spec:** age, nationality, salary, market_value (from players.xlsx where used).

**Known data quality issues:**  
- `minutes_played`: mixed format ("1234" vs "1234 min").  
- Numeric fields: possible spaces (e.g. "8 ,2").  
- Empty cells in clean_sheets, saves for non-GKs.

---

# Page 3 — Financial Efficiency  
## "Did We Spend Smart?" (Business Performance)

**Message:** Did financial investment translate into performance?

### 3.1 Wage bill vs league position

| Visual | Description |
|--------|-------------|
| Chart | Scatter or bar: wage bill (total salary) per club vs final league position (or points) |

**Source file:** **Available.** Player salary from `players.xlsx` (DimPlayer via Staging_Players). Wage bill = sum of player salaries by club (DimPlayer + FactPlayerSeason). Position/points from match results (derived).  
**Data required:** DimPlayer[Salary], FactPlayerSeason (club link), match results for position/points.

**Data quality:** Ensure `players.xlsx` has Salary column; trim and type as Decimal. Nulls allowed.

---

### 3.2 Cost per point KPI

| Visual | Description |
|--------|-------------|
| Formula | Wage bill (or transfer spend) / total points |
| Scope | Per club |

**Source file:** Would use wage bill (missing) or transfer spend. Transfer spend can be **partially** derived from `transfers_2023_2024.txt` (see below).

**Data required:**  
- Points: from `match_results_2023_2024.csv` (derived).  
- Cost: salary (missing) or from transfers: `arrival_club`, `amount_ME`, filter by season window.

**Data quality:**  
- [ ] Transfer amounts: `amount_ME` has mixed formats (22.2, 56.9M€, 67.3 M, 74.2M€) — normalise to numeric.  
- [ ] Dates: filter 2023-24 season; some transfer_date are 2024-08 (next season) — define "season" window.

---

### 3.3 Salary vs contribution scatter

| Visual | Description |
|--------|-------------|
| Axes | X: salary, Y: goals+assists or other contribution metric |
| Scope | Per player |

**Source file:** **Available.** Player salary from `players.xlsx` (DimPlayer); goals/assists and minutes from `player_stats_season.csv` (FactPlayerSeason).  
**Data required:** DimPlayer[Salary], FactPlayerSeason[Goals], [Assists], [MinutesPlayed]; link via PlayerKey.

**Data quality:** Same as 2.6 and 3.1; ensure salary column in `players.xlsx` is loaded and typed.

---

### 3.4 Market value vs actual production

| Visual | Description |
|--------|-------------|
| Idea | Compare market value (or transfer fee) to output (goals, assists, minutes) |

**Source file:** **Available.** Market value (and salary) from `players.xlsx` (DimPlayer). **Partial alternative:** transfer fee for transferred players from `transfers_2023_2024.txt`. Performance from `player_stats_season.csv` (FactPlayerSeason).

**Data required:**  
- From transfers: `player_id`, `amount_ME` (as proxy for "value" for transferred players), `arrival_club`, `departure_club`, `transfer_date`.  
- From player_stats: `goals`, `assists`, `minutes_played`, `club`.

**Data quality:**  
- [ ] Normalise `amount_ME` (see 3.2).  
- [ ] Link by `player_id`; handle players with multiple transfers in period.  
- [ ] Free transfers (0) and loans: define how to treat in "value".

---

### 3.5 Most cost-efficient players

| Visual | Description |
|--------|-------------|
| Idea | Contribution per € (or per million €) — e.g. (goals+assists) / salary or / transfer_fee |

**Source file:** Same as 3.4. **Available:** salary and market value from `players.xlsx` (DimPlayer); contribution from FactPlayerSeason (goals, assists, or clean_sheets/saves for GK).

**Data quality:** Same as 3.2 and 3.4.

---

## Page 3 — File & field summary

| File | Fields used |
|------|-------------|
| `match_results_2023_2024.csv` | For points / position (derived) |
| `transfers_2023_2024.txt` | transfer_id, player, player_id, departure_club, arrival_club, transfer_type, amount_ME, transfer_date |
| `player_stats_season.csv` | club, goals, assists, minutes_played (for contribution) |
| `players.xlsx` | player_id, salary, market_value (DimPlayer) for wage bill, salary vs contribution, market value vs production, cost-efficient players |

**Available for Page 3:** Player salary and market value from `players.xlsx` (Staging_Players → DimPlayer); wage bill = sum of salaries by club. Transfer spend from transfers file for cost-per-point when salary not used.

**Known data quality issues:**  
- `amount_ME`: mixed units and formats (M€, M, decimals); need parsing and unit (e.g. M€).  
- `transfer_date`: mixed seasons; define 2023-24 window (e.g. 1 July 2023 - 30 June 2024 or 1 Sept - 31 Jan + summer).  
- Some transfers are from/to non-Ligue 1 clubs — filter by Ligue 1 club names if needed.

---

# Page 4 — Transfers, Availability & Risk  
## "What Influenced the Season?" (Change & Risk)

**Message:** What external factors shaped performance — recruitment or disruptions?

### 4.1 Transfers in/out summary

| Visual | Description |
|--------|-------------|
| Chart | Count and/or total fee: transfers IN vs OUT per club (and optionally by window) |

**Source file:** `transfers_2023_2024.txt`  
**Data required:**

| Field | Use |
|-------|-----|
| `departure_club`, `arrival_club` | Classify IN (arrival_club = our club) vs OUT (departure_club = our club) |
| `transfer_type`, `amount_ME` | Volume and spend |
| `transfer_date` | Filter by season; split summer vs winter if needed |

**Data quality checks:**  
- [ ] Club names align with match_results and player_stats (e.g. "Paris SG" vs "PSG")  
- [ ] Date parsing; consistent transfer_date format  
- [ ] amount_ME normalised for totals |

---

### 4.2 New signings contribution

| Visual | Description |
|--------|-------------|
| Idea | Goals/assists/minutes from players who "joined" during the season (or in summer 2023) |

**Source files:** `transfers_2023_2024.txt` + `player_stats_season.csv`  
**Data required:**  
- Transfers: `player_id`, `arrival_club`, `transfer_date` (to define "new signing").  
- Player stats: `player_id`, `club`, `goals`, `assists`, `minutes_played`.

**Data quality checks:**  
- [ ] Same player_id in both files  
- [ ] transfer_date within chosen window (e.g. summer 2023, or + winter 2024)  
- [ ] arrival_club = club in player_stats for post-transfer contribution |

---

### 4.3 Injury timeline vs results

| Visual | Description |
|--------|-------------|
| Idea | Timeline of "unavailable" players (injuries) and overlay with match results or points |

**Source file:** **No dedicated injury file.**  
**Data required:** Injury events (player, start/end date or matches missed) — **not in dataset.**  
**Workaround:** Use "matches missed" inferred from player_stats (e.g. 38 − matches_played for key players) as proxy for "availability"; no true injury timeline. Document as **GAP** or "Proxy only".

---

### 4.4 Matches missed by key players

| Visual | Description |
|--------|-------------|
| Idea | For key players (e.g. top by goals/assists), show matches_played vs 38 → "matches missed" |

**Source file:** `player_stats_season.csv`  
**Data required:**

| Field | Use |
|-------|-----|
| `matches_played`, `player_id`, `full_name`, `club` | Matches missed = 38 − matches_played (for outfield; adjust for late signings if needed) |

**Data quality checks:**  
- [ ] matches_played ≤ 38  
- [ ] Consider goalkeepers separately (often not 38)  
- [ ] Loan/transfer timing: player may have joined mid-season — 38 is max for full season |

---

### 4.5 Cards & impact on lost points

| Visual | Description |
|--------|-------------|
| Idea | Discipline: red cards / suspensions and correlation with results (e.g. points in matches with red, or after suspension) |

**Source files:**  
- `player_stats_season.csv`: `yellow_cards`, `red_cards` (totals).  
- `disciplinary_sanctions.csv`: `player_id`, `club`, `sanction_date`, `sanction_type`, `suspension_matches`, `reason`.

**Data required:**  
- From sanctions: suspension_matches, sanction_date, player_id, club.  
- From match results: points per match; link suspension to "matches missed" (no match-level "who was suspended" in data — use sanctions as proxy for "games out").  
- "Lost points" could be estimated (e.g. average points per match when key player suspended) — requires definition.

**Data quality checks:**  
- [ ] sanction_date format consistent  
- [ ] suspension_matches numeric; empty handled  
- [ ] fine_euros: some empty — optional |

---

## Page 4 — File & field summary

| File | Fields used |
|------|-------------|
| `transfers_2023_2024.txt` | transfer_id, player, player_id, departure_club, arrival_club, transfer_type, amount_ME, transfer_date, agent |
| `player_stats_season.csv` | player_id, full_name, club, matches_played, goals, assists, minutes_played, yellow_cards, red_cards |
| `disciplinary_sanctions.csv` | sanction_id, player, player_id, club, sanction_date, sanction_type, reason, suspension_matches, fine_euros |

**Missing for full spec:** Dedicated injury/availability dataset (dates and matches missed per injury).

**Known data quality issues:**  
- Transfers: amount_ME and date (see Page 3).  
- Sanctions: date format; suspension_matches and fine_euros can be blank.

---

# Page 5 — Fans, Stadium & Executive Summary  
## "What Does It Mean for the Club?" (Strategic Wrap-Up)

**Message:** What is the overall impact of this season — competitively, financially, and commercially?

### 5.1 Attendance trend

| Visual | Description |
|--------|-------------|
| Chart | Line or bar: attendance over time (matchday or date) — per club or total |

**Source file:** `stadium_attendance.csv` or `match_results_2023_2024.csv`  
**Data required:**  
- From `stadium_attendance.csv`: `club`, `matchday`, `date`, `attendance`.  
- From match_results: `attendance`, `matchday`, `match_date` (and home_team for club).  
**Note:** Two sources for attendance — align on one or cross-validate; date/delimiter in stadium_attendance is `;` and `date` may have mixed formats (as in match_results).

**Data quality checks:**  
- [ ] stadium_attendance: delimiter `;`; decimal in fill_rate may be `,` (e.g. 96,6)  
- [ ] date formats (e.g. 19 Aug 2023, 18/08/2023)  
- [ ] Consistency between stadium_attendance and match_results attendance where same match |

---

### 5.2 Revenue per match

| Visual | Description |
|--------|-------------|
| Idea | Revenue (€) per match — e.g. ticket revenue or total matchday revenue |

**Source file:** **NOT PRESENT.**  
**Data required:** Match-level or club-level revenue (ticket sales, commercial, etc.).  
**Data quality:** **GAP — No revenue data.** Proxy: attendance × average ticket price if such price data is added later; or omit visual.

---

### 5.3 Capacity utilization %

| Visual | Description |
|--------|-------------|
| Chart | fill_rate (or attendance/capacity) by matchday or by club |

**Source file:** `stadium_attendance.csv`  
**Data required:**

| Field | Use |
|-------|-----|
| `club`, `matchday`, `date` | Time and filter |
| `attendance`, `stadium_capacity`, `fill_rate` | Utilization (use fill_rate if already %; else compute attendance/capacity) |

**Data quality checks:**  
- [ ] fill_rate: European decimal (e.g. 96,6) → parse as 96.6  
- [ ] stadium_capacity constant per club; no zeros  
- [ ] fill_rate ≤ 100 (or >100 if over-capacity allowed) |

---

### 5.4 Squad value rank vs final rank

| Visual | Description |
|--------|-------------|
| Idea | Compare "squad value" rank (e.g. by transfer market value or wage bill) with final league position |

**Source file:** **Available** when `players.xlsx` is present: squad value = sum of `market_value` (or `salary`) by club from DimPlayer. **Alternative:** aggregate of transfer fees from `transfers_2023_2024.txt` by club as proxy. Final rank from match results or PDF.

**Data required:** DimPlayer[MarketValue] or [Salary] by club; or transfers `arrival_club`, `departure_club`, `amount_ME`; standings from match results or PDF.

**Data quality:** Same as Page 3. When using players.xlsx, ensure MarketValue/Salary columns are loaded and typed; nulls allowed.

---

### 5.5 Season summary panel (success drivers + risks)

| Visual | Description |
|--------|-------------|
| Content | KPIs and short narrative: key success drivers (e.g. home form, top scorers, recruitment) and risks (injuries, discipline, financial) |
| Data | Synthesised from all pages; can use calculated KPIs and tables from existing visuals |

**Source files:** All — no extra fields; use outputs from other pages and possibly text from `season_report_2023_2024.pdf` for narrative.

---

## Page 5 — File & field summary

| File | Fields used |
|------|-------------|
| `stadium_attendance.csv` | club, matchday, date, opponent, attendance, stadium_capacity, fill_rate (weather, temperature_c optional) |
| `match_results_2023_2024.csv` | attendance, matchday, match_date, home_team (for cross-check or primary attendance if preferred) |
| `transfers_2023_2024.txt` | arrival_club, departure_club, amount_ME (for value proxy) |
| `season_report_2023_2024.pdf` | Narrative / reference |

**Missing for full spec:** Revenue per match (or ticket price). Squad value and wage bill are provided by `players.xlsx` when available (see Gaps and data sources).

**Known data quality issues:**  
- stadium_attendance: delimiter `;`; fill_rate as 96,6; date format.  
- Match_results: attendance sometimes empty.

---

# Data Quality Summary for Analysis

## By file

| File | Delimiter / format | Critical issues to analyse |
|------|--------------------|----------------------------|
| match_results_2023_2024.csv | Comma | Date formats (YYYY-MM-DD, DD Mon YYYY, DD/MM/YYYY); empty attendance; 380 rows + header |
| player_stats_season.csv | Comma | minutes_played "1234" vs "1234 min"; spaces in numbers (e.g. "8 ,2"); missing clean_sheets/saves for non-GK |
| transfers_2023_2024.txt | Tab | amount_ME (22.2, 56.9M€, 67.3 M); transfer_date (season boundary); club name alignment |
| stadium_attendance.csv | Semicolon | fill_rate 96,6 (comma decimal); date format; alignment with match_results |
| disciplinary_sanctions.csv | Comma | sanction_date; empty suspension_matches and fine_euros |
| players.xlsx | Excel | player_id alignment with player_stats; Age (Whole Number), Nationality (Text), Salary and MarketValue (Decimal); trim; two sheets (field + goalkeepers) appended |
| clubs.xlsx | Excel | ClubName trim and alignment with names in match_results, player_stats, transfers, etc.; StadiumName, Capacity (Whole Number) |

## Gaps and data sources

**Provided by `players.xlsx` (when present in dataset):** Player age, nationality, salary, market value (Staging_Players → DimPlayer). Wage bill = sum of salaries by club.

| Need | Used in pages | Suggestion |
|------|----------------|------------|
| Player age | 2 (Squad) | From `players.xlsx` (Staging_Players → DimPlayer) |
| Player nationality | 2 (Squad) | Same |
| Player salary / club wage bill | 3 (Financial), 5 (value rank) | Provided by `players.xlsx` (DimPlayer.Salary); wage bill = sum of player salaries by club |
| Market value (player/squad) | 2, 3, 5 | From `players.xlsx` (DimPlayer.MarketValue) or use transfer fee as proxy where applicable |
| Injury timeline / availability | 4 (Risk) | New source or use "matches missed" from player_stats only |
| Revenue per match | 5 (Fans) | New source or omit / proxy |

## Cross-file checks

- **Club names:** Same spelling across match_results, player_stats, transfers, stadium_attendance, disciplinary_sanctions (e.g. "Paris SG", "Olympique Lyonnais").  
- **player_id:** Consistent between player_stats, transfers, disciplinary_sanctions.  
- **Match identity:** Matchday + home_team + away_team (or match_id) align between match_results and stadium_attendance if both used for attendance.

---

# Next steps

1. **Data quality report:** For each file, run completeness (nulls), format (dates, numbers), and consistency (club names, player_id) checks.  
2. **Data prep:** Normalise dates, amount_ME, minutes_played, fill_rate; choose single attendance source or document difference.  
3. **Gap handling:** Decide which visuals to build with existing data, which to mark "N/A", and which to add after new data (age, nationality, salary, revenue, injuries).  
4. **Power BI:** Build data model (club, player, match, transfer, sanction, attendance); implement calculated columns/measures for points, rank, form, contribution %, cost per point (where data exists).

This specification is ready to support a structured **data quality analysis** and subsequent dashboard implementation.
