# investigating_of_positional_adjustments
Do CF really play 10 runs better in the corners?

# Importing data set from pybaseball
from pybaseball import fielding_stats
import pandas as pd

seasons = range(2005, 2025) # Last 20 seasons

# Fetch and combine all seasons into one DataFrame
df_list = []
for season in seasons:
    data = fielding_stats(season, qual=0)  # Get all players, no innings filter
    data["season"] = season  # Add a season column
    df_list.append(data)

# Combine all seasons
df = pd.concat(df_list, ignore_index=True)

# Save to CSV for PostgreSQL
df.to_csv("fielding_stats_last_20_seasons.csv", index=False)
print("Multi-year export successful!")
