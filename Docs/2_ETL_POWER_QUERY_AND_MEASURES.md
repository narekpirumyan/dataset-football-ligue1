# ETL: From Raw Data to Star Schema (Power Query) + DAX Measures

**Audience:** You know ETL in Python (pandas, read_csv, merge, transform). This guide maps those ideas to **Power Query (M)** and **Power BI**, then adds **DAX measures**.

**References:**  
- [0_POWER_BI_DASHBOARD_SPECIFICATION.md](./0_POWER_BI_DASHBOARD_SPECIFICATION.md) — data quality issues by file  
- [1_POWER_BI_STAR_SCHEMA_DESIGN.md](./1_POWER_BI_STAR_SCHEMA_DESIGN.md) — target schema and relationships  

---

## Part A — Power Query (M) for Python users

### A.1 Concepts mapping

| Python / pandas | Power Query (M) / Power BI |
|-----------------|----------------------------|
| `pd.read_csv("file.csv")` | **Get Data** → Text/CSV (or Excel, Folder) → creates a **Query** |
| Pipeline of steps | **Applied Steps** (right pane): each step is a transformation; order matters |
| `df["col"] = df["col"].str.strip()` | **Transform** → Format → Trim (or **Add Column** → **Custom Column** with `Text.Trim([col])`) |
| `df.merge(other, on="key")` | **Merge Queries** (Home tab): pick left table, right table, join keys → expands with columns from right table |
| `df.drop_duplicates(subset=["col"])` | **Remove Rows** → Remove Duplicates (select columns first) |
| `df.assign(new = ...)` | **Add Column** → Custom Column → write M expression |
| Saving intermediate result | A **Query** is the “saved” result; other queries can **reference** it (e.g. `= DimClub`) |
| Running the pipeline | **Close & Apply** (loads to model) or **Refresh** (re-runs all queries) |

### A.2 Where to write M code

- **Custom Column:** `Add Column` → **Custom Column** → formula box (e.g. `Text.Trim([match_date])`).
- **Advanced Editor:** **Home** → **Advanced Editor** — edits the full M script for the current query. Use this to paste multi-line logic (e.g. date parsing, amount parsing).

### A.3 Execution order and dependencies

- Queries run in **dependency order**: if Query B references Query A, A runs first.
- Build **dimensions first** (DimClub, DimPosition, DimPlayer, DimDate), then **facts** that reference them via Merge.
- **Staging (raw) queries** that only clean data (no merge) can be created first; then dimension/fact queries reference staging and merge to dimensions.

### A.4 Data types in Power Query

- Set types explicitly: **Transform** → **Data Type** (Whole Number, Decimal, Date, Text).
- If a column has mixed formats, fix in a **Custom Column** (e.g. parse text to number) then change type.

---

## Part B — Data quality fixes (by source)

These issues come from [0_POWER_BI_DASHBOARD_SPECIFICATION.md](./0_POWER_BI_DASHBOARD_SPECIFICATION.md). Apply the fixes in **staging** queries so all downstream tables use clean data.

### B.1 match_results_2023_2024.csv

| Issue | Fix in Power Query |
|-------|--------------------|
| **Mixed date formats** in `match_date`: `2023-08-12`, `19 Aug 2023`, `18/08/2023` | Parse to a single Date type. Option 1: try **Date.FromText**; if it fails on some rows, use a **Custom Column** that tries multiple formats, e.g. `try Date.FromText([match_date]) otherwise try Date.From(Date.FromText(Text.Replace([match_date]," ","-"))) otherwise null`. Safer: **Advanced Editor** — add a function or conditional: if the column already parses as date, use it; else try `Date.FromText` with replaced separators, or split "19 Aug 2023" and use Date.From. See **Section C.2** for a reusable date parse. |
| **Empty attendance** (e.g. some rows have blank) | Leave as null or replace with 0; type **Whole Number**. For stadium visuals we use stadium_attendance as primary; match_results attendance is optional. |
| **380 matches + header** | No fix needed; ensure no duplicate `match_id` (Remove Duplicates on `match_id` if needed). |

**Recommended staging query name:** `Staging_MatchResults`

---

### B.2 player_stats_season.csv

| Issue | Fix in Power Query |
|-------|--------------------|
| **minutes_played** mixed: `1572` vs `289 min`, `2754 min` | **Custom Column**: extract number only. Example: `Number.FromText(Text.Replace(Text.Trim([minutes_played]), " min", ""))` — if some values are already number, use `if [minutes_played] is number then [minutes_played] else Number.FromText(Text.Replace(Text.Trim(Text.From([minutes_played])), " min", ""))`. Then set data type **Whole Number**. |
| **Spaces in numbers** (e.g. assists `8 ,2`) | **Transform** → Format → **Trim** on text columns; for numeric columns that are currently text: **Custom Column** `Number.FromText(Text.Replace(Text.Trim(Text.From([assists])), " ", ""))` (repeat for goals, and any other numeric fields that may have spaces). Then set type to **Whole Number** or **Decimal**. |
| **Empty clean_sheets, saves** for non-GK | Leave as null; type **Whole Number** (or Decimal). Do not fill with 0 if it means “not applicable”. |
| **Position** consistency | **Trim** position; optionally **Replace Values** for known typos (e.g. "Left Back" → "Left back") so DimPosition has consistent names. |

**Recommended staging query name:** `Staging_PlayerStats`

---

### B.3 transfers_2023_2024.txt

| Issue | Fix in Power Query |
|-------|--------------------|
| **Tab delimiter** | When loading: **Get Data** → Text/CSV → select file → set **Delimiter** to **Tab**. |
| **amount_ME** mixed: `22.2`, `56.9M€`, `67.3 M`, `74.2M€`, `0` | **Custom Column**: (1) Text.From([amount_ME]); (2) remove "M€", " M", "M", spaces; (3) Number.FromText. If result is null or invalid, use 0. Example: `let t = Text.Trim(Text.Replace(Text.Replace(Text.Replace(Text.From([amount_ME]), "M€", ""), " M", ""), " ", "")) in if t = "" or t = null then 0 else Number.FromText(t)` (adjust for your exact variants). Then set type **Decimal**. |
| **transfer_date** format and **season boundary** | Normalise to **Date** (same approach as match_date). Filter to season 2023–24 in a later step if needed (e.g. 2023-07-01 to 2024-06-30); optional. |
| **Club names** (departure_club, arrival_club) | Only Ligue 1 clubs exist in DimClub; external clubs (e.g. "AC Milan", "FC Barcelona") will not find a match in Merge. Option: add an **"External"** row to DimClub and map non-Ligue 1 clubs to it in a Custom Column before Merge, or leave as null and use **Merge with null** handling. |

**Recommended staging query name:** `Staging_Transfers`

---

### B.4 stadium_attendance.csv

| Issue | Fix in Power Query |
|-------|--------------------|
| **Semicolon delimiter** | **Get Data** → Text/CSV → set **Delimiter** to **Semicolon**. |
| **fill_rate** European decimal: `96,6` | **Custom Column**: `Number.FromText(Text.Replace(Text.From([fill_rate]), ",", "."))` then set type **Decimal** (or Percentage if you prefer). |
| **date** format | Same as match_date: normalise to **Date**. |
| **Consistency with match_results** | Use stadium_attendance as the single source for attendance/capacity/fill_rate; no need to merge match_results attendance into this table. |

**Recommended staging query name:** `Staging_StadiumAttendance`

---

### B.5 disciplinary_sanctions.csv

| Issue | Fix in Power Query |
|-------|--------------------|
| **sanction_date** | Normalise to **Date** (same as above). |
| **Empty suspension_matches, fine_euros** | Replace null with 0 for **SuspensionMatches** (Whole Number); leave **FineEuros** as null or 0, type **Decimal**/Currency. |

**Recommended staging query name:** `Staging_Sanctions`

---

### B.6 Cross-file consistency

- **Club names:** Build DimClub from **distinct** names collected from all staging queries; use **Trim** and **Replace Values** once in DimClub so spelling is unique (e.g. "Paris SG" everywhere). All facts then resolve club via Merge to DimClub, so no need to change club names in each source.
- **player_id:** Use as natural key; ensure Trim so "J0001" does not become " J0001 ". Staging_PlayerStats, Staging_Transfers, Staging_Sanctions should all use the same player_id list when building DimPlayer.

---

## Part C — Reusable M helpers (optional)

You can put these in a **blank query** and reference them from other queries (e.g. `fnParseDate`, `fnParseAmountME`). **Home** → **New Source** → **Blank Query** → **Advanced Editor** paste, then rename query to e.g. `Helpers`.

### C.1 Helper: parse multiple date formats

```powerquery
// Helper: fnParseDate
// Usage: Add Column -> Invoke Custom Function -> fnParseDate, column = [match_date]
(txt as text) as nullable date =>
  let
    t = Text.Trim(txt),
    try1 = try Date.FromText(t) otherwise null,
    try2 = if try1 = null then try Date.FromText(Text.Replace(Text.Replace(t, " ", "-"), "/", "-")) otherwise null else try1,
    try3 = if try2 = null then try #date(Number.FromText(Text.End(t, 4)), 
          Date.Month(Date.FromText("1 " & Text.Start(Text.Middle(t, 3, 3), 3) & " 2020")), 
          Number.FromText(Text.Start(t, 2))) otherwise null else try2
  in try3
```

Alternatively, use **Transform** → **Date** → **Using Locale** with a locale that matches one format, then fix remaining nulls with a second Custom Column that tries another format. Simplest robust approach: one **Custom Column** that uses `Date.FromText` after **Replace**: replace `"/"` with `"-"`, and for "19 Aug 2023" use a small lookup for month name to number then `#date(Year, Month, Day)`.

### C.2 Helper: parse amount_ME (transfers)

```powerquery
// Helper: fnParseAmountME
// Usage: Add Column -> Invoke fnParseAmountME on [amount_ME]
(raw as any) as number =>
  let
    t = Text.Trim(Text.Replace(Text.Replace(Text.Replace(Text.From(raw), "M€", ""), " M", ""), " ", "")),
    num = try Number.FromText(t) otherwise 0
  in if t = "" or t = null then 0 else num
```

---

## Part D — Build order and steps (Power Query)

Follow this order so that every query that does a **Merge** has the referenced query already built.

### D.1 Staging queries (data quality only)

Create these first; no Merge to dimensions.

1. **Staging_MatchResults**  
   - Source: `match_results_2023_2024.csv`.  
   - Steps: Promote headers; trim text columns; fix **match_date** (Custom Column or helper) → type Date; set **matchday** Whole Number, **home_score** / **away_score** Whole Number; **attendance** Whole Number (nulls allowed); remove duplicates on **match_id** if any.  
   - Keep columns: match_id, matchday, match_date, home_team, away_team, home_score, away_score, stadium, referee, attendance.

2. **Staging_PlayerStats**  
   - Source: `player_stats_season.csv`.  
   - Steps: Promote headers; trim all text columns; fix **minutes_played** (strip " min", number); fix **goals**, **assists**, **matches_played**, **starts**, **yellow_cards**, **red_cards** (trim spaces, then Number.FromText); set correct types; trim **position**.  
   - Keep all columns needed for FactPlayerSeason and for DimPlayer/DimPosition source.

3. **Staging_Transfers**  
   - Source: `transfers_2023_2024.txt` (Tab).  
   - Steps: Trim text; parse **amount_ME** (helper or Custom Column); parse **transfer_date** → Date; keep transfer_id, player, player_id, departure_club, arrival_club, transfer_type, amount_ME (cleaned), transfer_date (as date).

4. **Staging_StadiumAttendance**  
   - Source: `stadium_attendance.csv` (Semicolon).  
   - Steps: Trim; fix **fill_rate** (comma → dot, Decimal); fix **date** → Date; set **attendance**, **stadium_capacity** Whole Number.

5. **Staging_Sanctions**  
   - Source: `disciplinary_sanctions.csv`.  
   - Steps: Trim; **sanction_date** → Date; **suspension_matches** → Whole Number (null → 0); **fine_euros** → Decimal (null allowed).

---

### D.2 DimClub

- **Source:** Combine distinct club names from all staging tables.  
- **Steps:**
  1. Create a list of tables: `{Staging_MatchResults[home_team], Staging_MatchResults[away_team], Staging_PlayerStats[club], Staging_Transfers[departure_club], Staging_Transfers[arrival_club], Staging_StadiumAttendance[club], Staging_Sanctions[club]}` — in M you do this by **Append** of tables. Easiest: create one table per source with one column "ClubName", then **Append Queries** (Append as New) to get one long table of names.
  2. **Remove Duplicates** on ClubName.
  3. **Trim** ClubName.
  4. **Add Index Column** (1, 1) → rename to **ClubKey**.
  5. Optionally: merge with a grouped stadium_attendance (Group By club → Max capacity, First stadium) to add StadiumName and Capacity; or leave those columns for later.
- **Output columns:** ClubKey (Whole Number), ClubName (Text), optionally StadiumName, Capacity.

**M sketch (one way to get distinct clubs):**

```powerquery
// DimClub: combine clubs from all sources
let
  Home = Table.SelectColumns(Staging_MatchResults, {"home_team"}),
  RenameHome = Table.RenameColumns(Home, {{"home_team", "ClubName"}}),
  Away = Table.SelectColumns(Staging_MatchResults, {"away_team"}),
  RenameAway = Table.RenameColumns(Away, {{"away_team", "ClubName"}}),
  ClubsFromMatches = Table.Distinct(Table.Combine({RenameHome, RenameAway})),
  ClubsFromPlayers = Table.RenameColumns(Table.SelectColumns(Staging_PlayerStats, {"club"}), {{"club", "ClubName"}}),
  ClubsFromTransfersDep = Table.RenameColumns(Table.SelectColumns(Staging_Transfers, {"departure_club"}), {{"departure_club", "ClubName"}}),
  ClubsFromTransfersArr = Table.RenameColumns(Table.SelectColumns(Staging_Transfers, {"arrival_club"}), {{"arrival_club", "ClubName"}}),
  ClubsFromStadium = Table.RenameColumns(Table.SelectColumns(Staging_StadiumAttendance, {"club"}), {{"club", "ClubName"}}),
  ClubsFromSanctions = Table.RenameColumns(Table.SelectColumns(Staging_Sanctions, {"club"}), {{"club", "ClubName"}}),
  Combined = Table.Distinct(Table.Combine({ClubsFromMatches, ClubsFromPlayers, ClubsFromTransfersDep, ClubsFromTransfersArr, ClubsFromStadium, ClubsFromSanctions})),
  Trimmed = Table.TransformColumns(Combined, {{"ClubName", Text.Trim}}),
  DistinctClubs = Table.Distinct(Trimmed),
  AddKey = Table.AddIndexColumn(DistinctClubs, "ClubKey", 1, 1)
in AddKey
```

---

### D.3 DimPosition

- **Source:** `Staging_PlayerStats`.  
- **Steps:** Select **position** → **Remove Duplicates** → **Trim** → **Add Index Column** (1, 1) → rename to **PositionKey**; rename **position** to **PositionName**.  
- **Output:** PositionKey, PositionName.

---

### D.4 DimPlayer

- **Source:** `Staging_PlayerStats` (distinct player_id, full_name).  
- **Steps:** Select **player_id**, **full_name** → **Remove Duplicates** (on player_id) → **Trim** → **Add Index Column** (1, 1) → **PositionKey**; rename to **PlayerKey**. Optionally: **Merge** with `players.xlsx` (if present) on player_id to add Age, Nationality, Salary, MarketValue.  
- **Output:** PlayerKey, PlayerId, FullName, (Age, Nationality, Salary, MarketValue if available).

---

### D.5 DimDate

- **Source:** Calendar (no file).  
- **Steps:** **New Source** → **Blank Query** → **Advanced Editor**:

```powerquery
let
  StartDate = #date(2023, 7, 1),
  EndDate = #date(2024, 8, 31),
  Count = Duration.Days(EndDate - StartDate) + 1,
  Dates = List.Dates(StartDate, Count, #duration(1, 0, 0, 0)),
  ToTable = Table.FromList(Dates, Splitter.SplitByNothing(), {"DateKey"}, null, ExtraValues.Error),
  AddDate = Table.AddColumn(ToTable, "Date", each [DateKey], type date),
  AddYear = Table.AddColumn(AddDate, "Year", each Date.Year([DateKey]), Int64.Type),
  AddMonth = Table.AddColumn(AddYear, "Month", each Date.Month([DateKey]), Int64.Type),
  AddDay = Table.AddColumn(AddMonth, "Day", each Date.Day([DateKey]), Int64.Type)
in AddDay
```

- Set **DateKey** and **Date** to **Date** type.  
- **Output:** DateKey (PK for relationships), Date, Year, Month, Day.

---

### D.6 DimMatch

- **Source:** `Staging_MatchResults`.  
- **Steps:**
  1. **Merge** with DimClub: left = Staging_MatchResults[home_team], right = DimClub[ClubName] → expand only **ClubKey** → rename to **HomeClubKey**.
  2. **Merge** with DimClub again: left = Staging_MatchResults[away_team], right = DimClub[ClubName] → expand only **ClubKey** → rename to **AwayClubKey**.
  3. Select columns: **match_id** (rename to **MatchKey**), **matchday** (Matchday), **match_date** (MatchDate), **HomeClubKey**, **AwayClubKey**, **stadium** (Stadium), **referee** (Referee).
  4. Set types: MatchKey Text, Matchday Whole Number, MatchDate Date, HomeClubKey/AwayClubKey Whole Number.
- **Output:** MatchKey, Matchday, MatchDate, HomeClubKey, AwayClubKey, Stadium, Referee.

---

### D.7 FactClubMatch

- **Source:** `Staging_MatchResults`.  
- **Grain:** Two rows per match (one home club, one away club).  
- **Steps:**
  1. **Merge** with DimClub twice to get HomeClubKey and AwayClubKey (as in DimMatch).
  2. **Add Custom Column** — Home rows: e.g. `Table.FromRows` or duplicate and filter. Easier: **Add Custom Column** that returns a **table** with two rows per row (home and away), then **Expand**. Simpler approach without expanding:
     - **Duplicate** the query (or reference Staging_MatchResults twice). In first copy: keep only columns needed for **home** side: MatchKey = match_id, ClubKey = HomeClubKey, IsHome = true, Points = if home_score > away_score then 3 else if home_score = away_score then 1 else 0, GoalsFor = home_score, GoalsAgainst = away_score, Result = if home_score > away_score then "W" else if home_score = away_score then "D" else "L".
     - In second copy: same but for **away** side: ClubKey = AwayClubKey, IsHome = false, Points = if away_score > home_score then 3 else ..., GoalsFor = away_score, GoalsAgainst = home_score, Result = ...
  3. **Append** the two tables (home and away) into one.  
  4. Select final columns: MatchKey, ClubKey, IsHome, Points, GoalsFor, GoalsAgainst, Result. Set types.
- **Output:** MatchKey, ClubKey, IsHome, Points, GoalsFor, GoalsAgainst, Result.

**M sketch (single query that builds both sides):**

```powerquery
// FactClubMatch: need HomeClubKey, AwayClubKey from merge first
let
  Merged = Table.NestedJoin(Staging_MatchResults, {"home_team"}, DimClub, {"ClubName"}, "Home", JoinKind.LeftOuter),
  ExpandedHome = Table.ExpandTableColumn(Merged, "Home", {"ClubKey"}, {"HomeClubKey"}),
  Merged2 = Table.NestedJoin(ExpandedHome, {"away_team"}, DimClub, {"ClubName"}, "Away", JoinKind.LeftOuter),
  ExpandedAway = Table.ExpandTableColumn(Merged2, "Away", {"ClubKey"}, {"AwayClubKey"}),
  HomeRows = Table.AddColumn(ExpandedAway, "IsHome", each true),
  HomeRows2 = Table.SelectColumns(HomeRows, {"match_id", "HomeClubKey", "IsHome", "home_score", "away_score"}),
  HomePoints = Table.AddColumn(HomeRows2, "Points", each if [home_score] > [away_score] then 3 else if [home_score] = [away_score] then 1 else 0),
  HomeGFGA = Table.AddColumn(Table.AddColumn(HomePoints, "GoalsFor", each [home_score]), "GoalsAgainst", each [away_score]),
  HomeResult = Table.AddColumn(HomeGFGA, "Result", each if [home_score] > [away_score] then "W" else if [home_score] = [away_score] then "D" else "L"),
  HomeFinal = Table.SelectColumns(Table.RenameColumns(HomeResult, {{"match_id", "MatchKey"}, {"HomeClubKey", "ClubKey"}}), {"MatchKey", "ClubKey", "IsHome", "Points", "GoalsFor", "GoalsAgainst", "Result"}),
  AwayRows = Table.AddColumn(ExpandedAway, "IsHome", each false),
  AwayRows2 = Table.SelectColumns(AwayRows, {"match_id", "AwayClubKey", "IsHome", "home_score", "away_score"}),
  AwayPoints = Table.AddColumn(AwayRows2, "Points", each if [away_score] > [home_score] then 3 else if [away_score] = [home_score] then 1 else 0),
  AwayGFGA = Table.AddColumn(Table.AddColumn(AwayPoints, "GoalsFor", each [away_score]), "GoalsAgainst", each [home_score]),
  AwayResult = Table.AddColumn(AwayGFGA, "Result", each if [away_score] > [home_score] then "W" else if [away_score] = [home_score] then "D" else "L"),
  AwayFinal = Table.SelectColumns(Table.RenameColumns(AwayResult, {{"match_id", "MatchKey"}, {"AwayClubKey", "ClubKey"}}), {"MatchKey", "ClubKey", "IsHome", "Points", "GoalsFor", "GoalsAgainst", "Result"}),
  Combined = Table.Combine({HomeFinal, AwayFinal})
in Combined
```

---

### D.8 FactPlayerSeason

- **Source:** `Staging_PlayerStats`.  
- **Steps:**
  1. **Merge** DimClub on club → ClubKey; DimPosition on position → PositionKey; DimPlayer on player_id → PlayerKey.
  2. **Add Column** MatchesMissed = 38 - [matches_played] (or blank for goalkeepers if you prefer).
  3. Select: PlayerKey, ClubKey, PositionKey, Goals, Assists, MinutesPlayed, MatchesPlayed, MatchesMissed, Starts, YellowCards, RedCards (and optional stats).
  4. Set all types (keys Whole Number, measures as needed).
- **Output:** As per schema.

---

### D.9 FactTransfer

- **Source:** `Staging_Transfers`.  
- **Steps:**
  1. **Merge** DimPlayer on player_id → PlayerKey.
  2. **Merge** DimClub on departure_club → DepartureClubKey (clubs not in DimClub: use **null** or add "External" to DimClub and map there).
  3. **Merge** DimClub on arrival_club → ArrivalClubKey.
  4. Add **TransferDateKey**: same as transfer_date (date) for relationship to DimDate — ensure this date exists in DimDate (your calendar 2023-07-01 to 2024-08-31 covers it).
  5. Select: TransferKey (transfer_id), PlayerKey, DepartureClubKey, ArrivalClubKey, TransferDateKey, AmountME, TransferType.
- **Output:** As per schema.

---

### D.10 FactSanction

- **Source:** `Staging_Sanctions`.  
- **Steps:** **Merge** DimPlayer (player_id → PlayerKey), DimClub (club → ClubKey); add **SanctionDateKey** = sanction_date (must be in DimDate). Select: SanctionKey, PlayerKey, ClubKey, SanctionDateKey, SanctionType, SuspensionMatches, FineEuros (optional Reason). Set types.
- **Output:** As per schema.

---

### D.11 FactAttendance

- **Source:** `Staging_StadiumAttendance`.  
- **Steps:** **Merge** DimClub (club → ClubKey); add **DateKey** = date column (for DimDate). Select: ClubKey, Matchday, DateKey, Attendance, StadiumCapacity, FillRatePct (optional: Weather, TemperatureC). Optionally: **Merge** with DimMatch or Staging_MatchResults on (home_team = club, matchday, match_date ≈ date) to get MatchKey — only if you want to link attendance to matches.
- **Output:** As per schema.

---

### D.12 Optional: StandingsByMatchday

- **Source:** FactClubMatch + DimMatch.  
- In Power Query: **Reference** FactClubMatch; **Merge** DimMatch on MatchKey to get Matchday; **Group By** ClubKey and Matchday, **Sum** Points; then add **Cumulative Points** (running sum per club over matchdays — requires sort and running sum in M, or do this in DAX as a calculated table).  
- Alternatively: build **StandingsByMatchday** as a **DAX calculated table** (see Part F).

---

## Part E — Relationships in Power BI

After **Close & Apply**, in **Model** view:

1. **FactClubMatch[MatchKey]** → **DimMatch[MatchKey]** (Many to One, Single filter).
2. **FactClubMatch[ClubKey]** → **DimClub[ClubKey]** (Many to One, Single).
3. **FactPlayerSeason[PlayerKey]** → **DimPlayer[PlayerKey]** (Many to One, Single).
4. **FactPlayerSeason[ClubKey]** → **DimClub[ClubKey]** (Many to One, Single).
5. **FactPlayerSeason[PositionKey]** → **DimPosition[PositionKey]** (Many to One, Single).
6. **FactTransfer[PlayerKey]** → **DimPlayer[PlayerKey]** (Many to One, Single).
7. **FactTransfer[DepartureClubKey]** → **DimClub[ClubKey]** (Many to One, Single).
8. **FactTransfer[ArrivalClubKey]** → **DimClub[ClubKey]** (Many to One, Single).
9. **FactTransfer[TransferDateKey]** → **DimDate[DateKey]** (Many to One, Single).
10. **FactSanction[PlayerKey]** → **DimPlayer[PlayerKey]** (Many to One, Single).
11. **FactSanction[ClubKey]** → **DimClub[ClubKey]** (Many to One, Single).
12. **FactSanction[SanctionDateKey]** → **DimDate[DateKey]** (Many to One, Single).
13. **FactAttendance[ClubKey]** → **DimClub[ClubKey]** (Many to One, Single).
14. **FactAttendance[DateKey]** → **DimDate[DateKey]** (Many to One, Single).

Drag from the fact column to the dimension column to create each relationship; set **Cardinality** to Many to One and **Cross filter direction** to Single (from dimension to fact).

**FactTransfer and DimClub:** FactTransfer has two foreign keys to DimClub (ArrivalClubKey, DepartureClubKey). Create both relationships, then set **one** as active (e.g. ArrivalClubKey) and the other as **inactive** (Model view → select the relationship → **Properties** → uncheck "Make this relationship active"). Use **USERELATIONSHIP** in the "Revenue out" measure to use the inactive relationship (see F.3).

---

## Part F — DAX measures

Create a **Measure table**: **Modeling** → **New Table** → e.g. `Measures = ROW("_", BLANK())` then hide the column "_" and put all measures in this table. Or create measures in the fact table they refer to.

### F.1 Page 1 — Season performance

```dax
Total Points = SUM(FactClubMatch[Points])

Total Goals For = SUM(FactClubMatch[GoalsFor])

Total Goals Against = SUM(FactClubMatch[GoalsAgainst])

Matches Played = COUNTROWS(FactClubMatch)

Wins = CALCULATE(COUNTROWS(FactClubMatch), FactClubMatch[Result] = "W")
Draws = CALCULATE(COUNTROWS(FactClubMatch), FactClubMatch[Result] = "D")
Losses = CALCULATE(COUNTROWS(FactClubMatch), FactClubMatch[Result] = "L")

Home Points = CALCULATE(SUM(FactClubMatch[Points]), FactClubMatch[IsHome] = TRUE())
Away Points = CALCULATE(SUM(FactClubMatch[Points]), FactClubMatch[IsHome] = FALSE())
```

**Cumulative Points (for rank evolution):** Use with **DimMatch[Matchday]** on axis and **DimClub** in legend. Filter context must be one club (or all) and one matchday; we sum points for that club for all matchdays ≤ current.

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

**League Rank (at current matchday):** Use in a visual with **DimMatch[Matchday]** on X and **DimClub[ClubName]** in legend; measure on Y. Rank is per matchday.

```dax
League Rank =
RANKX(
  ALL(DimClub[ClubName]),
  [Cumulative Points],
  [Cumulative Points],
  DESC,
  Dense
)
```

**Form (last 5 results as text):** Optional; can be complex. Simpler option: use a **calculated table** or **Power Query** that pre-computes "FormLast5" per club per matchday (e.g. concatenation of Result for last 5 matches). Then use that column in a card or table. Alternatively, a measure that returns the last 5 results for the selected club (requires context of one club and one matchday):

```dax
Form Last 5 =
VAR _Club = SELECTEDVALUE(DimClub[ClubKey])
VAR _Matchday = SELECTEDVALUE(DimMatch[Matchday])
VAR _Last5 = TOPN(5, FILTER(ALL(FactClubMatch, DimMatch), FactClubMatch[ClubKey] = _Club && DimMatch[Matchday] <= _Matchday), DimMatch[Matchday], DESC)
RETURN CONCATENATEX(_Last5, FactClubMatch[Result], "")
```

---

### F.2 Page 2 — Squad impact

```dax
Total Goals = SUM(FactPlayerSeason[Goals])
Total Assists = SUM(FactPlayerSeason[Assists])
Goals Plus Assists = [Total Goals] + [Total Assists]

Team Goals = SUM(FactPlayerSeason[Goals])   // in club filter context

Contribution Pct = DIVIDE([Total Goals], [Team Goals], BLANK())   // use for one player in club context

Total Minutes Played = SUM(FactPlayerSeason[MinutesPlayed])
Matches Missed = SUM(FactPlayerSeason[MatchesMissed])
```

---

### F.3 Page 3 — Financial efficiency

Use **one active** relationship from FactTransfer to DimClub (e.g. on **ArrivalClubKey**) and one **inactive** (on **DepartureClubKey**). Then:

```dax
Transfer Spend In = SUM(FactTransfer[AmountME])
// When you filter by club (slicer on DimClub), the active relationship filters FactTransfer by ArrivalClubKey = selected club.

Transfer Revenue Out = CALCULATE(SUM(FactTransfer[AmountME]), USERELATIONSHIP(FactTransfer[DepartureClubKey], DimClub[ClubKey]))
// USERELATIONSHIP switches to the inactive relationship so the club filter applies to DepartureClubKey (transfers out).

Net Transfer Spend = [Transfer Spend In] - [Transfer Revenue Out]

Cost per Point = DIVIDE([Transfer Spend In], [Total Points], BLANK())
```

If you prefer not to use an inactive relationship, use explicit filters:

```dax
Transfer Spend In = CALCULATE(SUM(FactTransfer[AmountME]), FactTransfer[ArrivalClubKey] IN VALUES(DimClub[ClubKey]))
Transfer Revenue Out = CALCULATE(SUM(FactTransfer[AmountME]), FactTransfer[DepartureClubKey] IN VALUES(DimClub[ClubKey]))
```

---

### F.4 Page 4 — Transfers, availability & risk

```dax
Transfers In Count = CALCULATE(COUNTROWS(FactTransfer), FactTransfer[ArrivalClubKey] IN VALUES(DimClub[ClubKey]))
Transfers Out Count = CALCULATE(COUNTROWS(FactTransfer), FactTransfer[DepartureClubKey] IN VALUES(DimClub[ClubKey]))

Total Suspension Matches = SUM(FactSanction[SuspensionMatches])
Total Fines = SUM(FactSanction[FineEuros])
```

**New Signings Contribution:** Sum of goals/assists for players who transferred IN to the selected club in the season. Easiest: add a calculated column **IsNewSigning** on FactPlayerSeason in PQ (1 if there exists a row in FactTransfer with same PlayerKey and ArrivalClubKey = ClubKey and transfer date in 2023–24). Then measure:

```dax
New Signings Goals = CALCULATE(SUM(FactPlayerSeason[Goals]), FactPlayerSeason[IsNewSigning] = 1)
New Signings Assists = CALCULATE(SUM(FactPlayerSeason[Assists]), FactPlayerSeason[IsNewSigning] = 1)
```

If you don't add IsNewSigning in PQ, you can do it in DAX with a measure using EXISTS/IN over FactTransfer — more complex.

---

### F.5 Page 5 — Fans, stadium & summary

```dax
Attendance = SUM(FactAttendance[Attendance])
Avg Attendance = AVERAGE(FactAttendance[Attendance])
Capacity Utilization Pct = AVERAGE(FactAttendance[FillRatePct])
// Or: DIVIDE(SUM(FactAttendance[Attendance]), SUM(FactAttendance[StadiumCapacity]), BLANK())

Squad Value Rank = RANKX(ALL(DimClub), [Net Transfer Spend], [Net Transfer Spend], DESC, Dense)

Final Rank = RANKX(ALL(DimClub), [Total Points], [Total Points], DESC, Dense)
// Use at matchday 38 or in a summary table
```

---

### F.6 Optional: StandingsByMatchday as calculated table

If you did not build it in Power Query:

```dax
StandingsByMatchday =
ADDCOLUMNS(
  SUMMARIZE(
    CROSSJOIN(VALUES(DimClub[ClubKey]), VALUES(DimMatch[Matchday])),
    DimClub[ClubKey],
    DimMatch[Matchday]
  ),
  "CumulativePoints",
  VAR _Club = DimClub[ClubKey]
  VAR _MD = DimMatch[Matchday]
  RETURN CALCULATE(SUM(FactClubMatch[Points]), FactClubMatch[ClubKey] = _Club, DimMatch[Matchday] <= _MD, ALL(DimMatch[Matchday])),
  "Rank",
  RANKX(
    FILTER(ALL(DimClub), DimClub[ClubKey] = EARLIER(DimClub[ClubKey])),
    [CumulativePoints],
    [CumulativePoints],
    DESC,
    Dense
  )
)
```

(Note: RANKX over FILTER in ADDCOLUMNS is tricky; a simpler approach is to build StandingsByMatchday in Power Query with running sum and rank, or use the **Cumulative Points** and **League Rank** measures in a visual with DimMatch[Matchday] and DimClub.)

---

## Part G — Checklist

- [ ] All staging queries load and apply data quality fixes (dates, numbers, trim, delimiters).
- [ ] DimClub has no duplicate names; ClubKey is integer.
- [ ] DimPosition, DimPlayer, DimDate, DimMatch built and keys consistent.
- [ ] FactClubMatch has exactly 2 rows per match (760 rows for 380 matches).
- [ ] All Merge joins succeed (no unexpected null keys); external clubs in transfers handled (null or "External").
- [ ] Relationships created in Model view; filter direction Single from dimension to fact.
- [ ] Measures created and tested (e.g. Total Points = 38×3 max per club; rank 1–20).
- [ ] StandingsByMatchday or rank/accumulation measures used for Page 1 rank evolution visual.

This ETL path takes you from the current raw files and data quality issues to the star schema and measures required by the dashboard specification.
