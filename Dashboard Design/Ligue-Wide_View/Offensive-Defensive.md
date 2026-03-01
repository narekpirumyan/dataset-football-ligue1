# Ligue-Wide View — Offensive / Defensive

**Category:** Attack and defence metrics at league level (all clubs).

**Data source:** Power BI data model — `FactClubMatch`, `DimClub`, `DimMatch` (sporting); `FactSanction`, `DimClub` (discipline).

---

## A. Attack–Defence map (efficiency frontier)

**Visual type:** Scatter chart

**Purpose:** Compare clubs by attacking output and defensive solidity to identify profiles:
- **Dominant:** high goals for, low goals against (top-right if Y = “fewer is better”, or top-left if Y inverted).
- **Fragile:** high goals for but high goals against (attack-minded, leaky).
- **Pragmatic:** low goals for, low goals against (tight, low-scoring).
- **Weak:** low goals for, high goals against.

**Axes and size:**

| Role   | Field / measure                         | Notes |
|--------|----------------------------------------|-------|
| **X-axis** | Goals for per match                    | Average goals scored per match (GoalsFor / Match). |
| **Y-axis** | Goals against per match (inverted)     | Average goals conceded per match. Invert axis or use a “defence score” so **high = better** (e.g. Y = −GoalsAgainst/Match, or invert the Y-axis in the visual so top = fewer goals against). |
| **Size**   | Points                                 | Bubble size = total points (or points per match). |
| **Legend / category** | Club (DimClub[ClubName])     | One point per club; colours from report theme palette. |

**DAX measures (league-wide, one value per club):**

```dax
Matches Played = COUNTROWS(FactClubMatch)

Goals For = SUM(FactClubMatch[GoalsFor])
Goals Against = SUM(FactClubMatch[GoalsAgainst])

Goals For per Match = DIVIDE([Goals For], [Matches Played], BLANK())
Goals Against per Match = DIVIDE([Goals Against], [Matches Played], BLANK())

Total Points = SUM(FactClubMatch[Points])
```

---

### Quadrant divider (cross at centre)

**Option 2 — Image (cross)**

1. Use the provided image **`quadrant_cross.jpeg`** (recommended: JPEG, white background, black cross, thick stroke) or **`quadrant_cross.png`** (transparent background, black cross). Both use a black, thick cross for visibility.
2. **Insert → Image** and load `quadrant_cross.jpeg` (often works better in Power BI than PNG). Resize and position it so the centre of the cross matches the desired centre of the scatter (use axis labels to align).
3. **Send to back** for the image; **Bring to front** for the scatter so points are visible above the cross.
4. **Important — make the scatter background transparent:** Select the scatter chart → **Format** (paint roller) → **General** → **Effects** → **Background**. Set **Transparency** to 100% (or “Aucune” / no fill) so the image behind shows through. Without this, the scatter’s opaque background hides the image.

**Tips**

- **Reference values:** Use a card or table with two measures (e.g. median X, median Y) on the same page to know where the “centre” is, then align the lines or image to those values on the axes.
- **Lock:** After alignment, you can lock the shapes (Format shape → Lock) to avoid moving them by mistake.
- If you change filters or data and the axis scale changes, realign the lines or image so the cross stays at the intended centre.

---

**Interpretation:** Top-left = few goals conceded (strong defence); right = many goals scored (strong attack). Dominant clubs sit top-right (attack + defence); fragile clubs right and down; pragmatic clubs left and up; weak clubs bottom-left.

