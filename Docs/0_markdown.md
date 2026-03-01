# Power BI Data Model Schema — Final Design

**Purpose:** This document describes the star schema data model for the Ligue 1 Dashboard as implemented in Power BI. It is aligned with **Data_Model.bim**, which is the **source of truth** for tables, columns, and relationships.

**Context:** The dashboard uses a star schema with dimension tables (DimClub, DimDate, DimPlayer, DimMatch), fact tables (FactSanction, FactAttendance, FactClubMatch, FactPlayerSeason, FactTransfer), and one calculated table (RivalryPair), connected via foreign key relationships.

---

## Schema Overview

The data model follows a star schema pattern with:
- **4 Dimension Tables**: `DimClub`, `DimDate`, `DimPlayer`, `DimMatch`
- **5 Fact Tables**: `FactSanction`, `FactAttendance`, `FactClubMatch`, `FactPlayerSeason`, `FactTransfer`
- **1 Calculated Table**: `RivalryPair` (DAX) — one row per unordered pair of clubs that have played each other; used for rivalry visuals.

All relationships are one-to-many (1:*), with dimensions on the "1" side and facts on the "*" side.

---

## Dimension Tables

### DimClub
**Purpose:** Stores static information about each football club.

| Attribute | Type | Description |
|-----------|------|-------------|
| `ClubKey` | Integer (PK) | Primary key — surrogate key for relationships |
| `club_id` | Text | Natural identifier for the club |
| `ClubName` | Text | Full name of the club |
| `city` | Text | City where the club is located |
| `stadium` | Text | Stadium name |
| `capacity` | Text | Stadium capacity (as in Data_Model.bim) |
| `founded` | Text | Year club was founded (as in Data_Model.bim) |
| `president` | Text | Club president name |
| `coach` | Text | Head coach name |
| `budget_ME` | Text | Club budget in millions of euros (as in Data_Model.bim) |
| `colors` | Text | Club colors |
| `External` | Boolean | Flag for external/non-Ligue 1 clubs |
| `Club_URL` | Text | URL for club logo or external link |

**Relationships:**
- `DimClub (1)` → `FactSanction (*)` via `ClubKey`
- `DimClub (1)` → `FactAttendance (*)` via `ClubKey`
- `DimClub (1)` → `FactClubMatch (*)` via `ClubKey`
- `DimClub (1)` → `FactPlayerSeason (*)` via `ClubKey`
- `DimClub (1)` → `FactTransfer (*)` via `ArrivalClubKey` (arrivals)
- `DimClub (1)` → `FactTransfer (*)` via `DepartureClubKey` (departures)

---

### DimDate
**Purpose:** Standard date dimension for time-based analysis and filtering.

| Attribute | Type | Description |
|-----------|------|-------------|
| `DateKey` | Text (PK) | Primary key — date value used for relationships (as in Data_Model.bim) |
| `Date` | DateTime | Date value for display |
| `Year` | Integer | Year (e.g., 2023, 2024) |
| `Month` | Integer | Month number (1-12) |
| `Day` | Integer | Day of month (1-31) |

**Relationships:**
- `DimDate (1)` → `FactSanction (*)` via `SanctionDateKey`
- `DimDate (1)` → `FactAttendance (*)` via `DateKey`
- `DimDate (1)` → `DimMatch (*)` via `DateKey` (match date)
- `DimDate (1)` → `FactTransfer (*)` via `TransferDateKey`

**Note:** The relationship between `DimDate` and `DimMatch` is dimension-to-dimension, indicating that each match has a specific date.

---

### DimPlayer
**Purpose:** Stores static information about individual players.

| Attribute | Type | Description |
|-----------|------|-------------|
| `PlayerKey` | Integer (PK) | Primary key — surrogate key for relationships |
| `PlayerId` | Text | Natural identifier for the player |
| `FullName` | Text | Player's full name (unique per player; duplicates may include nationality in parentheses) |
| `Position` | Text | Player position (e.g., Forward, Midfielder, Defender, Goalkeeper) |
| `DateOfBirth` | DateTime | Player date of birth |
| `Nationality` | Text | Player nationality |
| `MarketValue` | Double | Player market value (aggregatable) |
| `Age` | Integer | Player age (calculated from date of birth) |

**Relationships:**
- `DimPlayer (1)` → `FactSanction (*)` via `PlayerKey`
- `DimPlayer (1)` → `FactPlayerSeason (*)` via `PlayerKey`
- `DimPlayer (1)` → `FactTransfer (*)` via `PlayerKey`

---

### DimMatch
**Purpose:** Stores descriptive information about each football match.

| Attribute | Type | Description |
|-----------|------|-------------|
| `MatchKey` | Text (PK) | Primary key — match identifier |
| `Matchday` | Integer | Matchday number (1-38) |
| `HomeClubKey` | Integer (FK) | Foreign key to DimClub (home team) |
| `AwayClubKey` | Integer (FK) | Foreign key to DimClub (away team) |
| `Stadium` | Text | Stadium where match was played |
| `Referee` | Text | Match referee name |
| `Attendance` | Integer | Match attendance (descriptive attribute) |
| `DateKey` | Text (FK) | Foreign key to DimDate |
| `HomeClubName` | Text (calculated) | LOOKUPVALUE(DimClub[ClubName], DimClub[ClubKey], HomeClubKey) — for display in rivalry history |
| `AwayClubName` | Text (calculated) | LOOKUPVALUE(DimClub[ClubName], DimClub[ClubKey], AwayClubKey) — for display in rivalry history |

**Relationships:**
- `DimMatch (1)` → `FactAttendance (*)` via `MatchKey`
- `DimMatch (1)` → `FactClubMatch (*)` via `MatchKey`
- `DimDate (1)` → `DimMatch (*)` via `DateKey` (dimension-to-dimension)

**Note:** `Attendance` in `DimMatch` is a descriptive attribute of the match; `FactAttendance` holds attendance in a club/date context. DimMatch also has measures (e.g. Goals Home, Goals Away, Is Rivalry Match) for rivalry visuals.

---

## Fact Tables

### FactSanction
**Purpose:** Records details about sanctions issued to clubs or players.

| Attribute | Type | Description |
|-----------|------|-------------|
| `SanctionKey` | Text (PK) | Primary key — sanction identifier |
| `PlayerKey` | Integer (FK) | Foreign key to DimPlayer |
| `ClubKey` | Integer (FK) | Foreign key to DimClub |
| `SanctionDateKey` | Text (FK) | Foreign key to DimDate |
| `SanctionType` | Text | Type of sanction |
| `Reason` | Text | Reason for sanction |
| `SuspensionMatches` | Integer | Number of matches suspended (aggregatable) |
| `FineEuros` | Integer | Fine amount in euros (aggregatable) |

**Grain:** One row per sanction event.

---

### FactAttendance
**Purpose:** Stores attendance-related metrics for matches.

| Attribute | Type | Description |
|-----------|------|-------------|
| `ClubKey` | Integer (FK) | Foreign key to DimClub (home team) |
| `Matchday` | Integer | Matchday number |
| `DateKey` | Text (FK) | Foreign key to DimDate |
| `MatchKey` | Text (FK) | Foreign key to DimMatch (from join with DimMatch) |
| `Opponent` | Text | Opposing team name |
| `Attendance` | Integer | Number of attendees |
| `StadiumCapacity` | Integer | Stadium capacity |
| `FillRatePct` | Double | Fill rate percentage |
| `Weather` | Text | Weather conditions |
| `TemperatureC` | Integer | Temperature in Celsius |

**Grain:** One row per club per matchday (home matches).

---

### FactClubMatch
**Purpose:** Records match statistics specific to a club's performance in a given match.

| Attribute | Type | Description |
|-----------|------|-------------|
| `MatchKey` | Text (FK) | Foreign key to DimMatch |
| `ClubKey` | Integer (FK) | Foreign key to DimClub |
| `IsHome` | Boolean | True if club is home team, False if away |
| `Points` | Integer | Points earned (3/1/0) |
| `GoalsFor` | Integer | Goals scored by the club |
| `GoalsAgainst` | Integer | Goals conceded by the club |
| `Result` | Text | Match result: "W" (Win), "D" (Draw), "L" (Loss) |
| `Goal difference` | Integer (calculated) | GoalsFor − GoalsAgainst |

**Grain:** One row per club per match (two rows per match: one for home team, one for away team).

**Design Rationale:** Storing two rows per match (one per participating club) simplifies filtering and aggregation by club. When filtering by a specific club, all matches for that club are automatically included without complex home/away logic in measures.

---

### FactPlayerSeason
**Purpose:** Stores player performance statistics aggregated by season.

| Attribute | Type | Description |
|-----------|------|-------------|
| `PlayerKey` | Integer (FK) | Foreign key to DimPlayer |
| `ClubKey` | Integer (FK) | Foreign key to DimClub |
| `Goals` | Integer | Total goals scored |
| `Assists` | Integer | Total assists |
| `MinutesPlayed` | Integer | Total minutes played |
| `MatchesPlayed` | Integer | Number of matches played |
| `MatchesMissed` | Text | Number of matches missed (as in Data_Model.bim) |
| `Starts` | Integer | Number of starts |
| `YellowCards` | Integer | Total yellow cards |
| `RedCards` | Integer | Total red cards |
| `Shots` | Integer | Total shots |
| `ShotsOnTarget` | Integer | Shots on target |
| `CleanSheets` | Integer | Clean sheets (primarily for goalkeepers) |
| `Saves` | Integer | Saves (primarily for goalkeepers) |
| `SuccessfulDribbles` | Integer | Successful dribbles |
| `Interceptions` | Integer | Interceptions |
| `SuccessfulTackles` | Integer | Successful tackles |

**Grain:** One row per player per season (aggregated statistics).

---

### FactTransfer
**Purpose:** Records information about player transfers between clubs.

| Attribute | Type | Description |
|-----------|------|-------------|
| `TransferKey` | Text (PK) | Primary key - transfer identifier |
| `PlayerKey` | Integer (FK) | Foreign key to DimPlayer |
| `ArrivalClubKey` | Integer (FK) | Foreign key to DimClub (destination club) |
| `DepartureClubKey` | Integer (FK) | Foreign key to DimClub (source club) |
| `TransferDateKey` | Date (FK) | Foreign key to DimDate |
| `TransferType` | Text | Type of transfer (Purchase, Loan, Free transfer, etc.) |
| `AmountME` | Decimal (Measure) | Transfer amount in millions of euros |
| `Agent` | Text | Agent name |

**Grain:** One row per transfer event.

**Note:** `ArrivalClubKey` and `DepartureClubKey` create two separate relationships to `DimClub`, allowing analysis of transfers in (arrivals) and transfers out (departures) separately.

---

## Calculated Table

### RivalryPair
**Purpose:** One row per unordered pair of clubs that have played each other (from DimMatch). Used for rivalry visuals (goals, sanctions, attendance, points per pair). No physical relationship to other tables; filtering and measures use `ClubKey1` and `ClubKey2` in DAX.

| Attribute | Type | Description |
|-----------|------|-------------|
| `ClubKey1` | Integer | Smaller of HomeClubKey and AwayClubKey for the pair |
| `ClubKey2` | Integer | Larger of HomeClubKey and AwayClubKey for the pair |
| `PairLabel` | Text | Display label, e.g. "Club A – Club B" |

**Grain:** One row per unique unordered pair (ClubKey1 < ClubKey2). Table is built in DAX from DimMatch.

---

## Relationship Summary

| From Table | To Table | Relationship Type | Key Columns |
|------------|----------|-------------------|-------------|
| FactSanction | DimClub | Many-to-One | ClubKey |
| FactSanction | DimPlayer | Many-to-One | PlayerKey |
| FactSanction | DimDate | Many-to-One | SanctionDateKey |
| FactAttendance | DimClub | Many-to-One | ClubKey |
| FactAttendance | DimDate | Many-to-One | DateKey |
| FactAttendance | DimMatch | Many-to-One | MatchKey |
| FactClubMatch | DimClub | Many-to-One | ClubKey |
| FactClubMatch | DimMatch | Many-to-One | MatchKey |
| FactPlayerSeason | DimClub | Many-to-One | ClubKey |
| FactPlayerSeason | DimPlayer | Many-to-One | PlayerKey |
| FactTransfer | DimPlayer | Many-to-One | PlayerKey |
| FactTransfer | DimClub | Many-to-One | ArrivalClubKey |
| FactTransfer | DimClub | Many-to-One | DepartureClubKey |
| FactTransfer | DimDate | Many-to-One | TransferDateKey |
| DimMatch | DimDate | Many-to-One | DateKey |

---

## Key Design Principles

1. **Star Schema Pattern:** Central fact tables connected to dimension tables via foreign keys enable efficient filtering and aggregation.

2. **Single Source of Truth:** Dimension tables (DimClub, DimPlayer) ensure consistent naming and attributes across all fact tables.

3. **Two-Row Match Design:** `FactClubMatch` stores two rows per match (one per club) to simplify club-based filtering and analysis.

4. **Time Intelligence:** `DimDate` enables time-based analysis across all fact tables that reference dates.

5. **Measure Aggregation:** Numeric columns used in visuals can be aggregated (SUM, AVERAGE, COUNT, etc.); complex logic is implemented as DAX measures in the model.

6. **Flexible Filtering:** Relationships allow filtering by club, player, date, or match, with automatic propagation to related fact tables.

---

## Implementation Notes

- **Source of truth:** The canonical definition of this model is **Data_Model.bim**. This document is kept in sync with it (tables, columns, calculated table RivalryPair, and key measures).
- **Power BI Relationships:** Relationships use "Single" filter direction from dimension to fact tables.
- **DAX Measures:** Complex calculations (rank, form, contribution %, rivalry KPIs, HTML cards) are implemented as DAX measures; calculated columns are used only where needed (e.g. HomeClubName, AwayClubName, Goal difference).
- **HTML Content Visuals:** Data is supplied via DAX measures that return HTML strings, using this model as the data source.
- **Performance:** The star schema minimizes joins and allows efficient filtering through dimension tables.

---

This schema provides the foundation for all dashboard pages and enables analysis of club performance, player statistics, transfers, sanctions, attendance, and rivalries across the Ligue 1 season.
