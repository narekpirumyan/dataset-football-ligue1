# Ligue-Wide View — Offensive / Defensive

**Category:** Attack and defence metrics at league level (all clubs).

**Data source:** Power BI data model — `FactClubMatch`, `DimClub`, `DimMatch`.

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
| **Legend / category** | Club (DimClub[ClubName])     | One point per club; use **club_colors.json** for consistent colours. |

**DAX measures (league-wide, one value per club):**

```dax
Matches Played = COUNTROWS(FactClubMatch)

Goals For = SUM(FactClubMatch[GoalsFor])
Goals Against = SUM(FactClubMatch[GoalsAgainst])

Goals For per Match = DIVIDE([Goals For], [Matches Played], BLANK())
Goals Against per Match = DIVIDE([Goals Against], [Matches Played], BLANK())

Total Points = SUM(FactClubMatch[Points])
```

**Optional (for “high = better” on Y without inverting axis):**

```dax
Defence Score = - [Goals Against per Match]
```
Use `[Defence Score]` as Y so that higher = fewer goals conceded. Otherwise use `[Goals Against per Match]` as Y and **invert the Y-axis** in the scatter format.

**Power BI setup:**

1. Add a **Scatter chart** visual.
2. **X-axis:** `[Goals For per Match]`.
3. **Y-axis:** `[Goals Against per Match]` (then Format → Y-axis → Reverse scale) **or** `[Defence Score]`.
4. **Size:** `[Total Points]`.
5. **Legend / Details:** `DimClub[ClubName]` (one point per club).
6. Apply the **Visual Charter** club colours (Format by field value from the Club Colors table loaded from `club_colors.json`).
7. Add tooltips: ClubName, Goals For per Match, Goals Against per Match, Total Points (and optionally Goals For, Goals Against).

---

### Quadrant divider (cross at centre)

Power BI scatter charts do not support reference lines. To divide the chart into four quadrants with a cross, use **shapes** or an **image** placed behind the scatter and aligned manually.

**Option 1 — Two line shapes (recommended)**

1. **Add the scatter chart** and fix its size and position on the report page. Do not resize it after adding the lines, or you will need to realign.
2. **Insert → Formes (Shapes) → Ligne** (or the line tool). Draw a **vertical line** long enough to span the scatter height. In the Format pane: set a light colour (e.g. grey), thin stroke; set **Arrière-plan (Send to back)** so the scatter stays on top.
3. Insert a second shape: **Ligne** again, and draw a **horizontal line** spanning the scatter width. Same formatting and send to back.
4. **Align to centre (manual):**
   - Use the **median** (or average) of your data as the crossing point so the quadrants are balanced. For example, note the approximate X value (e.g. median Goals for per match ≈ 1.2) and Y value (e.g. median Goals against per match ≈ 1.2) from your axes.
   - Move the **vertical line** so it passes through that X position on the scatter (use the axis labels as a guide).
   - Move the **horizontal line** so it passes through that Y position. The two lines should cross at the centre of the data.
5. **Group (optional):** Select both lines → right‑click → **Grouper (Group)** so they move together if you reposition the scatter later.
6. Place the **scatter visual on top**: right‑click the scatter → **Mettre au premier plan (Bring to front)** so the bubbles are always above the lines.
7. **Make the scatter background transparent:** Select the scatter → **Format** → **General** → **Effects** → **Background** → Transparency 100% (or no fill) so the lines behind are visible.

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
