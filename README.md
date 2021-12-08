# Scenerio

Cyclistic is a bike-share company in Chicago. The director of marketing believes the company’s future success depends on maximizing the number of annual memberships. Therefore, the data team wants to understand how casual riders and annual members use Cyclistic bikes differently. From these insights, the team will help the marketing team design a new marketing strategy to convert casual riders into annual members.

# Prepare data

The Cyclistic’s historical trip data was used for this analysis to analyze and identify trends.
Link to the data [here](https://divvy-tripdata.s3.amazonaws.com/index.html). This analysis took data
from November 2020 to October 2021.

# Inspect and clean data with Excel

- Rename rideable_type to bike_type, member_casual to user_type
- Add a column called “day_of_week,” and calculate the day of the week that each ride started using the “WEEKDAY”
command( noting that 1 = Sunday and 7 = Saturday)
- Remove duplicates
- Delete unnesscessary columns (start_station_id, end_station_id, start_lat, start_lng, end_lat, end_lng)
- Detect cells in station_name column contains “BIKE CHECKING”, "Temp"
or “TESTING” which indicate test rides by the company (these values will be removed in BigQuery).

# Transform data with BigQuery

- Merge 12 tables of 12 months into one table
```sql
CREATE TABLE `cyclistic-bikeshare-334403.cyclistic.12_months` AS (
    SELECT * FROM `cyclistic-bikeshare-334403.cyclistic.202011`
    UNION ALL
    SELECT * FROM `cyclistic-bikeshare-334403.cyclistic.202012`
    UNION ALL
    SELECT * FROM `cyclistic-bikeshare-334403.cyclistic.202101`
    UNION ALL
    SELECT * FROM `cyclistic-bikeshare-334403.cyclistic.202102`
    UNION ALL
    SELECT * FROM `cyclistic-bikeshare-334403.cyclistic.202103`
    UNION ALL
    SELECT * FROM `cyclistic-bikeshare-334403.cyclistic.202104`
    UNION ALL
    SELECT * FROM `cyclistic-bikeshare-334403.cyclistic.202105`
    UNION ALL
    SELECT * FROM `cyclistic-bikeshare-334403.cyclistic.202106`
    UNION ALL
    SELECT * FROM `cyclistic-bikeshare-334403.cyclistic.202107`
    UNION ALL
    SELECT * FROM `cyclistic-bikeshare-334403.cyclistic.202108`
    UNION ALL
    SELECT * FROM `cyclistic-bikeshare-334403.cyclistic.202109`
    UNION ALL
    SELECT * FROM `cyclistic-bikeshare-334403.cyclistic.202110`
    ORDER BY started_at
```
- Add columns “ride_length”, “ride_month” and convert day_of_week from numbers to strings, then fitler out test rides and rides with duration <= 0
```sql
WITH new_table AS (
    SELECT * EXCEPT (day_of_week),
            DATETIME_DIFF (ended_at, started_at, MINUTE) AS ride_length,
            EXTRACT (MONTH from started_at) AS ride_month,
            CASE 
                WHEN day_of_week = 1 THEN "Sunday"
                WHEN day_of_week = 2 THEN "Monday"
                WHEN day_of_week = 3 THEN "Tueday"
                WHEN day_of_week = 4 THEN "Wednesday"
                WHEN day_of_week = 5 THEN "Thursday"
                WHEN day_of_week = 6 THEN "Friday"
            ELSE "Saturday" END AS day_of_week
    FROM `cyclistic-bikeshare-334403.cyclistic.12_months`  
)
SELECT *
FROM new_table 
WHERE  start_station_name NOT LIKE "%CHECKING%"
    AND start_station_name NOT LIKE "%TESTING%"
    AND start_station_name NOT LIKE "%Temp%"
    AND end_station_name NOT LIKE "%CHECKING%"
    AND end_station_name NOT LIKE "%TESTING%"
    AND start_station_name NOT LIKE "%Temp%"
    AND ride_length > 0
```
Save the query results as "final_table" and proceed to the analyzing step

# Analyze data

- Total number of rides by customer type
```sql
SELECT user_type,
        COUNT(user_type) AS num_of_rides
FROM `cyclistic-bikeshare-334403.cyclistic.final_table`
GROUP BY user_type
```
- Number of rides by bike type
```sql
SELECT  bike_type,
        user_type,
        COUNT(bike_type) AS num_of_rides,
FROM `cyclistic-bikeshare-334403.cyclistic.final_table`
GROUP BY bike_type, user_type
ORDER BY bike_type
```
- Average ride length by customer type
```sql
SELECT user_type,
    ROUND(AVG(ride_length), 2) AS avg_ride_length
FROM `cyclistic-bikeshare-334403.cyclistic.final_table`
GROUP BY user_type
```
- Number of rides by month 
```sql
SELECT ride_month,
	   user_type,
       COUNT(user_type) AS num_of_rides
FROM `cyclistic-bikeshare-334403.cyclistic.final_table`
GROUP BY ride_month, user_type
ORDER BY ride_month
```
- Average ride length by month
```sql
SELECT ride_month,
	   user_type,
       ROUND(AVG(ride_length), 2) AS avg_ride_length
FROM `cyclistic-bikeshare-334403.cyclistic.final_table`
GROUP BY ride_month, user_type
ORDER BY ride_month
```
- Number of rides by day of week
```sql
SELECT day_of_week,
	   user_type,
       COUNT(user_type) AS num_of_rides
FROM `cyclistic-bikeshare-334403.cyclistic.final_table`
GROUP BY day_of_week, user_type
ORDER BY day_of_week
```
- Average ride length by day of week
```sql
SELECT day_of_week,
	   user_type,
       ROUND(AVG(ride_length), 2) AS avg_ride_length
FROM `cyclistic-bikeshare-334403.cyclistic.final_table`
GROUP BY day_of_week, user_type
ORDER BY day_of_week
```
- Number of rides by hour of the day
```sql
SELECT EXTRACT(HOUR from started_at) AS hour_of_the_day,
	   COUNT(started_at) AS num_of_rides,
	   user_type
FROM `cyclistic-bikeshare-334403.cyclistic.final_table`
GROUP BY hour_of_the_day, user_type
ORDER BY hour_of_the_day
```
# Data visualization with Tableau

![cyclistic bike-share case study](https://user-images.githubusercontent.com/91933248/145233385-76406915-3e9f-4cc5-aaaa-3ea8e468215f.png)

# Key insights and recommendations
## Key insights
- Members accounted for more than half of bike users (54.7%), while the proportion of casual riders was 45.3%.
- The casual riders' average ride length was about 3 times higher than that of members.
- Classic bike was the most chosen bike type by both members and casual riders.
- Both type of users enjoyed riding in summer months (July to September); however they spent more time for a riding trip in Feburary.
- Members had a pretty steady number of rides each day throughout the week, casual riders, on the other hand, took more rides on weekend as apposed to weekdays.
- Regarding the average ride length by weekday, members rode for about 12 to 15 minutes a day (from Monday to Sunday), while casual riders rode more on weekends than they did on weekday (about 30 minutes on weekdays and 35 to 40 minutes on weekends).
- Casual riders had more moring rides than members. At noon, the number of rides of casual riders and members were almost the same, and they both hit the peak at 5pm.

## Recommendations:

- Launch promotion compaigns on summer months and offer casual riders a discount on weekend if they sign up for annual membership
- Consider adding a weekend-only membership to convert more casual riders to members
- It seems that casual riders haven't had the need of steady bike use everyday. Carry out research to find out the reasons then execute solutions to convert more casual riders to members. 
