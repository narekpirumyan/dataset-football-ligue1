# ETL: From Raw Data to Star Schema (Power Query) + DAX Measures

**Audience:** You know ETL in Python (pandas, read_csv, merge, transform). This guide maps those ideas to **Power Query (M)** and **Power BI**, then adds **DAX measures**.

**References:**  
- [0_POWER_BI_DASHBOARD_SPECIFICATION.md](./0_POWER_BI_DASHBOARD_SPECIFICATION.md) — data quality issues by file  
- [1_POWER_BI_STAR_SCHEMA_DESIGN.md](./1_POWER_BI_STAR_SCHEMA_DESIGN.md) — target schema and relationships  

**Data sources:** CSV/TSV files in Dataset; **clubs.xlsx** (when present) for club list and attributes (Staging_Clubs → DimClub); **players.xlsx** (when present) for player age, nationality, salary, and market value (Staging_Players → DimPlayer). See dashboard spec “Available Data Files” and “Gaps and data sources”.

**Scope (squad/players):** The model supports **both field players and goalkeepers**. Do not filter out any position in staging or dimensions. **Position** is stored as an attribute on **DimPlayer** (from `position` in player_stats) and includes "Goalkeeper" and all field positions; use it in reports to scope visuals (e.g. goals/assists for outfield, clean_sheets/saves for goalkeepers). See dashboard spec Page 2 for per-visual scope.

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
- Build **dimensions first** (DimClub, DimPlayer, DimDate), then **facts** that reference them via Merge.
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
| **Position** consistency | **Trim** position; optionally **Replace Values** for known typos (e.g. "Left Back" → "Left back") so DimPlayer has consistent position names. Keep **Goalkeeper** and all field positions (do not filter out any position). |

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
| **fill_rate** European decimal: `96,6` | **Do not** set the column type to Number/Decimal before fixing: Power Query will interpret `96,6` as `966` (comma dropped). **Fix first:** Add a **Custom Column** (e.g. FillRatePct) with formula `Number.FromText(Text.Replace(Text.Trim(Text.From([fill_rate])), ",", "."))` so `96,6` → `96.6`. Remove the original **fill_rate** column, rename the new column to **fill_rate** (or **FillRatePct**), then set type **Decimal**. **If already 966:** Add Custom Column `if [fill_rate] > 100 then [fill_rate] / 10 else [fill_rate]`, then replace/rename and set type Decimal. |
| **date** format | Same as match_date: normalise to **Date**. |
| **Consistency with match_results** | Use stadium_attendance as the single source for attendance/capacity/fill_rate; no need to merge match_results attendance into this table. |

**Recommended staging query name:** `Staging_StadiumAttendance`

---

### B.5 disciplinary_sanctions.csv

| Issue | Fix in Power Query |
|-------|--------------------|
| **sanction_date** | Normalise to **Date** (same as above). |
| **Empty suspension_matches, fine_euros** | Replace null with 0 for **SuspensionMatches** (Whole Number); leave **FineEuros** as null or 0, type **Decimal**/Currency. |
| **All columns** | Keep **player** (player name from source), player_id, club, sanction_id, sanction_type, reason, sanction_date, suspension_matches, fine_euros. Do not drop any column. |

**Recommended staging query name:** `Staging_Sanctions`

---

### B.6 players.xlsx (player dimension enrichment)

**Structure:** `players.xlsx` contains **separate sheets** for **field players** and **goalkeepers** (e.g. "FieldPlayers" and "Goalkeepers", or similar names). Both must be loaded and **combined into one table** so DimPlayer can be enriched for all players (field + goalkeepers) in a single merge.

| Issue | Fix in Power Query |
|-------|--------------------|
| **Multiple sheets** | Load **each sheet** as a separate query (e.g. `Players_Field`, `Players_Goalkeepers`), or use **Get Data** → **Excel** → select **multiple worksheets** and combine. Ensure both tables end up with the **same column set**: at least **player_id**, and whichever of Age, Nationality, Salary, MarketValue exist. If one sheet has different column names, rename to match the other (e.g. both must have **player_id**, **Age**, **Nationality**, **Salary**, **MarketValue**). Then **Append** (Ajouter des requêtes) the two tables into one — field players first, goalkeepers second (or vice versa). **Remove Duplicates** on **player_id** if a player appears in both (keep one row). |
| **player_id** | Must match `player_stats_season` and other sources (e.g. "J0001"). **Trim** the column on the combined table; ensure no leading/trailing spaces so the merge with DimPlayer base succeeds. |
| **Column names** | Standardise across sheets: rename to **player_id**, **Age**, **Nationality**, **Salary**, **MarketValue** for consistency with the schema. If a sheet lacks a column (e.g. no MarketValue for goalkeepers), add a column with **null** so the combined table has the same columns. |
| **Types** | Age → Whole Number; Nationality → Text; Salary, MarketValue → Decimal. Nulls allowed for missing values. |

**Recommended staging query name:** `Staging_Players` (one table = all players from both sheets).  

**Note:** If your workbook uses different sheet names (e.g. "Joueurs", "Gardiens"), use those when selecting worksheets; the steps are the same (load both → same column layout → append → dedupe on player_id → trim and set types).

---

### B.7 Cross-file consistency

- **Club names:** Build DimClub from **distinct** names collected from all staging queries; use **Trim** and **Replace Values** once in DimClub so spelling is unique (e.g. "Paris SG" everywhere). All facts then resolve club via Merge to DimClub, so no need to change club names in each source.
- **player_id:** Use as natural key; ensure Trim so "J0001" does not become " J0001 ". Staging_PlayerStats, Staging_Transfers, Staging_Sanctions, and Staging_Players (players.xlsx) should use consistent player_id so the DimPlayer merge succeeds.

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

Follow this order so that every query that does a **Merge** has the referenced query already built. Target schema and all source columns are defined in [1_POWER_BI_STAR_SCHEMA_DESIGN.md](./1_POWER_BI_STAR_SCHEMA_DESIGN.md) (Section 3 — Available sources; Sections 3.1–3.4 dimensions; Section 4 — Fact tables). **Do not drop columns** from source files; keep all attributes in dimensions and facts even if not used in the dashboard.

### D.1 Staging queries (data quality only)

Create these first; no Merge to dimensions.

1. **Staging_MatchResults**  
   - Source: `match_results_2023_2024.csv`.  
   - Steps: Promote headers; trim text columns; fix **match_date** (Custom Column or helper) → type Date; set **matchday** Whole Number, **home_score** / **away_score** Whole Number; **attendance** Whole Number (nulls allowed); remove duplicates on **match_id** if any.  
   - Keep columns: match_id, matchday, match_date, home_team, away_team, home_score, away_score, stadium, referee, attendance.

2. **Staging_PlayerStats**  
   - Source: `player_stats_season.csv`.  
   - Steps: Promote headers; trim all text columns; fix **minutes_played** (strip " min", number); fix **goals**, **assists**, **matches_played**, **starts**, **yellow_cards**, **red_cards** (trim spaces, then Number.FromText); set correct types; trim **position**.  
   - **Keep all players** (field players and goalkeepers); do not filter by position. **position** must include "Goalkeeper" and all outfield positions. Keep **clean_sheets** and **saves** (null for non-GK is expected; used for goalkeeper-specific visuals).  
   - Keep all columns needed for FactPlayerSeason and for DimPlayer source (including position).

3. **Staging_Transfers**  
   - Source: `transfers_2023_2024.txt` (Tab).  
   - Steps: Trim text; parse **amount_ME** (helper or Custom Column); parse **transfer_date** → Date; keep **all** source columns: transfer_id, **player**, player_id, departure_club, arrival_club, transfer_type, amount_ME (cleaned), transfer_date (as date), **agent**. Do not drop any column.

4. **Staging_StadiumAttendance**  
   - Source: `stadium_attendance.csv` (Semicolon).  
   - Steps: Trim text columns. **fill_rate:** do **not** change type to Number before fixing — otherwise `96,6` becomes `966`. Add **Custom Column** (e.g. FillRatePct) with `Number.FromText(Text.Replace(Text.Trim(Text.From([fill_rate])), ",", "."))`, remove original **fill_rate**, rename new column to **fill_rate** or **FillRatePct**, set type **Decimal**. Fix **date** → Date (same as match_date). Set **attendance**, **stadium_capacity** to Whole Number.

5. **Staging_Sanctions**  
   - Source: `disciplinary_sanctions.csv`.  
   - Steps: Trim; **sanction_date** → Date; **suspension_matches** → Whole Number (null → 0); **fine_euros** → Decimal (null allowed).

6. **Staging_Clubs** (when `clubs.xlsx` is present)  
   - **Source:** `clubs.xlsx` (Excel).  
   - **Goal:** One table with club list and attributes for DimClub (single source of truth).  
   - **Steps:** Get Data → Excel → `clubs.xlsx` → select the worksheet. Promote first row as headers. Select/rename columns to **ClubName**, **StadiumName** (or Stadium), **Capacity** (or equivalent). **Trim** ClubName. Set types: ClubName Text, StadiumName Text, Capacity Whole Number.  
   - **Output:** One table **Staging_Clubs** with columns ClubName, StadiumName, Capacity. Used in **D.2 DimClub** when the file is present.

7. **Staging_Players** (enriches DimPlayer — **field players and goalkeepers**)  
   - **Source:** `players.xlsx` (Excel) with **two sheets**: one for field players, one for goalkeepers (sheet names may vary, e.g. "FieldPlayers", "Goalkeepers").  
   - **Goal:** One combined table with all players (field + goalkeepers) and columns: **player_id**, **Age**, **Nationality**, **Salary**, **MarketValue**, so DimPlayer can be enriched in a single merge.  
   - **Steps:**  
     1. **Load the field players sheet:** Get Data → Excel → `players.xlsx` → select the **field players** worksheet. Promote first row as headers. Select/keep columns that map to player_id, Age, Nationality, Salary, MarketValue; rename to standard names if needed. Optionally add a column **SourceSheet** = "Field" (for traceability; not required for DimPlayer).  
     2. **Load the goalkeepers sheet:** New query or Get Data again → same file → select the **goalkeepers** worksheet. Promote first row as headers. **Ensure the same column names** as the field players table (rename columns if the sheet uses different names). If the goalkeepers sheet is missing a column (e.g. MarketValue), add a **Custom Column** with value null. Optionally add **SourceSheet** = "Goalkeeper".  
     3. **Combine:** **Append** (Ajouter des requêtes) the goalkeepers table to the field players table (or the other way around). Result = one table with all players from both sheets.  
     4. **Remove Duplicates** on **player_id** (in case a player appears in both sheets; keep one row).  
     5. **Trim** the **player_id** column so it matches `Staging_PlayerStats` (e.g. "J0001").  
     6. Set types: **player_id** Text, **Age** Whole Number, **Nationality** Text, **Salary** Decimal, **MarketValue** Decimal.  
   - **Output:** One table **Staging_Players** with columns player_id, Age, Nationality, Salary, MarketValue (and optionally SourceSheet). Used in **D.3 DimPlayer** (merge on player_id to add Age, Nationality, Salary, MarketValue for all players).

---

### D.2 DimClub

- **Source (when `clubs.xlsx` is present):** **Staging_Clubs** (from clubs.xlsx). Optionally append any distinct club names from other staging tables that are not in Staging_Clubs (e.g. "External" for non-Ligue 1 transfer clubs) so all facts can resolve to a row.
- **Source (when `clubs.xlsx` is absent):** Combine distinct club names from all staging tables (match_results, player_stats, transfers, stadium_attendance, sanctions).
- **Steps when using Staging_Clubs:**
  1. Start from **Staging_Clubs** (columns ClubName, StadiumName, Capacity). **Trim** ClubName if not already done in staging.
  2. **Add Index Column** (1, 1) → rename to **ClubKey**.
  3. (Optional) Append missing clubs: build a table of distinct ClubName from all other sources; **anti-join** or **except** with Staging_Clubs[ClubName]; append those rows to DimClub with null StadiumName/Capacity; renumber ClubKey or add new keys for appended rows.
- **Steps when not using clubs.xlsx:**
  1. Create one table per source with one column "ClubName" (from home_team, away_team, club, departure_club, arrival_club), then **Append Queries** → **Remove Duplicates** → **Trim** → **Add Index Column** (1, 1) → **ClubKey**.
  2. Optionally merge with grouped stadium_attendance (Group By club → Max capacity, First stadium) to add StadiumName and Capacity.
- **Output columns:** ClubKey (Whole Number), ClubName (Text), optionally StadiumName, Capacity.

**M sketch (fallback when clubs.xlsx is absent):**

```powerquery
// DimClub: combine clubs from all sources (use when Staging_Clubs not available)
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

### D.3 DimPlayer

- **Source:** `Staging_PlayerStats` (distinct player_id, full_name, position) and `Staging_Players` (from players.xlsx) to enrich with Age, Nationality, Salary, MarketValue.  
- **Steps:**  
  1. From **Staging_PlayerStats**: select **player_id**, **full_name**, **position** → **Remove Duplicates** (on player_id) → **Trim** all three columns → **Add Index Column** (1, 1) → rename the index column to **PlayerKey**; rename **player_id** to **PlayerId**, **full_name** to **FullName**, **position** to **Position**.  
  2. **Merge** with **Staging_Players**: left = current table (DimPlayer base), right = **Staging_Players**; join on **PlayerId** = **player_id** (or the player_id column name in Staging_Players). Join kind: **Left Outer** (keep all players even if not in Excel).  
  3. Expand the merged column: check only **Age**, **Nationality**, **Salary**, **MarketValue** (uncheck “Select all”; use original column names or rename to these).  
  4. Set types: PlayerKey Whole Number, PlayerId Text, FullName Text, Position Text, Age Whole Number, Nationality Text, Salary Decimal, MarketValue Decimal.  
- **Output:** PlayerKey, PlayerId, FullName, Position, Age, Nationality, Salary, MarketValue.

---

### D.4 DimDate

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

### D.5 DimMatch

- **Source:** `Staging_MatchResults`.  
- **Steps:**
  1. **Merge** with DimClub: left = Staging_MatchResults[home_team], right = DimClub[ClubName] → expand only **ClubKey** → rename to **HomeClubKey**.
  2. **Merge** with DimClub again: left = Staging_MatchResults[away_team], right = DimClub[ClubName] → expand only **ClubKey** → rename to **AwayClubKey**.
  3. Select columns: **match_id** (rename to **MatchKey**), **matchday** (Matchday), **match_date** (MatchDate), **HomeClubKey**, **AwayClubKey**, **stadium** (Stadium), **referee** (Referee), **attendance** (Attendance; keep all source attributes).
  4. Set types: MatchKey Text, Matchday Whole Number, MatchDate Date, HomeClubKey/AwayClubKey Whole Number, Attendance Whole Number (null allowed).
- **Output:** MatchKey, Matchday, MatchDate, HomeClubKey, AwayClubKey, Stadium, Referee, Attendance.

---

### D.6 FactClubMatch

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

### D.7 FactPlayerSeason

- **Source:** `Staging_PlayerStats`.  
- **Scope:** **All players** (field players and goalkeepers). No filtering by position; position is stored on DimPlayer and can be used for filtering in reports.
- **Steps:**
  1. **Merge** DimClub on club → ClubKey; DimPlayer on player_id → PlayerKey.
  2. **Add Column** MatchesMissed = 38 - [matches_played]. (Goalkeepers may have fewer than 38 matches; the formula still applies; use DimPlayer[Position] in reports to interpret.)
  3. Select and keep **all** columns from source: PlayerKey, ClubKey, Goals, Assists, MinutesPlayed, MatchesPlayed, MatchesMissed, Starts, YellowCards, RedCards, Shots, ShotsOnTarget, CleanSheets, Saves, SuccessfulDribbles, Interceptions, SuccessfulTackles.
  4. Set all types (keys Whole Number; measures Whole Number or Decimal as appropriate).
- **Output:** PlayerKey, ClubKey, Goals, Assists, MinutesPlayed, MatchesPlayed, MatchesMissed, Starts, YellowCards, RedCards, Shots, ShotsOnTarget, CleanSheets, Saves, SuccessfulDribbles, Interceptions, SuccessfulTackles (all attributes from player_stats retained).

---

### D.8 FactTransfer

- **Source:** `Staging_Transfers`.  
- **Steps:**
  1. **Merge** DimPlayer on player_id → PlayerKey.
  2. **Merge** DimClub on departure_club → DepartureClubKey (clubs not in DimClub: use **null** or add "External" to DimClub and map there).
  3. **Merge** DimClub on arrival_club → ArrivalClubKey.
  4. Add **TransferDateKey**: same as transfer_date (date) for relationship to DimDate — ensure this date exists in DimDate (your calendar 2023-07-01 to 2024-08-31 covers it).
  5. Select: TransferKey (transfer_id), PlayerKey, **PlayerName** (from transfers.player), DepartureClubKey, ArrivalClubKey, TransferDateKey, AmountME, TransferType, Agent (keep all source attributes).
- **Output:** TransferKey, PlayerKey, PlayerName, DepartureClubKey, ArrivalClubKey, TransferDateKey, AmountME, TransferType, Agent.

---

### D.9 FactSanction

- **Source:** `Staging_Sanctions`.  
- **Steps:** **Merge** DimPlayer (player_id → PlayerKey), DimClub (club → ClubKey); add **SanctionDateKey** = sanction_date (must be in DimDate). Select: SanctionKey, PlayerKey, **PlayerName** (from disciplinary_sanctions.player), ClubKey, SanctionDateKey, SanctionType, Reason, SuspensionMatches, FineEuros (keep all source attributes). Set types.
- **Output:** SanctionKey, PlayerKey, PlayerName, ClubKey, SanctionDateKey, SanctionType, Reason, SuspensionMatches, FineEuros.

---

### D.10 FactAttendance

- **Source:** `Staging_StadiumAttendance`.  
- **Steps:** **Merge** DimClub (club → ClubKey); add **DateKey** = date column (for DimDate). Select: ClubKey, Matchday, DateKey, **Opponent**, Attendance, StadiumCapacity, FillRatePct, **Weather**, **TemperatureC** (keep all source attributes). Optionally: **Merge** with DimMatch or Staging_MatchResults on (home_team = club, matchday, match_date ≈ date) to get MatchKey.
- **Output:** ClubKey, Matchday, DateKey, MatchKey (optional), Opponent, Attendance, StadiumCapacity, FillRatePct, Weather, TemperatureC.

---

### D.11 Optional: StandingsByMatchday

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
5. **FactTransfer[PlayerKey]** → **DimPlayer[PlayerKey]** (Many to One, Single).
6. **FactTransfer[DepartureClubKey]** → **DimClub[ClubKey]** (Many to One, Single).
7. **FactTransfer[ArrivalClubKey]** → **DimClub[ClubKey]** (Many to One, Single).
8. **FactTransfer[TransferDateKey]** → **DimDate[DateKey]** (Many to One, Single).
9. **FactSanction[PlayerKey]** → **DimPlayer[PlayerKey]** (Many to One, Single).
10. **FactSanction[ClubKey]** → **DimClub[ClubKey]** (Many to One, Single).
11. **FactSanction[SanctionDateKey]** → **DimDate[DateKey]** (Many to One, Single).
12. **FactAttendance[ClubKey]** → **DimClub[ClubKey]** (Many to One, Single).
13. **FactAttendance[DateKey]** → **DimDate[DateKey]** (Many to One, Single).

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

**Scope:** All players (field + goalkeepers). Use **DimPlayer[Position]** (or a position slicer) to filter when needed: e.g. goals/assists leaders for "Outfield only", or goalkeeper-specific metrics (CleanSheets, Saves) when position = Goalkeeper. For "value vs performance" scatter: field players → Goals/Assists; goalkeepers → CleanSheets/Saves (if FactPlayerSeason has those columns).

```dax
Total Goals = SUM(FactPlayerSeason[Goals])
Total Assists = SUM(FactPlayerSeason[Assists])
Goals Plus Assists = [Total Goals] + [Total Assists]

Team Goals = SUM(FactPlayerSeason[Goals])   // in club filter context

Contribution Pct = DIVIDE([Total Goals], [Team Goals], BLANK())   // use for one player in club context

Total Minutes Played = SUM(FactPlayerSeason[MinutesPlayed])
Matches Missed = SUM(FactPlayerSeason[MatchesMissed])

// Optional — goalkeeper performance (if FactPlayerSeason has CleanSheets, Saves):
Total Clean Sheets = SUM(FactPlayerSeason[CleanSheets])
Total Saves = SUM(FactPlayerSeason[Saves])
// Use with filter DimPlayer[Position] = "Goalkeeper" for GK-only visuals.
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

- [ ] All staging queries load and apply data quality fixes (dates, numbers, trim, delimiters); Staging_Clubs (clubs.xlsx) when present for DimClub; Staging_Players (players.xlsx) for DimPlayer enrichment.
- [ ] DimClub has no duplicate names; ClubKey is integer (from Staging_Clubs when clubs.xlsx present, else from distinct names across sources).
- [ ] DimPlayer (with Position, Age, Nationality, Salary, MarketValue from Staging_Players and Staging_PlayerStats), DimDate, DimMatch built and keys consistent.
- [ ] FactClubMatch has exactly 2 rows per match (760 rows for 380 matches).
- [ ] All Merge joins succeed (no unexpected null keys); external clubs in transfers handled (null or "External").
- [ ] Relationships created in Model view; filter direction Single from dimension to fact.
- [ ] Measures created and tested (e.g. Total Points = 38×3 max per club; rank 1–20).
- [ ] StandingsByMatchday or rank/accumulation measures used for Page 1 rank evolution visual.

This ETL path takes you from the current raw files and data quality issues to the star schema and measures required by the dashboard specification.
