# Investigating Positional Adjustments in Baseball

This project investigates whether centerfielders (CF) really perform better when playing in the corners (RF/LF), and if this performance difference aligns with the positional adjustments in WAR (Wins Above Replacement).

## Data Collection

The fielding statistics for players across the last 20 seasons (2005-2024) were fetched using the `pybaseball` library and combined into a single DataFrame. This data includes the fielding metrics of UZR and DRS. I declined to include OAA because data only goes back to 2016 and doesn't include arm strength. 

The primary dataset is stored in the file `fielding_stats_last_20_seasons.csv`, which includes all player fielding stats for the seasons 2005 through 2024.

## Methodology

We will explore the following:
- **Positional Adjustments**: Do CF players perform better when playing in the corner outfield positions (RF/LF)?
- **WAR Adjustments**: Are the differences in performance in line with WAR positional adjustments?

To perform the analysis, we will import the dataset into PostgreSQL and use SQL queries to:
1. Aggregate the fielding data by player and position.
2. Analyze the performance differences between CF and corner outfield positions.
3. Compare the observed performance differences to the expected positional adjustments in WAR.

## How to Use

To replicate the analysis:
1. Import the `fielding_stats_last_20_seasons.csv` into the PostgreSQL database.
2. Use SQL queries to perform the analysis.
   - Load the CSV data into a PostgreSQL table.
   - Aggregate stats like DRS and UZR for centerfielders and compare with the corner outfield positions.
   - Filter data based on innings played, and calculate performance differences.

## SQL Queries for Analysis
```sql
-- Create table to store fielding stats
CREATE TABLE fielding_stats (
    player_id VARCHAR(20) NOT NULL,
    player_name TEXT NOT NULL,
    season INT,
    team VARCHAR(10),
    pos VARCHAR(5),
    inn FLOAT,
    drs INT,
    uzr FLOAT
);

-- Insert data into the table (assuming CSV data is imported)
COPY fielding_stats FROM '/path/to/fielding_stats_last_20_seasons.csv' DELIMITER ',' CSV HEADER;

-- Initial query to analyze DRS and UZR by position, filtering to qualified players (>2/3 of a season's innings) and excluding P and C because no UZR data exists for these positions
SELECT 
	pos, 
	ROUND(AVG(drs),2) AS avg_drs,
	ROUND(AVG(uzr)::numeric, 2) AS avg_uzr
FROM fielding_stats
WHERE inn > 900
AND pos NOT IN ('C', 'P')
GROUP BY pos;
```
![image](https://github.com/user-attachments/assets/79cd9183-8cf2-49bd-b000-20e9556d73bf)

This table makes it clear that whether we're looking at positionally-adjusted metrics (DRS) or raw defensive data (UZR), the tougher defensive positions already tend to have the better defenders. But we're interested in outfielders, specifically outfielders that have spent significant time in both CF and in one or both of the corner spots. 

How many players over this 20-year period have at least a season's worth of innings (1458 innings) in both?

```
SELECT COUNT(*) 
FROM (
    SELECT player_name
    FROM fielding_stats
    WHERE pos IN ('CF', 'LF', 'RF')
    GROUP BY player_name
    HAVING 
        SUM(CASE WHEN pos = 'CF' THEN inn ELSE 0 END) >= 1458
        AND SUM(CASE WHEN pos IN ('LF', 'RF') THEN inn ELSE 0 END) >= 1458
) AS qualified_players;

```
![image](https://github.com/user-attachments/assets/dbeb4fcf-5f1b-4392-9d60-095355991b55)

There are 72 total players that have enough innings to qualify for both conditions, but let's do a sanity check on the data to make sure that they do.

```
SELECT player_name, 
       ROUND(SUM(CASE WHEN pos = 'CF' THEN inn ELSE 0 END)::numeric,2) AS innings_played_cf,
       ROUND(SUM(CASE WHEN pos IN ('LF', 'RF') THEN inn ELSE 0 END)::numeric,2) AS innings_played_corner
FROM fielding_stats
WHERE player_name IN (
    SELECT player_name
    FROM fielding_stats
    WHERE pos IN ('CF', 'LF', 'RF')
    GROUP BY player_name
    HAVING 
        SUM(CASE WHEN pos = 'CF' THEN inn ELSE 0 END) >= 1458
        AND SUM(CASE WHEN pos IN ('LF', 'RF') THEN inn ELSE 0 END) >= 1458
)
GROUP BY player_name
ORDER BY innings_played_cf;
```

![image](https://github.com/user-attachments/assets/a00185c8-d285-40ba-a12e-70dee49bbd25)

The lowest number of innings in CF here is above the threshold, and repeating the query ordering by corner outfield innings shows the same. We're ready to analyze the defensive data now.


-- Compare CF to RF/LF (corner positions) performance
SELECT player_name, season, SUM(CASE WHEN position = 'CF' THEN UZR ELSE 0 END) AS cf_UZR,
       SUM(CASE WHEN position IN ('LF', 'RF') THEN UZR ELSE 0 END) AS corner_UZR
FROM fielding_stats
GROUP BY player_name, season
HAVING SUM(innings_played) > 1500;
