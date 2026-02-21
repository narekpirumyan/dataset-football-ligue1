# Power BI Data Model Schema — Final Design

**Purpose:** This document defines the final star schema data model for the Ligue 1 Dashboard as implemented in Power BI. This schema replaces the original CSV/TSV data sources and provides a normalized, relationship-based structure optimized for analytical reporting.

**Context:** The dashboard uses a star schema design with dimension tables (DimClub, DimDate, DimPlayer, DimMatch) and fact tables (FactSanction, FactAttendance, FactClubMatch, FactPlayerSeason, FactTransfer) connected via foreign key relationships.

---

## Schema Overview

The data model follows a star schema pattern with:
- **4 Dimension Tables**: `DimClub`, `DimDate`, `DimPlayer`, `DimMatch`
- **5 Fact Tables**: `FactSanction`, `FactAttendance`, `FactClubMatch`, `FactPlayerSeason`, `FactTransfer`

All relationships are one-to-many (1:*), with dimensions on the "1" side and facts on the "*" side.

---

## Dimension Tables

### DimClub
**Purpose:** Stores static information about each football club.

| Attribute | Type | Description |
|-----------|------|-------------|
| `ClubKey` | Integer (PK) | Primary key - surrogate key for relationships |
| `club_id` | Text | Natural identifier for the club |
| `ClubName` | Text | Full name of the club |
| `budget_ME` | Decimal | Club budget in millions of euros |
| `capacity` | Integer | Stadium capacity |
| `city` | Text | City where the club is located |
| `coach` | Text | Head coach name |
| `colors` | Text | Club colors |
| `External` | Boolean/Text | Flag for external/non-Ligue 1 clubs |
| `founded` | Integer | Year club was founded |
| `president` | Text | Club president name |
| `stadium` | Text | Stadium name |

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
| `DateKey` | Date (PK) | Primary key - date value used for relationships |
| `Date` | Date | Date value for display |
| `Day` | Integer | Day of month (1-31) |
| `Month` | Integer | Month number (1-12) |
| `Year` | Integer | Year (e.g., 2023, 2024) |

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
| `PlayerKey` | Integer (PK) | Primary key - surrogate key for relationships |
| `Playerid` | Text | Natural identifier for the player |
| `FullName` | Text | Player's full name |
| `Position` | Text | Player position (e.g., Forward, Midfielder, Defender, Goalkeeper) |
| `Age` | Integer | Player age |
| `Nationality` | Text | Player nationality |
| `MarketValue` | Decimal (Measure) | Player market value (can be aggregated) |

**Relationships:**
- `DimPlayer (1)` → `FactSanction (*)` via `PlayerKey`
- `DimPlayer (1)` → `FactPlayerSeason (*)` via `PlayerKey`
- `DimPlayer (1)` → `FactTransfer (*)` via `PlayerKey`

---

### DimMatch
**Purpose:** Stores descriptive information about each football match.

| Attribute | Type | Description |
|-----------|------|-------------|
| `MatchKey` | Text (PK) | Primary key - match identifier |
| `DateKey` | Date (FK) | Foreign key to DimDate |
| `Matchday` | Integer | Matchday number (1-38) |
| `HomeClubKey` | Integer (FK) | Foreign key to DimClub (home team) |
| `AwayClubKey` | Integer (FK) | Foreign key to DimClub (away team) |
| `Stadium` | Text | Stadium where match was played |
| `Referee` | Text | Match referee name |
| `Attendance` | Integer | Match attendance (descriptive attribute) |

**Relationships:**
- `DimMatch (1)` → `FactAttendance (*)` via `MatchKey`
- `DimMatch (1)` → `FactClubMatch (*)` via `MatchKey`
- `DimDate (1)` → `DimMatch (*)` via `DateKey` (dimension-to-dimension)

**Note:** `Attendance` in `DimMatch` is a descriptive attribute of the match, while `FactAttendance` contains attendance measures in a club/date context.

---

## Fact Tables

### FactSanction
**Purpose:** Records details about sanctions issued to clubs or players.

| Attribute | Type | Description |
|-----------|------|-------------|
| `SanctionKey` | Text (PK) | Primary key - sanction identifier |
| `ClubKey` | Integer (FK) | Foreign key to DimClub |
| `PlayerKey` | Integer (FK) | Foreign key to DimPlayer (nullable if club sanction) |
| `SanctionDateKey` | Date (FK) | Foreign key to DimDate |
| `SanctionType` | Text | Type of sanction |
| `Reason` | Text | Reason for sanction |
| `SuspensionMatches` | Integer (Measure) | Number of matches suspended |
| `FineEuros` | Decimal (Measure) | Fine amount in euros |

**Grain:** One row per sanction event.

---

### FactAttendance
**Purpose:** Stores attendance-related metrics for matches.

| Attribute | Type | Description |
|-----------|------|-------------|
| `ClubKey` | Integer (FK) | Foreign key to DimClub (home team) |
| `DateKey` | Date (FK) | Foreign key to DimDate |
| `MatchKey` | Text (FK) | Foreign key to DimMatch (optional) |
| `Matchday` | Integer (Measure) | Matchday number |
| `Opponent` | Text | Opposing team name |
| `Attendance` | Integer (Measure) | Number of attendees |
| `StadiumCapacity` | Integer (Measure) | Stadium capacity |
| `FillRatePct` | Decimal (Measure) | Fill rate percentage |
| `Weather` | Text | Weather conditions |
| `TemperatureC` | Decimal (Measure) | Temperature in Celsius |

**Grain:** One row per club per matchday (home matches).

---

### FactClubMatch
**Purpose:** Records match statistics specific to a club's performance in a given match.

| Attribute | Type | Description |
|-----------|------|-------------|
| `MatchKey` | Text (FK) | Foreign key to DimMatch |
| `ClubKey` | Integer (FK) | Foreign key to DimClub |
| `IsHome` | Boolean | True if club is home team, False if away |
| `Points` | Integer (Measure) | Points earned (3/1/0) |
| `GoalsFor` | Integer (Measure) | Goals scored by the club |
| `GoalsAgainst` | Integer (Measure) | Goals conceded by the club |
| `Result` | Text | Match result: "W" (Win), "D" (Draw), "L" (Loss) |

**Grain:** One row per club per match (two rows per match: one for home team, one for away team).

**Design Rationale:** Storing two rows per match (one per participating club) simplifies filtering and aggregation by club. When filtering by a specific club, all matches for that club are automatically included without complex home/away logic in measures.

---

### FactPlayerSeason
**Purpose:** Stores player performance statistics aggregated by season.

| Attribute | Type | Description |
|-----------|------|-------------|
| `PlayerKey` | Integer (FK) | Foreign key to DimPlayer |
| `ClubKey` | Integer (FK) | Foreign key to DimClub |
| `Goals` | Integer (Measure) | Total goals scored |
| `Assists` | Integer (Measure) | Total assists |
| `MinutesPlayed` | Integer (Measure) | Total minutes played |
| `MatchesPlayed` | Integer (Measure) | Number of matches played |
| `MatchesMissed` | Integer (Measure) | Number of matches missed |
| `Starts` | Integer (Measure) | Number of starts |
| `YellowCards` | Integer (Measure) | Total yellow cards |
| `RedCards` | Integer (Measure) | Total red cards |
| `Shots` | Integer (Measure) | Total shots |
| `ShotsOnTarget` | Integer (Measure) | Shots on target |
| `CleanSheets` | Integer (Measure) | Clean sheets (primarily for goalkeepers) |
| `Saves` | Integer (Measure) | Saves (primarily for goalkeepers) |
| `SuccessfulDribbles` | Decimal (Measure) | Successful dribbles |
| `Interceptions` | Decimal (Measure) | Interceptions |
| `SuccessfulTackles` | Decimal (Measure) | Successful tackles |

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

5. **Measure Aggregation:** All numeric values marked as measures (Σ) can be aggregated using SUM, AVERAGE, COUNT, etc., in Power BI visuals.

6. **Flexible Filtering:** Relationships allow filtering by club, player, date, or match, with automatic propagation to related fact tables.

---

## Implementation Notes

- **Power BI Relationships:** All relationships should be configured with "Single" filter direction from dimension to fact tables.
- **DAX Measures:** Complex calculations (rank, form, contribution percentages) should be implemented as DAX measures rather than calculated columns.
- **HTML Content Visuals:** When using HTML Content visuals, data must be queried from this underlying model using DAX or Power Query, then embedded into custom HTML/JavaScript.
- **Performance:** The star schema design optimizes query performance by minimizing joins and enabling efficient filtering through dimension tables.

---

This schema provides the foundation for all dashboard pages and enables comprehensive analysis of club performance, player statistics, transfers, sanctions, and attendance across the Ligue 1 season.
