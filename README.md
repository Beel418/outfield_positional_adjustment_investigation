# Investigating Positional Adjustments in Baseball

This project investigates whether centerfielders (CF) really perform better when playing in the corners (RF/LF), and if this performance difference aligns with the positional adjustments in WAR (Wins Above Replacement). When calculating WAR for defense, both Baseball Reference (using the stat DRS) and Fangraphs (using UZR) factor in positional adjustments to account for the difficulty of the defensive position, as a player who can hold down the fort at shortstop is much more valuable than a defensive stalwart at first base. However, some have criticised these adjustments as being too extreme, especially as fewer balls are being put into play over time as the percentage of plays that end in either a strike-out, walk, or homerun (all plays that don't involve the defense) has increased over time. I'm going to do an investigation into the defensive data to see if these criticisms are valid. Given that the three outfield positions are relatively similar in responsibility (that of the chase down flyballs and keep advancing runners at bay variety), I will be looking into how performance changes when moving from a centerfielder to a corner outfielder position (and vice-versa).

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

-- Initial query to analyze DRS and UZR by position, filtering to qualified players (900 innings)
and excluding P and C because no UZR data exists for these positions.
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

This table makes it clear that  the tougher defensive positions already tend to have the better defenders. But we're interested in outfielders, specifically outfielders that have spent significant time in both CF and in one or both of the corner spots. 

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

First let's add in DRS and UZR data and save it to a view.

```
CREATE VIEW qualified_cf_corner_outfielders AS
SELECT player_name, 
	   ROUND(SUM(CASE WHEN pos = 'CF' THEN inn ELSE 0 END)::numeric,2) AS innings_played_cf,
	   SUM(CASE WHEN pos = 'CF' THEN drs ELSE 0 END) AS drs_cf,
	   ROUND(SUM(CASE WHEN pos = 'CF' THEN uzr ELSE 0 END)::numeric,2) AS uzr_cf,
           ROUND(SUM(CASE WHEN pos IN ('LF', 'RF') THEN inn ELSE 0 END)::numeric,2) AS innings_played_corner,
	   SUM(CASE WHEN pos IN ('LF', 'RF') THEN drs ELSE 0 END) AS drs_corner,
	   ROUND(SUM(CASE WHEN pos IN ('LF', 'RF') THEN uzr ELSE 0 END)::numeric,2) AS uzr_corner
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
ORDER BY innings_played_cf DESC;
```
Next, let's take a look at DRS. Let's adjust the DRS data we have by a full-season basis (1458 innings) and take the averages of both.

```
SELECT
	ROUND(AVG(drs_cf / (innings_played_cf/1458)),2) AS drs_cf_adjusted,
	ROUND(AVG(drs_corner / (innings_played_corner/1458)),2) AS drs_corner_adjusted
FROM qualified_cf_corner_outfielders
```
![image](https://github.com/user-attachments/assets/bd95fbd6-e0ac-4f4e-8f98-c6b7b6b86abc)

This group of outfielders have played 7.27 runs better in a corner position than in CF. According to Baseball Reference, their positional adjustments for the corner outfield spots are -7 runs each while in CF it is +2.5 runs. So we should expect the average difference to be about 9.5 runs. There's a gap of ~30% between real and expected values. Let's examine UZR next with the same logic we used for DRS.

![image](https://github.com/user-attachments/assets/4ec1f82b-fc02-41c4-b001-70ab6144ab37)

Here the gap is a bit smaller, with this group of outfielders having played just 5.3 runs better in a corner outfield position compared to centerfield. Fangraphs positional adjustments are also larger, with -7.5 runs each for both LF and RF and the same +2.5 runs in CF, meaning that we should expect the average difference to be about 10 runs. There's a larger gap of nearly 50% between real and expected values. Perhaps the positional adjustments are too harsh, but a potential explanatory factory could be that center fielders tend to be moved to the corners as they get older and past their prime (as Mike Trout is set to do in 2025), so by the time they have made the move they have already lost a step defensively. A deeper analysis should factor in age, but we'll get to that later.


