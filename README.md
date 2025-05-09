## fresh_segments
a digital marketing agency that helps other businesses analyse trends in online ad click behaviour for their unique customer base.



## Overview 
Clients share their customer lists with the Fresh Segments team who then aggregate interest metrics and generate a single dataset worth of metrics for further analysis.

In particular - the composition and rankings for different interests are provided for each client showing the proportion of their customer list who interacted with online assets related to each interest for each month.

Danny needs assistance to analyse aggregated metrics for an example client and provide some high level insights about the customer list and their interests.

## Objectives 
- Interest Analysis
- Segment Analysis
- Index Analysis


## Dataset
The dataset for this project is sourced from [Danny Ma](https://www.linkedin.com/in/datawithdanny)

## About Dataset 
- Fresh_segments.interest metrics  table contains information about aggregated interest metrics for a specific major client of Fresh Segments which makes up a large proportion of their customer base.

Each record in this table represents the performance of a specific interest_id based on the client’s customer base interest measured through clicks and interactions with specific targeted advertising content.

 The table below is a part of the Interest metrics table. 
 ![interest metrics](https://github.com/Ifeoma28/fresh_segments/blob/48e32eb6a29b39329a7c08cbb1f7ea7f96d08aa4/interest%20metrics.png)
 
 For example - let’s interpret the first row of the interest_metrics table together:
In July 2018, the composition metric is 11.89, meaning that 11.89% of the client’s customer list interacted with the interest interest_id = 32486 

we can link this  interest_id to a separate mapping table to find the segment name called “Vacation Rental Accommodation Researchers”.


- Fresh_segments.interest map contains  links the interest_id with their relevant interest information.

-  You will need to join this table onto the previous interest_details table to obtain the interest_name as well as any details about the summary information.
The table below shows part of the interest map table.
![interest map](https://github.com/Ifeoma28/fresh_segments/blob/502040404bdcbd76213478a0366d63c2213903eb/interest%20map.png)

## Tools Used
- MSSQL for querying the database.
- Data cleaning and exploration.
- Power bi for visualization 
1) I changed the blank collumns to nulls.
2) I changed the data type of the column and extracted the year column from month_year column.
3) The data contained 1194 null interest ID (records)
4) The interest metrics table contained 1202 distinct interests
```
SELECT COUNT(DISTINCT interest_id) AS interest_count
FROM interest_metrics_2;
-- 1202 Interests
```
The interest map table contained information on 1209 interests and their ID.
These are the queries performed when cleaning the interest metrics dataset.
```
--update NULL values
--UPDATE interest_metrics
--SET _year = NULL
--WHERE LTRIM(RTRIM(_year)) = '';

--ALTER TABLE interest_metrics
--ALTER COLUMN _year INT;

--UPDATE interest_metrics
--SET month_year = NULL
--WHERE month_year = 'null';

--UPDATE interest_metrics
--SET interest_id = NULL
--WHERE interest_id = 'null';

--ALTER TABLE interest_metrics
--ALTER COLUMN month_year VARCHAR(10);

--UPDATE interest_metrics
--SET month_year = 
	--CONCAT(
	--SUBSTRING(month_year,4, 4), -- year
	--'-',
	--SUBSTRING(month_year,1,2), -- month
	--'-01' -- day)
--WHERE month_year IS NOT NULL;

--ALTER TABLE interest_metrics
--ALTER COLUMN  month_year DATE;

```

## BUSINESS QUESTIONS

# Interest Analysis
- Which interests have been present in all month_year dates in our dataset?
```
-- i used the distinct fxn to know how many unique dates we have in the month_year column(1st day of the month)
-- then filtered the interest_id to know how that have 14 unique month year count.
SELECT interest_id,unique_month_year_count,interest_name
FROM
	(SELECT interest_id, COUNT(DISTINCT month_year) AS unique_month_year_count
	FROM interest_metrics_2 me
	GROUP BY interest_id) AS month_table
JOIN interest_map ma ON ma.id = month_table.interest_id
WHERE unique_month_year_count = 14;
-- we can see that 480 interests were present in all our datasets.
```
- Using this same total_months measure - calculate the cumulative percentage of all records starting at 14 months - which total_months value passes the 90% cumulative percentage value?
```
WITH interest_month_count AS
(SELECT interest_id, COUNT(DISTINCT month_year) AS total_month
FROM interest_metrics_2
GROUP BY interest_id
),
total_counts AS (SELECT total_month,COUNT(*) AS interest_count
FROM interest_month_count
GROUP BY total_month),
cummulative_calc AS ( SELECT total_month,SUM(CASE WHEN total_month >= 14 THEN interest_count ELSE NULL END)  AS cummulative_count,
SUM(interest_count) OVER() AS total_records
FROM total_counts
GROUP BY total_month,interest_count)
SELECT total_month,cummulative_count,total_records,ROUND((CAST(cummulative_count AS float)/total_records)*100.0,2)
AS cummulative_percent
FROM cummulative_calc
WHERE total_month >= 14
ORDER BY 1 DESC;
```
- If we were to remove all interest_id values which are lower than the total_months value we found in the previous question - how many total data points would we be removing?
we would be removing ?
```
WITH interest_count AS (SELECT interest_id, COUNT(DISTINCT month_year) AS total_month
FROM interest_metrics_2
GROUP BY interest_id),
interest_to_remove AS (SELECT interest_id,total_month
FROM interest_count
WHERE total_month <> 14)
-- 14 is the unique number of month year date we have
SELECT * FROM interest_to_remove
-- we would be removing 723 interests which is 6,360 data points
;

```
- If we remove these interests, we would be left with just 480 interests (7913) that is present in all month_year dates in our dataset.

# Segment Analysis
- Using our filtered dataset by removing the interests with less than 6 months worth of data, which are the top 10 and bottom 10 interests which have the largest composition values in any month_year? Only use the maximum composition value for each interest but you must keep the corresponding month_year.
```
-- using our filtered datasets (< 14 months) by removing the interests with less than 6 months worth of data,
-- which are the top 10 and bottom 10 interests which have the largest composition values in 
-- any month year ? only use the maximum composition value for each interest but keep the
-- corresponding month_year using SQL
-- im using the second table 
-- first lets get the top10 interests by composition with 6+ months worth of data
WITH interest_count AS (SELECT interest_id, COUNT(DISTINCT month_year) AS total_month
FROM interest_metrics_2 
GROUP BY interest_id),
filtered_interest AS (SELECT im.*
FROM interest_count ic
JOIN interest_metrics_2 im ON im.interest_id = ic.interest_id
WHERE ic.total_month >= 6
),
-- next get the maximum composition by interest with its corresponding month_year
max_composition_per_interest AS (
SELECT interest_id,composition,month_year
FROM 
	( SELECT *,
		RANK() OVER(PARTITION BY interest_id ORDER BY composition DESC) AS rank
		FROM filtered_interest) AS ranked
WHERE rank =1
)
SELECT TOP 10  mpi.*,ip.interest_name
FROM max_composition_per_interest mpi
JOIN interest_map ip  ON mpi.interest_id = ip.id
ORDER BY composition DESC;
-- interest with id 21057 (Work comes first travelers) is the highest with a composition of 21.2
```
```
-- now for the bottom 10 interests with 6+ worth of data
WITH interest_count AS (SELECT interest_id, COUNT(DISTINCT month_year) AS total_month
FROM interest_metrics_2 
GROUP BY interest_id),
filtered_interest AS (SELECT im.*
FROM interest_count ic
JOIN interest_metrics_2 im ON im.interest_id = ic.interest_id
WHERE ic.total_month >= 6
),
-- next get the maximum composition by interest with its corresponding month_year
max_composition_per_interest AS (
SELECT interest_id,composition,month_year
FROM 
	( SELECT *,
		RANK() OVER(PARTITION BY interest_id ORDER BY composition DESC) AS rank
		FROM filtered_interest) AS ranked
WHERE rank =1
)
SELECT TOP 10 mpi.*,ip.interest_name
FROM max_composition_per_interest mpi
JOIN interest_map ip  ON mpi.interest_id = ip.id
ORDER BY composition ASC;
-- interest_id 33958 (Astrology enthusiasts) is the least with a composition of 1.88
```
look at the average composition of interests with 6months worth of data over time below .
Work Comes First Travelers dominated consistently across months, showing not only high average composition but also a relatively stable trajectory.
![Top and bottom interests](https://github.com/Ifeoma28/fresh_segments/blob/bf2cc8f23d6ee23d3589906b94b8dad9d45bee0f/Top%20and%20bottom%20interest.png)

- Which 5 interests had the lowest average ranking value?
```
-- which 5 interests had the lowest average ranking value
-- i used the cast fxn because the interest name column is in text datatype 
-- so i converted to varchar before i could perform a join
SELECT TOP 5 interest_id,AVG(ranking) AS avg_ranking,CAST(ip.interest_name AS VARCHAR(MAX)) AS interest_name
FROM interest_metrics_2  im
JOIN interest_map ip ON im.interest_id = ip.id
GROUP BY interest_id,CAST(ip.interest_name AS VARCHAR(MAX))
ORDER BY 2 ASC;
-- Winter Apparel shoppers with ID(41548) had the lowest average ranking of 1 followed by Fitness Activity tracker users with ID(42203)
-- and Mens shoe shoppers with ID (115),Elite cycling gear shoppers with ID (48154) AND shoe shoppers with ID (171)
```
This can be seen in the decline in average composition over time.
![low ranked](https://github.com/Ifeoma28/fresh_segments/blob/073db56ca186b79e8bd43c61346ee686ff976a2f/low%20interest%20ranking.png)

- Which 5 interests had the largest standard deviation in their percentile_ranking value?
```
SELECT TOP 5 interest_id,STDEV(percentile_ranking) AS stdev_percentile_ranking,CAST(ip.interest_name AS VARCHAR(MAX)) AS interest_name
FROM interest_metrics_2 im
JOIN interest_map ip ON im.interest_id = ip.id
GROUP BY interest_id,CAST(ip.interest_name AS VARCHAR(MAX)) 
ORDER BY 2 DESC;
-- Blockbuster movie fans with Id (6260) has the largest standard deviation in their percentile ranking value
```
![standard deviation](https://github.com/Ifeoma28/fresh_segments/blob/ce517436ed2f977c1f7576df6dddb92be2aeb05e/blockbuster%202.png)

- For the 5 interests found in the previous question - what was minimum and maximum percentile_ranking values for each interest and its corresponding year_month value?
```
SELECT interest_id,month_year,MIN(percentile_ranking) AS min_percentile,MAX(percentile_ranking) AS max_percentile
FROM interest_metrics_2
WHERE interest_id IN (6260, 131, 150, 23, 20764)
GROUP BY interest_id,month_year
ORDER BY 1;
-- the minimum and max.percentiles keep reducing every first day of the month for these interests with the largest standard deviation
```
- For these 5 interests that the largest standard deviation in their percentile_ranking value, we can see a steady decrease in the minimum and maximum percentile_ranking every month 

- How would you describe our customers in this segment based off their composition and ranking values? What sort of products or services should we show to these customers and what should we avoid?

1) Customers with the Largest Standard Deviation.
These customers engage with interests that show high volatility—their composition and ranking values fluctuate significantly over time.

Example: Blockbuster Movie Fans had large swings in percentile ranking—indicating periods of intense interest followed by drops.

2) Customers with the Lowest Standard Deviation
These customers show stable, consistent engagement across time. Interests like Winter Apparel Shoppers or Fitness Tracker Users have steady composition values despite having a low average ranking. 

- Products to show
1) High Stdv :Timely, seasonal, trend-based and  Campaigns aligned with current events (e.g. new movie releases, tech launches).
2) Low stdv : Subscriptions, core offerings

- Products not to show
1) High Stdv : Long-term product pitches or generic ads—these may miss the engagement window.
2) low Stdv: Flashy, short-term offers or rapidly changing messaging—they don’t need urgency to engage.

Also, I plotted Average Composition vs. Standard Deviation of Percentile Ranking to understand volatility.

And here’s the kicker:
There was no clear pattern.
That tells me:

- High engagement doesn’t always mean stability(low standard deviation).
- Low average interests can still be rock solid in how they rank over time.


# Index Analysis
- The index_value is a measure which can be used to reverse calculate the average composition for Fresh Segments clients.
Average composition can be calculated by dividing the composition column by the index_value column rounded to 2 decimal places.

- What is the top 10 interests by the average composition for each month?
```
WITH Monthly_Avg_Composition AS (
    SELECT
        month_year,
        interest_id,
       ROUND( SUM(composition) * 1.0 / NULLIF(SUM(index_value), 0),2) AS avg_composition
    FROM interest_metrics_2
    GROUP BY month_year, interest_id
),
Top_10_Interests_Per_Month AS (
    SELECT *
        FROM(
        SELECT *,
        ROW_NUMBER() OVER (PARTITION BY month_year ORDER BY avg_composition DESC) AS rank
    FROM Monthly_Avg_Composition
) AS ranked
WHERE rank <= 10
)
SELECT 
    tm.*,CAST(ip.interest_name AS VARCHAR(MAX)) AS interest_name,tm.avg_composition
FROM Top_10_Interests_Per_Month tm
JOIN interest_map ip ON tm.interest_id = ip.id
ORDER BY 1,3 DESC;
-- for the month of July 2018 the top interest is Las vegas trip planners (6324)
-- for the month of August 2018 the top interest still remains Las vegas trip planners (6324)
-- for the month of september 2018 the top interest is Work comes first travelers (21057)
-- for the month of october 2018 the top interest is still Work comes first travelers(21057)
-- for the month of november 2018 the top interest is Work comes first travelers(21057)
-- for the month of december 2018 the top interest is Work comes first travelers (21057)
-- for the month of january 2019 the top interest is Work comes first travelers (21057)
-- for the month of january 2019 the top interest is Work comes first travelers (21057)
-- for the month of March 2019 the top interest is Alabama trip planners (7541)
-- for the month of April 2019 the top interest is solar energy researchers(6065)
-- for the month of May 2019 the top interest is Readers of honduran content (21245)
-- for the month of June 2019 the top interest is Las vegas trip planners (6324)
-- for the month of July 2019 the top interest is Las vegas trip planners (6324)
-- for the month of August 2019 the top interest is cosmetics and beauty shopper (4898)
```
- For all of these top 10 interests - which interest appears the most often?
Its obvious the Work comes first travelers with ID (21057) appears most often.

- What is the average of the average composition for the top 10 interests for each month?
```
WITH Monthly_Avg_Composition AS (
    SELECT
        month_year,
        interest_id,
        SUM(composition) AS total_composition,
        SUM(index_value) AS total_index,
       ROUND( SUM(composition) * 1.0 / NULLIF(SUM(index_value), 0),2) AS avg_composition
    FROM interest_metrics_2
    GROUP BY month_year, interest_id
),
Top_10_Interests_Per_Month AS (
    SELECT *
        FROM(
        SELECT *,
        ROW_NUMBER() OVER (PARTITION BY month_year ORDER BY avg_composition DESC) AS rank
    FROM Monthly_Avg_Composition
) AS ranked
WHERE rank <= 10
)
SELECT 
    DISTINCT month_year,ROUND(AVG(avg_composition) OVER(PARTITION BY month_year),2) AS avg_top_10interests
FROM Top_10_Interests_Per_Month
GROUP BY month_year,avg_composition;
```
- What is the 3 month rolling average of the max average composition value from September 2018 to August 2019 and include the previous top ranking interests in the output shown below.
```
WITH Monthly_Avg_Composition AS (
    SELECT
        month_year,
        interest_id,CAST(ip.interest_name AS VARCHAR(MAX)) AS interest_name,
       ROUND( SUM(composition) * 1.0 / NULLIF(SUM(index_value), 0),2) AS avg_composition
    FROM interest_metrics_2 im
	JOIN interest_map ip ON im.interest_id = ip.id
    GROUP BY month_year, interest_id,CAST(ip.interest_name AS VARCHAR(MAX))
),
Top_Interests_Per_Month AS (
    SELECT *
        FROM(
        SELECT *,
        ROW_NUMBER() OVER (PARTITION BY month_year ORDER BY avg_composition DESC) AS rank
    FROM Monthly_Avg_Composition
) AS ranked
WHERE rank = 1
),
RollingAvg AS (SELECT month_year,interest_id,interest_name,
	avg_composition,ROUND(AVG(avg_composition) OVER(ORDER BY month_year
	ROWS BETWEEN 2 PRECEDING AND CURRENT ROW),2) 
	AS rollin_3month_avg,LAG(avg_composition) OVER(ORDER BY month_year) AS one_month_ago,
	LAG(avg_composition,2)  OVER (ORDER BY month_year) AS two_month_ago
	FROM Top_Interests_Per_Month
)
SELECT 
    month_year,Ra.interest_id,Ra.interest_name ,
	avg_composition AS max_index_composition,rollin_3month_avg,
	CAST(it.interest_name AS VARCHAR) + ':' + CAST(one_month_ago AS VARCHAR) AS one_month_interest ,
	CAST(it.interest_name AS VARCHAR) + ':' + CAST(two_month_ago AS VARCHAR) AS two_month_interest 
FROM RollingAvg Ra
LEFT JOIN interest_map it ON Ra.interest_id = it.id
LEFT JOIN interest_map ip ON Ra.interest_name = CAST(ip.interest_name AS VARCHAR(MAX))
WHERE month_year BETWEEN '2018-09-01'AND'2019-08-01'
ORDER BY 1,4 DESC;
```
- Provide a possible reason why the max average composition might change from month to month? Could it signal something is not quite right with the overall business model for Fresh Segments?
1) Yes, the maximum average composition changing month to month can be a normal reflection of shifting interests.
   
2) People’s interests change due to seasonality, news cycles, or life events.
For example: “Alabama Trip Planners” may spike in summer and drop in fall.

3) Marketing Campaigns or Partnerships
If Fresh Segments collaborates with brands or runs promotions targeting specific interests (e.g., solar energy or cosmetics), that could temporarily boost certain segments.

## Insights 
- Out of 1,202 distinct interests, 480 interests were consistently present across all 14 unique months, showing a strong baseline for longitudinal analysis.

- However, 1,194 null records indicate missing or incomplete data—worth investigating to ensure data quality and impact on analysis.
  
- By filtering out interests with less than 6 months of data, i ensured consistency and reliability in trend analysis.

- Top Interest: Work Comes First Travelers (ID: 21057) peaked with a composition of 21.2, indicating a strong affinity or engagement.

- Bottom Interest: Astrology Enthusiasts (ID: 33958) had a composition of 1.88, suggesting low engagement or niche relevance.
  
- Ranking Trends
Winter Apparel Shoppers (ID: 41548) had the lowest average ranking (1), showing strong, consistent performance.

- Other strong performers include Fitness Activity Tracker Users, Men's Shoe Shoppers, and Elite Cycling Gear Shoppers.

- Blockbuster Movie Fans (ID: 6260) showed the highest standard deviation in percentile ranking, suggesting fluctuating interest—possibly linked to movie releases or seasonal trends.
- Customers with Low Standard Deviation
Top Interests:
1) Winter Apparel Shoppers

2) Online Role-Playing Game Enthusiasts

3) Readers of Honduran Content

4) Sci-Fi Movie and TV Enthusiasts

5) Fitness Activity Tracker Users

- out of these top 5 interests with low standard deviation,Winter Apparel Shoppers and Fitness Activity Tracker have a very high composition. Also Readers of Honduran content in the month of May 2019 have a high average composition too. it suggests a strong and consistently popular segment, which is great for broad promotions.
  
- Monthly Top Interests (July 2018 – August 2019)
Las Vegas Trip Planners (ID: 6324) and Work Comes First Travelers (ID: 21057) repeatedly dominated multiple months.

- Solar Energy Researchers, Alabama Trip Planners, and Cosmetics & Beauty Shoppers emerged in specific months, suggesting temporal or campaign-based spikes.

- Peak Engagement Month
October 2018 had the highest average of average composition for top 10 interests at 7.07, indicating a period of heightened user engagement or interest alignment.

## Recommendation 
- Frequent shifts in top interests aren’t necessarily bad — but if Fresh Segments wants to build a sustainable and loyal user base, it needs to understand why certain interests dominate at certain times, and what causes them to fall off.
  
- Blockbuster Movie Fans had the highest standard deviation in average composition and it wasn’t random. The sharp decline reflects a deeper shift in entertainment consumption. For Fresh Segments, it’s a sign to re-evaluate how entertainment interests are evolving and whether certain legacy segments are worth pursuing.

- Interests like Men’s Shoe Shoppers, Shoe Shoppers, and Fitness Activity Tracker Users display a declining trend in composition.
This could imply shifting consumer behavior or seasonal relevance.

- A popular segment can also be the most unstable. (high standard deviation).
If Fresh Segments targets based solely on popularity, it might miss durable, low-noise segments.
- Don't push fitness or astrology-related offers unless you're targeting niche segments. Their low average ranking and possible lack of stickiness means poor ROI.
- 
