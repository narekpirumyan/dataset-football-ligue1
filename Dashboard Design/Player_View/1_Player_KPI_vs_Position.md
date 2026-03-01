# Player View — Player card: KPIs vs position average

**Purpose:** Display **9 gauge charts** that show the **sum** of each metric for the **selected player**, compared to a **reference line (target)**:
- **Default:** The reference is the **average of all players** (displayed as the target on the gauge).
- **With position filter:** If the user applies a **position filter** (e.g. selects "Forward"), the reference becomes the **average of players with the same position** as the selected player.

This allows comparing the player’s totals to the league average or to the average of their position (Forward, Central midfielder, etc.).

**Tables:** `DimPlayer` (PlayerKey, FullName, **Position**), `FactPlayerSeason` (PlayerKey, ClubKey, Goals, Assists, Shots, ShotsOnTarget, Saves, RedCards, SuccessfulDribbles, Interceptions, SuccessfulTackles, etc.). Reference schema: Ligue-Wide_View / 0_markdown.md.

---

## 1. Player selection

- **Slicer** on `DimPlayer[FullName]` (or `DimPlayer[PlayerKey]`) with **single selection**: the user selects one player.
- All visuals on the card are filtered by this slicer (context = one player).
- **Optional:** A **position filter** (slicer or filter on `DimPlayer[Position]`). When the user applies it, the gauge min, max, and target (reference line) use **same-position** averages and bounds instead of **all-players**.

---

## 2. The 9 gauge charts (metrics)

There are **9 gauge charts**, one per metric. Each gauge shows:

- **Value (needle):** The **sum** of that metric for the **selected player** over the filtered period (e.g. season).
- **Target (reference line):** The **average** of that metric — either over **all players** or over **players with the same position** if the position filter is applied. This average is displayed as the “target” on the gauge for visual comparison (it is a reference, not a goal to reach).
- **Min / Max (scale):** The scale bounds — either **global** (min/max over all players) or **same position** (min/max over players with the same position), consistent with the target.

| # | Metric           | FactPlayerSeason field   | Gauge value (player) | Target (reference)      |
|---|------------------|---------------------------|-----------------------|--------------------------|
| 1 | Saves            | Saves                     | SUM (selected player) | Avg all / same position  |
| 2 | Interceptions    | Interceptions             | SUM (selected player) | Avg all / same position  |
| 3 | Shots            | Shots                     | SUM (selected player) | Avg all / same position  |
| 4 | Shots on target  | ShotsOnTarget             | SUM (selected player) | Avg all / same position  |
| 5 | Goals            | Goals                     | SUM (selected player) | Avg all / same position  |
| 6 | Assists          | Assists                   | SUM (selected player) | Avg all / same position  |
| 7 | Dribbles         | SuccessfulDribbles        | SUM (selected player) | Avg all / same position  |
| 8 | Tackles          | SuccessfulTackles         | SUM (selected player) | Avg all / same position  |
| 9 | Red cards        | RedCards                  | SUM (selected player) | Avg all / same position  |

---

## 3. DAX measures

For each of the 9 metrics, you need:

1. **Player value (gauge value):** Sum of the metric for the selected player.
2. **Average (target / reference line):** Average over all players, or over same-position players when the position filter is applied.
3. **Min / Max (gauge scale):** Global or same-position, depending on the same logic.

**Player value (context = 1 player selected)** — use as the gauge **Value**:

```dax
Player Saves     = SUM(FactPlayerSeason[Saves])
Player Interceptions = SUM(FactPlayerSeason[Interceptions])
Player Shots     = SUM(FactPlayerSeason[Shots])
Player ShotsOnTarget = SUM(FactPlayerSeason[ShotsOnTarget])
Player Goals     = SUM(FactPlayerSeason[Goals])
Player Assists   = SUM(FactPlayerSeason[Assists])
Player Dribbles  = SUM(FactPlayerSeason[SuccessfulDribbles])
Player Tackles   = SUM(FactPlayerSeason[SuccessfulTackles])
Player RedCards  = SUM(FactPlayerSeason[RedCards])
```

**Average — all players (target when no position filter):**

```dax
Avg Saves All = CALCULATE(AVERAGE(FactPlayerSeason[Saves]), ALL(DimPlayer))
Avg Interceptions All = CALCULATE(AVERAGE(FactPlayerSeason[Interceptions]), ALL(DimPlayer))
// … same pattern for Shots, ShotsOnTarget, Goals, Assists, SuccessfulDribbles, SuccessfulTackles, RedCards
```

**Average — same position (target when position filter is applied):**

```dax
Avg Goals Same Position =
VAR CurrentPosition = SELECTEDVALUE(DimPlayer[Position])
RETURN
    IF(
        ISBLANK(CurrentPosition),
        BLANK(),
        CALCULATE(
            AVERAGE(FactPlayerSeason[Goals]),
            FILTER(DimPlayer, DimPlayer[Position] = CurrentPosition)
        )
    )
```

Repeat the same pattern for each of the 9 metrics (Saves, Interceptions, Shots, ShotsOnTarget, Goals, Assists, SuccessfulDribbles, SuccessfulTackles, RedCards). You can use one parameter or two measures (Avg … All vs Avg … Same Position) and choose the target in the gauge based on whether a position filter is active.

**Min / Max for gauge scale:**

- **Global:** e.g. `Min Goals All = CALCULATE(MIN(FactPlayerSeason[Goals]), ALL(DimPlayer))`, `Max Goals All = CALCULATE(MAX(FactPlayerSeason[Goals]), ALL(DimPlayer))`.
- **Same position:** e.g. `Min Goals Same Position = CALCULATE(MIN(FactPlayerSeason[Goals]), FILTER(DimPlayer, DimPlayer[Position] = SELECTEDVALUE(DimPlayer[Position])))`, and similarly for Max.

Use the same logic for all 9 metrics so that when the user applies the position filter, the gauge scale and target switch to same-position min/max and average.

---

## 4. Gauge chart configuration (per metric)

For **each of the 9 gauges**:

| Gauge element   | Role |
|-----------------|------|
| **Value**       | Player measure (sum for selected player): e.g. `[Player Goals]`, `[Player Saves]`, etc. |
| **Target**      | Average used as reference line: **average of all players** by default, or **average of same position** when the position filter is applied. |
| **Min**         | Min of the metric — global or same position (same choice as target). |
| **Max**         | Max of the metric — global or same position (same choice as target). |

The target is shown as a **constant line / goal** on the gauge so users can compare the player’s value to the average at a glance. It is a **reference**, not a mandatory target.

---

## 5. Power BI layout

- **9 gauge visuals**, one per metric: Saves, Interceptions, Shots, Shots on target, Goals, Assists, Dribbles, Tackles, Red cards.
- **Value** of each gauge = the corresponding “Player …” measure (sum for the selected player).
- **Target** of each gauge = the average measure (all players or same position, depending on the position filter).
- **Min / Max** of each gauge = the chosen min/max measures (global or same position).
- **Slicer:** `DimPlayer[FullName]` (single selection) at the top or side; it filters the whole page.
- **Position filter (optional):** When the user applies a filter on `DimPlayer[Position]`, all 9 gauges use same-position average as target and same-position min/max for the scale.

---

## 6. Summary

| Metric           | Gauge value (player) | Target (reference)     |
|------------------|----------------------|------------------------|
| Saves            | SUM(Saves)           | Avg all / same position |
| Interceptions    | SUM(Interceptions)   | Avg all / same position |
| Shots            | SUM(Shots)           | Avg all / same position |
| Shots on target  | SUM(ShotsOnTarget)   | Avg all / same position |
| Goals            | SUM(Goals)           | Avg all / same position |
| Assists          | SUM(Assists)         | Avg all / same position |
| Dribbles         | SUM(SuccessfulDribbles) | Avg all / same position |
| Tackles          | SUM(SuccessfulTackles)  | Avg all / same position |
| Red cards        | SUM(RedCards)        | Avg all / same position |

- **Default:** Target = average of **all players**; min/max = global.
- **With position filter:** Target = average of **players with the same position**; min/max = same position.

All text and behaviour are described in English and aligned with the 9 gauge charts listed above.
