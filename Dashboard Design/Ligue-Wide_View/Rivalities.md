# Ligue-Wide View — Rivalities

**Category:** Head-to-head confrontations between clubs (rivalries, derbys, classics).

**Objective:** Two pages — (1) identify which rivalries are most anticipated and followed (attendance, fill rate); (2) for a selected rivalry, get a global view: who has the advantage, friction between teams (sanctions), and spectator turnout.

**Data source:** Power BI data model — `RivalryPair` (calculated table), `DimClub`, `DimMatch`, `FactClubMatch`, `FactAttendance`, `FactSanction`.

---

## Rivalry logic (one row per unordered pair)

A **rivalry** = one **unordered pair** of clubs = all matches between these two clubs (home and away). The calculated table **RivalryPair** has one row per pair with `ClubKey1` &lt; `ClubKey2` and **PairLabel** = "Club1 – Club2". All measures that count goals, wins, attendance, sanctions, etc. for a rivalry must include **both directions**: (Home = C1 AND Away = C2) OR (Home = C2 AND Away = C1).

**RivalryPair definition (DAX):**

```dax
RivalryPair =
VAR PairsWithOrder =
    ADDCOLUMNS(
        SUMMARIZE(DimMatch, DimMatch[HomeClubKey], DimMatch[AwayClubKey]),
        "ClubKey1", IF([HomeClubKey] < [AwayClubKey], [HomeClubKey], [AwayClubKey]),
        "ClubKey2", IF([HomeClubKey] < [AwayClubKey], [AwayClubKey], [HomeClubKey])
    )
VAR UnorderedPairs = SUMMARIZE(PairsWithOrder, [ClubKey1], [ClubKey2])
RETURN
    ADDCOLUMNS(
        UnorderedPairs,
        "PairLabel",
            LOOKUPVALUE(DimClub[ClubName], DimClub[ClubKey], [ClubKey1])
                & " – " & LOOKUPVALUE(DimClub[ClubName], DimClub[ClubKey], [ClubKey2])
    )
```

---

## Page 1 — Rivalities (overview: most anticipated rivalries)

**Purpose:** See which rivalries attract the most spectators and fill the stadiums. Selection of a rivalry is done via a table listing all pairs.

### 1.1 Table — List of PairLabel (rivalry selector)

**Visual type:** Table

**Purpose:** Display the list of all rivalries (one row per pair). The user selects one or more rows to filter the report to that rivalry. Use this table as the main **rivalry selector** (cross-filter other visuals on the same page and, if needed, on page 2).

| Role    | Field              | Notes |
|--------|--------------------|-------|
| **Rows** | `RivalryPair[PairLabel]` | One row per rivalry. Enable **selection** (single or multi) so that clicking a row filters the report by that pair. |

**Power BI:** Add a **Table** visual with **PairLabel** as the only column. In **Format** → **Row** (or **Selection**), enable row selection so that the table acts as a selector. Ensure this table is on the Rivalities page and that the two stacked bar charts below use the same **RivalryPair** context (they show one bar per rivalry; when a row is selected, other pages can use the same filter if filters are shared).

---

### 1.2 Stacked bar chart — Average attendance by rivalry

**Visual type:** Stacked bar chart

**Purpose:** Compare average attendance across rivalries to see which confrontations are most followed.

| Role      | Field / measure                    | Notes |
|----------|-------------------------------------|-------|
| **Y-axis**  | `RivalryPair[PairLabel]`           | One bar per rivalry. |
| **X-axis / Value** | `[Avg Attendance in Rivalry]` | Average attendance for matches of that pair (both home/away directions). |

**DAX (table RivalryPair):**

```dax
Avg Attendance in Rivalry =
VAR C1 = MAX(RivalryPair[ClubKey1])
VAR C2 = MAX(RivalryPair[ClubKey2])
RETURN
    CALCULATE(
        AVERAGE(FactAttendance[Attendance]),
        FILTER(DimMatch,
            (DimMatch[HomeClubKey] = C1 && DimMatch[AwayClubKey] = C2)
                || (DimMatch[HomeClubKey] = C2 && DimMatch[AwayClubKey] = C1)),
        REMOVEFILTERS(DimClub)
    )
```

If attendance is stored in **DimMatch[Attendance]** instead of FactAttendance, use:

```dax
Avg Attendance in Rivalry =
VAR C1 = MAX(RivalryPair[ClubKey1])
VAR C2 = MAX(RivalryPair[ClubKey2])
RETURN
    CALCULATE(
        AVERAGE(DimMatch[Attendance]),
        FILTER(DimMatch,
            (DimMatch[HomeClubKey] = C1 && DimMatch[AwayClubKey] = C2)
                || (DimMatch[HomeClubKey] = C2 && DimMatch[AwayClubKey] = C1))
    )
```

**Power BI:** Stacked bar chart → **Y-axis** = PairLabel, **Values** = Avg Attendance in Rivalry. Sort descending to see the most attended rivalries first.

---

### 1.3 Stacked bar chart — Average fill rate by rivalry

**Visual type:** Stacked bar chart

**Purpose:** Compare average stadium fill rate (%) across rivalries — which confrontations fill the stadiums the most.

| Role      | Field / measure                    | Notes |
|----------|-------------------------------------|-------|
| **Y-axis**  | `RivalryPair[PairLabel]`           | One bar per rivalry. |
| **X-axis / Value** | `[Avg Fill Rate in Rivalry]`    | Average fill rate for matches of that pair. |

**DAX (table RivalryPair):**

```dax
Avg Fill Rate in Rivalry =
VAR C1 = MAX(RivalryPair[ClubKey1])
VAR C2 = MAX(RivalryPair[ClubKey2])
RETURN
    CALCULATE(
        AVERAGE(FactAttendance[FillRatePct]),
        FILTER(DimMatch,
            (DimMatch[HomeClubKey] = C1 && DimMatch[AwayClubKey] = C2)
                || (DimMatch[HomeClubKey] = C2 && DimMatch[AwayClubKey] = C1)),
        REMOVEFILTERS(DimClub)
    )
```

**Power BI:** Stacked bar chart → **Y-axis** = PairLabel, **Values** = Avg Fill Rate in Rivalry. Format value as **Percentage**. Sort descending.

---

## Page 2 — Rivalities (2) (detail for one rivalry)

**Purpose:** Global view of a **selected rivalry**: head-to-head balance (wins, draws, goals), disciplinary friction (sanctions by reason), and chronological match list. The user selects a rivalry (e.g. via the PairLabel table on this page or via filter from page 1). For the HTML cards and the match table, the context must be **exactly two clubs** (slicer on `DimClub[ClubName]` with 2 selected, or selection of one row in the PairLabel table if the report is set up so that it filters the two clubs).

### 2.1 HTML Cards — Head-to-head summary (matches, wins, draws, total goals)

**Visual type:** HTML Content (one measure returns the full HTML).

**Purpose:** Display five summary cards for the selected rivalry: number of matches, wins for club 1, wins for club 2, draws, and total goals. Gives at a glance who has the advantage.

**Prerequisite:** Exactly **2 clubs** selected (e.g. via a slicer on `DimClub[ClubName]` or via the PairLabel table so that the two clubs of the pair are in context).

**DAX measures** (used by the HTML measure; context = 2 clubs selected in DimClub):

- **Rivalry Match Count** — number of matches between the two clubs.
- **Rivalry Wins Club1** / **Rivalry Wins Club2** — wins for the club with the smaller/larger ClubKey.
- **Rivalry Draws** — number of draws.
- **Rivalry Goals Club1** / **Rivalry Goals Club2** — goals scored by each club in those matches; total goals = sum of both.

(Full DAX for these measures is in the data model; they filter DimMatch with (Home=C1 AND Away=C2) OR (Home=C2 AND Away=C1).)

**HTML measure (table RivalryPair):** **Rivalry Head-to-Head Cards HTML** — builds one HTML block with 5 cards: Match count, Wins Club 1, Wins Club 2, Draws, Total goals. Uses a CSS grid (`grid-template-columns: repeat(5, 1fr)`, `width: 100%`) so cards adapt to the container. Three colour groups: **Matches** = grey (#E8E8E8 / #9e9e9e), **Wins/Draws** = blue (#E3F2FD / #0066CC), **Total goals** = green (#E8F5E9 / #1AAB40). Values show **0** explicitly when the result is zero (no blank or "–").

**Power BI:** Add an **HTML Content** visual; put **Rivalry Head-to-Head Cards HTML** in **Values**. Ensure the slicer (2 clubs) or the PairLabel selection filters this visual so that exactly two clubs are in context.

---

### 2.2 Stacked bar chart — Total sanctions per rivalry (Reason as legend)

**Visual type:** Stacked bar chart

**Purpose:** For each rivalry, show total number of sanctions, broken down by **Reason** (legend). Highlights which rivalries have the most friction and for what reasons (e.g. on-field brawl, red cards, etc.).

| Role      | Field / measure                    | Notes |
|----------|-------------------------------------|-------|
| **Y-axis**  | `RivalryPair[PairLabel]`           | One bar per rivalry. |
| **Values** | Count of sanctions                 | Total sanctions in matches of that pair. |
| **Legend** | `FactSanction[Reason]`            | Stack segments by sanction reason. |

**DAX:** A measure that counts sanctions for the current rivalry (both clubs, match dates of the pair), e.g. **Sanctions in Rivalry** (or **Sanctions in Rivalry by Reason** if you need to expose Reason). For the stacked bar, use **FactSanction** (or a table that has Reason) so that **Reason** can be used as the legend; the value is the count of sanctions in the context of the rivalry (filter match dates and clubs C1, C2).

**Power BI:** Stacked bar chart → **Y-axis** = RivalryPair[PairLabel], **Values** = measure that counts sanctions (e.g. COUNTROWS(FactSanction) in rivalry context), **Legend** = FactSanction[Reason]. Ensure the rivalry context (PairLabel selection or filter) restricts to matches between the two clubs and that sanctions are filtered to those match dates and clubs.

---

### 2.3 Table — Chronological list of matches for the selected rivalry

**Visual type:** Table (or timeline)

**Purpose:** List all matches between the two selected clubs in chronological order: matchday, date, home/away teams, goals, optionally stadium/attendance. Completes the picture of the rivalry over time.

**Prerequisite:** Exactly **2 clubs** selected (same as for the HTML cards). The table source is **DimMatch**; only rows for matches between these two clubs are shown by using a **visual-level filter**: **Is Rivalry Match** = TRUE.

**Calculated columns (DimMatch):**

```dax
HomeClubName = LOOKUPVALUE(DimClub[ClubName], DimClub[ClubKey], DimMatch[HomeClubKey])
AwayClubName = LOOKUPVALUE(DimClub[ClubName], DimClub[ClubKey], DimMatch[AwayClubKey])
```

**Measures (DimMatch):**

```dax
Goals Home = CALCULATE(SUM(FactClubMatch[GoalsFor]), FactClubMatch[ClubKey] = MAX(DimMatch[HomeClubKey]))
Goals Away = CALCULATE(SUM(FactClubMatch[GoalsFor]), FactClubMatch[ClubKey] = MAX(DimMatch[AwayClubKey]))

Is Rivalry Match =
VAR SelectedKeys = VALUES(DimClub[ClubKey])
VAR HomeKey = MAX(DimMatch[HomeClubKey])
VAR AwayKey = MAX(DimMatch[AwayClubKey])
RETURN
    COUNTROWS(SelectedKeys) = 2
    && CONTAINS(SelectedKeys, DimClub[ClubKey], HomeKey)
    && CONTAINS(SelectedKeys, DimClub[ClubKey], AwayKey)
    && HomeKey <> AwayKey
```

**Power BI — steps:**

1. **Table visual:** Source = **DimMatch**. Columns: e.g. **Matchday**, **DateKey** (or date from DimDate), **HomeClubName**, **AwayClubName**, **[Goals Home]**, **[Goals Away]**. Optional: Stadium, Attendance.
2. **Visual filter:** Filter on this visual → **Is Rivalry Match** = TRUE. When exactly 2 clubs are selected, only matches between those two clubs appear.
3. **Sort:** By **Matchday** or **Date** (ascending).

---

## Summary

| Page   | Visual                         | Type               | Purpose |
|--------|--------------------------------|--------------------|---------|
| **Rivalities**   | Table PairLabel                 | Table (selector)   | Select a rivalry. |
| **Rivalities**   | Avg attendance by rivalry       | Stacked bar chart  | Which rivalries are most attended. |
| **Rivalities**   | Avg fill rate by rivalry        | Stacked bar chart  | Which rivalries fill the stadium most. |
| **Rivalities(2)**| HTML cards (W–D–L, goals)       | HTML Content       | Head-to-head summary for 2 selected clubs. |
| **Rivalities(2)**| Sanctions per rivalry by Reason | Stacked bar chart  | Friction between teams (sanctions by reason). |
| **Rivalities(2)**| Match history                   | Table / timeline    | Chronological list of matches for 2 selected clubs. |

**Interactions:** Use the PairLabel table (page 1) to select a rivalry; ensure page 2 receives the same filter (or use a 2-club slicer on page 2) so that the HTML cards and match table show the correct pair.
