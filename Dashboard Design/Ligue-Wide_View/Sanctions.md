# Ligue-Wide View — Sanctions

**Purpose:** Explore what factors may influence players receiving sanctions (e.g. competition progress via matchday, player position regarding cards) and understand each club’s overall behaviour regarding sanctions.

**Data source:** Power BI data model — `FactSanction`, `DimClub`, `DimPlayer` (Position), `DimMatch`, `DimDate`.

---

## Scope — selected reasons (optional)

If you count only certain sanction reasons (e.g. exclude “Controversial social media post”, “Late for anti-doping control”), use a measure that filters on `FactSanction[Reason]`. The visuals below can use either **total sanctions** or **Sanction Count (selected reasons)**; the exact list of reasons is defined in the model.

**Example base measure:**

```dax
Sanction Count (selected reasons) =
CALCULATE(
    COUNTROWS(FactSanction),
    FILTER(
        FactSanction,
        FactSanction[Reason] IN { "Abusive language in mixed zone", "Accumulation of yellow cards",
            "Deliberate elbow strike", "Inappropriate gesture towards an opponent",
            "Insulting the referee", "On-field brawl", "Red card - last man foul",
            "Repeated protesting", "Simulation", "Spitting",
            "Straight red card - dangerous tackle", "Unsportsmanlike conduct" }
    )
)
```

Use this measure (or a simple `COUNTROWS(FactSanction)`) in the visuals as needed.

---

## 1. Bar chart — Red cards by club (Position as legend)

**Visual type:** Bar chart

**Purpose:** Show **red cards** per club, broken down by **player position** (Position). Helps explore whether certain positions (e.g. Defender, Midfielder) are more associated with red cards and how each club behaves in terms of red cards by position.

| Role      | Field / measure                    | Notes |
|-----------|------------------------------------|-------|
| **Axis (category)** | `DimClub[ClubName]`         | One bar per club (or Y-axis for horizontal bars). |
| **Value** | Red card count                    | Count of red-card sanctions per club (e.g. filter FactSanction by SanctionType or Reason for red cards). |
| **Legend**| `DimPlayer[Position]`             | Segment or series by **player position** (poste). Requires linking sanctions to players (FactSanction[PlayerKey] → DimPlayer). |

**Data:** `FactSanction` (ClubKey, PlayerKey, Reason/SanctionType), `DimClub`, `DimPlayer` (Position). If red cards are identified by Reason (e.g. “Straight red card - dangerous tackle”, “Red card - last man foul”) or by SanctionType, use a measure that counts only those rows per club and, for the legend, use `DimPlayer[Position]` so that each bar is split or coloured by position.

**Power BI:** Bar chart → **Axis** = DimClub[ClubName], **Values** = measure (e.g. count of red-card sanctions per club), **Legend** = DimPlayer[Position]. Sort by value descending if needed.

---

## 2. Bar chart — Total sanctions: home vs away (all clubs)

**Visual type:** Bar chart

**Purpose:** Compare **total sanctions at home** vs **away** (league-wide, all clubs). Shows whether sanctions are more frequent in home or away matches overall.

| Role      | Field / measure                    | Notes |
|-----------|------------------------------------|-------|
| **Category** | Two categories: Home, Away       | E.g. via a small table or “Measure names” with two measures. |
| **Value** | Home sanction count, Away sanction count | One bar for total home sanctions, one for total away sanctions. |

**Data:** Each sanction is attributed to the **last match of that club on or before the sanction date**; then home/away is taken from that match (DimMatch: HomeClubKey, AwayClubKey). See DAX below.

**DAX (example — adapt Reason filter to your model):**

```dax
Sanctions Home =
SUMX(
    FILTER(FactSanction, FactSanction[Reason] IN { "…", "…" }),  // your selected reasons or remove filter for total
    VAR LastMatchDate =
        CALCULATE(
            MAX(DimMatch[DateKey]),
            FILTER(
                DimMatch,
                DimMatch[DateKey] <= FactSanction[SanctionDateKey]
                    && (DimMatch[HomeClubKey] = FactSanction[ClubKey]
                        || DimMatch[AwayClubKey] = FactSanction[ClubKey])
            )
        )
    VAR IsHome =
        NOT ISBLANK(LastMatchDate)
        && COUNTROWS(
            FILTER(DimMatch,
                DimMatch[DateKey] = LastMatchDate
                    && DimMatch[HomeClubKey] = FactSanction[ClubKey])
        ) > 0
    RETURN IF(IsHome, 1, 0)
)

Sanctions Away =
SUMX(
    FILTER(FactSanction, FactSanction[Reason] IN { "…", "…" }),
    VAR LastMatchDate = …
    VAR IsAway = …  // same logic with AwayClubKey
    RETURN IF(IsAway, 1, 0)
)
```

**Power BI:** Bar chart → **Category** = use a table with "Home" and "Away" or **Measure names** with **[Sanctions Home]** and **[Sanctions Away]**; **Values** = those two measures. One bar for home total, one for away total (all clubs).

---

## 3. Line chart — Cumulative sanctions by matchday

**Visual type:** Line chart

**Purpose:** Show the **cumulative** number of sanctions over the season (X = matchday 1–38, Y = total sanctions up to that matchday). Highlights how discipline incidents accumulate as the competition progresses (e.g. tension rising with matchday).

| Role      | Field / measure                    | Notes |
|-----------|------------------------------------|-------|
| **X-axis**| `DimMatch[Matchday]`              | One point per matchday (1–38). |
| **Y-axis**| Cumulative sanction count          | Sanctions with SanctionDateKey ≤ end of that matchday. |

**DAX (example):**

```dax
Cumulative Sanction Count =
VAR MatchdayVal = MAX(DimMatch[Matchday])
VAR EndDate =
    CALCULATE(
        MAX(DimMatch[DateKey]),
        ALL(DimMatch),
        DimMatch[Matchday] = MatchdayVal
    )
RETURN
    CALCULATE(
        [Sanction Count (selected reasons)],   // or COUNTROWS(FactSanction)
        FactSanction[SanctionDateKey] <= EndDate,
        ALL(FactSanction[SanctionDateKey])
    )
```

**Power BI:** Line chart → **X-axis** = DimMatch[Matchday], **Y-axis** = [Cumulative Sanction Count]. Sort matchday ascending. Optional: **Legend** = DimClub[ClubName] for one line per club.

---

## 4. Stacked bar chart — Sanctions by club and reason

**Visual type:** Stacked bar chart

**Purpose:** Show **total sanctions per club** with a **breakdown by reason** (legend). Compare discipline across clubs and see which reasons (e.g. red cards, simulation, unsportsmanlike conduct) dominate per club.

| Role      | Field / measure                    | Notes |
|-----------|------------------------------------|-------|
| **Axis (category)** | `DimClub[ClubName]`         | One bar per club. |
| **Value** | Sanction count                    | Total sanctions per club (and per reason for the stack). |
| **Legend**| `FactSanction[Reason]`            | Stack segments by sanction reason. |

**Power BI:** Stacked bar chart → **Axis** = DimClub[ClubName], **Values** = sanction count measure (e.g. [Sanction Count (selected reasons)] or COUNTROWS(FactSanction)), **Legend** = FactSanction[Reason]. Sort by total descending if needed. Tooltips: ClubName, Reason, count.

---

## 5. Donut chart — Distribution of sanctions by reason

**Visual type:** Donut chart

**Purpose:** Show the **distribution** of the number of sanctions **by reason** (one segment per reason). Highlights which types of incidents (e.g. accumulation of yellows, red cards, simulation) are most frequent overall.

| Role      | Field / measure                    | Notes |
|-----------|------------------------------------|-------|
| **Legend (or Axis)** | `FactSanction[Reason]`     | One segment per reason. |
| **Values**| Sanction count                    | Number of sanctions per reason. |

**Power BI:** Donut chart → **Legend** (or **Axis**) = FactSanction[Reason], **Values** = sanction count measure. Optional: sort by value descending, tooltips with reason and count (and % of total). Center label: total count or “Sanctions by reason”.

---

## Summary

| # | Visual                    | Type               | Purpose |
|---|---------------------------|--------------------|---------|
| 1 | Red cards by club         | Bar chart          | Red cards per club, **Position** as legend (player position vs red cards). |
| 2 | Home vs away              | Bar chart          | Total sanctions **home vs away** (all clubs). |
| 3 | Cumulative by matchday    | Line chart         | **Cumulative** sanctions by matchday (competition progress). |
| 4 | Sanctions by club & reason| Stacked bar chart  | Sanctions by **club**, stacked by **reason**. |
| 5 | Distribution by reason    | Donut chart        | **Distribution** of sanction count **by reason**. |

Together, these visuals support exploring factors that may influence sanctions (matchday, position, home/away) and each club’s overall sanction behaviour.
