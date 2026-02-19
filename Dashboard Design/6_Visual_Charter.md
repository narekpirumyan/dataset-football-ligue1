# Visual Charter

## Club colors

Each **ClubName** must have a single, consistent color across all report pages and visuals (charts, maps, tables, KPI cards, etc.).

### Principle

- One dedicated color per club.
- Use the same color for that club everywhere: in legends, series, conditional formatting, and slicers (if styled by club).
- Ensures immediate recognition of a club in any visual.

### Implementation (Power BI)

1. **Create a Club Colors table**  
   - Table with two columns: **ClubName** (or ClubKey) and **Color** (e.g. hex: `#1E88E5`).  
   - One row per club; assign a hex code per club.  
   - Load in the data model (e.g. from Excel/CSV or a calculated table).  
   - Do **not** link it to fact tables with a relationship if you only use it for formatting; use it as a mapping source.

2. **Use the color in visuals**  
   - **Charts (bar, line, column, etc.):** Format the data series or category by field value: choose the Club Colors table, **ClubName** as field, **Color** as values.  
   - **Conditional formatting:** For tables/matrices, use “Format by field value” and select the Club Colors table, **ClubName**, **Color**.  
   - **Theme (optional):** If you fix the palette order, you can rely on theme colors and ensure the same data series/category always uses the same theme index; a dedicated table is more robust and easier to maintain.

3. **Palette**  
   - Define a set of distinct hex colors (e.g. 20 for 20 clubs).  
   - Assign each ClubName to one hex in the Club Colors table.  
   - Prefer colors that are distinguishable for all users (e.g. avoid very light or very similar hues).

4. **Consistency**  
   - When adding a new club or renaming, update the Club Colors table and refresh so all visuals keep the same mapping.
