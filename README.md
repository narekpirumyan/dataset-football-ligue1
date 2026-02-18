# Ligue 1 — DataViz Project (Season 2023–2024)

Power BI dashboard project for **Ligue 1** using match results, player stats, transfers, stadium attendance, and disciplinary data.

## Contents

| Folder / File | Description |
|---------------|-------------|
| **Dataset/** | Source data: CSV, XLSX, TXT, PDF (match results, player stats, transfers, attendance, sanctions, clubs, finances). |
| **Docs/** | Specification and design: dashboard spec, star schema, ETL / Power Query & measures. |
| **FootDashboardV1.pbix** | Power BI report (5-page dashboard). |
| **PRESENTATION_EN.pdf** | Project presentation. |

## Documentation

- [0_POWER_BI_DASHBOARD_SPECIFICATION.md](Docs/0_POWER_BI_DASHBOARD_SPECIFICATION.md) — Dashboard concept, visuals, data mapping, quality checks  
- [1_POWER_BI_STAR_SCHEMA_DESIGN.md](Docs/1_POWER_BI_STAR_SCHEMA_DESIGN.md) — Star schema, dimensions, facts, relationships  
- [2_ETL_POWER_QUERY_AND_MEASURES.md](Docs/2_ETL_POWER_QUERY_AND_MEASURES.md) — Power Query steps and DAX measures  

## Data sources (summary)

- `match_results_2023_2024.csv` — Matchday, teams, scores, stadium, referee, attendance  
- `player_stats_season.csv` — Player stats (goals, assists, minutes, etc.)  
- `transfers_2023_2024.txt` — Transfers (departure/arrival club, type, amount)  
- `stadium_attendance.csv` — Attendance, capacity, fill rate  
- `disciplinary_sanctions.csv` — Sanctions, suspensions, fines  
- Additional XLSX/PDF in `Dataset/` for clubs, finances, standings, reports  

## License

Content in this repository is for educational / project use (IAE DataViz 2025).
