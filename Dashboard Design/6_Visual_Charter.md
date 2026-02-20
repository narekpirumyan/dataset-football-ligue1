# Visual Charter

## Club colors

Each **ClubName** must have a single, consistent color across all report pages and visuals (charts, maps, tables, KPI cards, etc.).

### Principle

- One dedicated color per club.
- Use the same color for that club everywhere: in legends, series, conditional formatting, and slicers (if styled by club).
- Ensures immediate recognition of a club in any visual.

### Source of club list and colors (JSON)

- **Club list:** The list of clubs is taken from **`Dataset/clubs.xlsx`** (column **`club_name`**). Use this file as the single source of truth for club names so that the color mapping stays aligned with the model.
- **Color mapping:** Club colors are defined in **`Dashboard Design/club_colors.json`**. The JSON is an array of objects with **`club_name`** and **`color`** (hex, e.g. `#004D9A`). One entry per club; when adding or renaming clubs, update the JSON and refresh the report.

### Implementation (Power BI)

1. **Load the Club Colors table from JSON**  
   - In Power Query: **Get data → From File → From JSON**. Select **`club_colors.json`** (or the path where you placed it).  
   - The JSON array becomes a table with columns **`club_name`** and **`color`**. Rename **`club_name`** to **ClubName** (or keep as-is and use in “Format by field value”) so it matches the field used in your visuals.  
   - Load this table into the model. Do **not** create a relationship to fact tables if you only use it for formatting; use it as the mapping source for “Format by field value”.

2. **Use the color in visuals**  
   - **Charts (bar, line, column, etc.):** Format the data series or category by field value: choose the Club Colors table, **ClubName** as field, **Color** as values.  
   - **Conditional formatting:** For tables/matrices, use “Format by field value” and select the Club Colors table, **ClubName**, **Color**.  
   - **Theme (optional):** If you fix the palette order, you can rely on theme colors and ensure the same data series/category always uses the same theme index; a dedicated table is more robust and easier to maintain.

3. **Palette**  
   - Define a set of distinct hex colors (e.g. 20 for 20 clubs).  
   - Assign each ClubName to one hex in the Club Colors table.  
   - Prefer colors that are distinguishable for all users (e.g. avoid very light or very similar hues).

4. **Consistency**  
   - Keep **`club_colors.json`** in sync with **`Dataset/clubs.xlsx`**: same **club_name** values (spelling and casing). When adding or renaming a club in the data, add or update the corresponding entry in the JSON, assign a new distinct hex color, then refresh the report so all visuals keep the same mapping.
