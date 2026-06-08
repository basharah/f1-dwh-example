# F1 Data Warehouse — OLAP Demo

A self-contained PostgreSQL + pgAdmin environment for demonstrating OLAP queries. The database is pre-loaded with Formula 1 race data.

## Requirements

- [Docker](https://www.docker.com/get-started) with Docker Compose

## Quick Start

```bash
docker compose up -d
```

Then open **http://localhost:5050** in a browser.

pgAdmin opens directly — no login required. Expand **Servers → F1 Data Warehouse** in the left panel, right-click → **Query Tool**, and enter the database password when prompted:

| Field    | Value    |
|----------|----------|
| Username | `f1user` |
| Password | `f1pass` |

## Database Schema

13 tables covering F1 seasons from 2010 onwards:

| Table | Description |
|---|---|
| `circuits` | Race circuits with location and coordinates |
| `seasons` | Championship years |
| `races` | Individual race events per season |
| `drivers` | Driver profiles |
| `constructors` | Constructor (team) profiles |
| `results` | Race finishing results per driver |
| `driverstandings` | Driver championship standings after each race |
| `constructorstandings` | Constructor championship standings after each race |
| `constructorresults` | Constructor points per race |
| `qualifying` | Qualifying session times (Q1/Q2/Q3) |
| `pitstops` | Pit stop times per driver per race |
| `sprintresults` | Sprint race results |
| `status` | Lookup table for race finish statuses |

## Connection Details

| | |
|---|---|
| Host | `localhost` |
| Port | `5432` |
| Database | `f1_dw` |
| Username | `f1user` |
| Password | `f1pass` |

These credentials work for any external SQL client (DBeaver, DataGrip, psql, etc.).

## Lifecycle Commands

```bash
# Start
docker compose up -d

# Stop (data is preserved)
docker compose down

# Full reset — wipes all data and re-initialises from SQL files
docker compose down -v && docker compose up -d
```

## Sample OLAP Queries

### 1. Running Points Total per Driver (Window Function)

Cumulative championship points after each race for a given season.

```sql
SELECT
    r.year, r.round, r.name AS race,
    d.forename || ' ' || d.surname AS driver,
    res.points,
    SUM(res.points) OVER (
        PARTITION BY r.year, res.driverId
        ORDER BY r.round
    ) AS cumulative_points
FROM results res
JOIN races r ON res.raceId = r.raceId
JOIN drivers d ON res.driverId = d.driverId
WHERE r.year = 2023
ORDER BY r.round, cumulative_points DESC;
```

### 2. Top 3 Drivers by Wins per Season (RANK)

Ranks drivers by race wins within each season and keeps only the podium.

```sql
SELECT year, driver, total_wins, rnk
FROM (
    SELECT
        r.year,
        d.forename || ' ' || d.surname AS driver,
        COUNT(*) FILTER (WHERE res.position = 1) AS total_wins,
        RANK() OVER (
            PARTITION BY r.year
            ORDER BY COUNT(*) FILTER (WHERE res.position = 1) DESC
        ) AS rnk
    FROM results res
    JOIN races r ON res.raceId = r.raceId
    JOIN drivers d ON res.driverId = d.driverId
    GROUP BY r.year, d.driverId, d.forename, d.surname
) t
WHERE rnk <= 3
ORDER BY year, rnk;
```

### 3. Constructor Points Year-over-Year (LAG)

Compares each constructor's total season points against the previous year.

```sql
SELECT
    c.name AS constructor,
    year,
    total_points,
    LAG(total_points) OVER (PARTITION BY c.name ORDER BY year) AS prev_year_points,
    total_points - LAG(total_points) OVER (PARTITION BY c.name ORDER BY year) AS diff
FROM (
    SELECT res.constructorId, r.year, SUM(res.points) AS total_points
    FROM results res
    JOIN races r ON res.raceId = r.raceId
    GROUP BY res.constructorId, r.year
) t
JOIN constructors c ON t.constructorId = c.constructorId
ORDER BY c.name, year;
```

### 4. Points by Year and Constructor with Subtotals (ROLLUP)

Aggregates points at three levels: per constructor per year, per year total, and grand total.

```sql
SELECT
    COALESCE(r.year::TEXT, 'ALL') AS year,
    COALESCE(c.name, 'ALL')       AS constructor,
    SUM(res.points)               AS total_points
FROM results res
JOIN races r ON res.raceId = r.raceId
JOIN constructors c ON res.constructorId = c.constructorId
GROUP BY ROLLUP(r.year, c.name)
ORDER BY r.year NULLS LAST, total_points DESC;
```

### 5. Average Positions Gained from Qualifying to Race Finish

Shows which drivers consistently outperform their qualifying position on race day.

```sql
SELECT
    r.year,
    d.forename || ' ' || d.surname        AS driver,
    ROUND(AVG(q.position)::numeric, 1)    AS avg_qualifying,
    ROUND(AVG(res.positionOrder)::numeric, 1) AS avg_finish,
    ROUND(AVG(q.position - res.positionOrder)::numeric, 1) AS avg_positions_gained
FROM qualifying q
JOIN results res ON q.raceId = res.raceId AND q.driverId = res.driverId
JOIN races r     ON q.raceId = r.raceId
JOIN drivers d   ON q.driverId = d.driverId
WHERE q.position IS NOT NULL
GROUP BY r.year, d.driverId, d.forename, d.surname
HAVING COUNT(*) >= 5
ORDER BY avg_positions_gained DESC;
```

## Project Structure

```
.
├── ddl.sql              # Table definitions
├── data.sql             # Seed data
├── docker-compose.yml
└── pgadmin/
    └── servers.json     # Pre-configured pgAdmin connection
```
