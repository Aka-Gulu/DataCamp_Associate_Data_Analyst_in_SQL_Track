### Date comparisons
________________________
```sql
-- Count requests created on January 31, 2017
SELECT count(*) 
  FROM evanston311
 WHERE date_created::date = '2017-01-31';

-- Count requests created on February 29, 2016
SELECT count(*)
  FROM evanston311 
 WHERE date_created >= '2016-02-29' 
   AND date_created < '2016-03-01';

-- Count requests created on March 13, 2017
SELECT count(*)
  FROM evanston311
 WHERE date_created >= '2017-03-13'
   AND date_created < '2017-03-13'::date + 1;
```

### Date arithmetic
________________________
```sql
-- Subtract the min date_created from the max
SELECT max(date_created) - min(date_created)
  FROM evanston311;

-- How old is the most recent request?
SELECT now() - max(date_created)
  FROM evanston311;

-- Add 100 days to the current timestamp
SELECT now() + '100 days'::interval;

-- Select the current timestamp, 
-- and the current timestamp + 5 minutes
SELECT now() + '5 minutes'::interval;
```

### Completion time by category
________________________
```sql
-- Select the category and the average completion time by category
SELECT category, 
       avg(date_completed - date_created) AS completion_time
  FROM evanston311
 group by category
-- Order the results
 order by completion_time desc;
```

### Date parts
________________________
```sql
-- Extract the month from date_created and count requests
SELECT EXTRACT(MONTH FROM date_created) AS month, 
       count(*)
  FROM evanston311
 -- Limit the date range
 WHERE EXTRACT(YEAR FROM date_created)>= 2016
   AND EXTRACT(YEAR FROM date_created) < 2018
 -- Group by what to get monthly counts?
 GROUP BY month;

-- Get the hour and count requests
SELECT EXTRACT(HOUR FROM date_created) AS hour,
       count(*)
  FROM evanston311
 GROUP BY hour
 -- Order results to select most common
 ORDER BY count(*) desc
 LIMIT 1;

-- Count requests completed by hour
SELECT date_part('hour',date_completed) AS hour,
       count(*)
  FROM evanston311
 group by hour
 order by count(*) desc;
```

### Variation by day of week
________________________
```sql
-- Select name of the day of the week the request was created 
SELECT to_char(date_created, 'day') AS day, 
       -- Select avg time between request creation and completion
       avg(date_completed - date_created) AS duration
  FROM evanston311 
 -- Group by the name of the day of the week and 
 -- integer value of day of week the request was created
 GROUP BY day, EXTRACT(DOW FROM date_created)
 -- Order by integer value of the day of the week 
 -- the request was created
 ORDER BY EXTRACT(DOW FROM date_created);
```

### Date truncation
________________________
```sql
-- Aggregate daily counts by month
SELECT date_trunc('month',day) AS month,
       avg(count)
  -- Subquery to compute daily counts
  FROM (SELECT date_trunc('day',date_created) AS day,
               count(*) AS count
          FROM evanston311
         GROUP BY day) AS daily_count
 GROUP BY month
 ORDER BY month;
 ```

### Find missing dates
________________________
```sql
SELECT day
-- 1) Subquery to generate all dates
-- from min to max date_created
  FROM (SELECT generate_series(min(date_created),
                               max(date_created),
                               '1 day')::date AS day
          -- What table is date_created in?
          FROM evanston311) AS all_dates
-- 4) Select dates (day from above) that are NOT IN the subquery
 WHERE day NOT IN  
       -- 2) Subquery to select all date_created values as dates
       (SELECT date_created::date
          FROM evanston311);
```

### Custom aggregation periods
________________________
```sql
-- Generate 6 month bins covering 2016-01-01 to 2018-06-30

-- Create lower bounds of bins
SELECT generate_series('2016-01-01',  -- First bin lower value
                       '2018-01-01',  -- Last bin lower value
                       '6 months'::interval) AS lower,
-- Create upper bounds of bins
       generate_series('2016-07-01',  -- First bin upper value
                       '2018-07-01',  -- Last bin upper value
                       '6 months'::interval) AS upper;

-- Count number of requests made per day
SELECT day, count(date_created) AS count
-- Use a daily series from 2016-01-01 to 2018-06-30 
-- to include days with no requests
  FROM (SELECT generate_series('2016-01-01',  -- series start date
                               '2018-06-30',  -- series end date
                               '1 day'::interval)::date AS day) AS daily_series
       LEFT JOIN evanston311
       -- match day from above (which is a date) to date_created
       ON day = date_created::date
 GROUP BY day;


-- Bins from Step 1
WITH bins AS (
	 SELECT generate_series('2016-01-01',
                            '2018-01-01',
                            '6 months'::interval) AS lower,
            generate_series('2016-07-01',
                            '2018-07-01',
                            '6 months'::interval) AS upper),
-- Daily counts from Step 2
     daily_counts AS (
     SELECT day, count(date_created) AS count
       FROM (SELECT generate_series('2016-01-01',
                                    '2018-06-30',
                                    '1 day'::interval)::date AS day) AS daily_series
            LEFT JOIN evanston311
            ON day = date_created::date
      GROUP BY day)
-- Select bin bounds 
SELECT lower, 
       upper, 
       -- Compute median of count for each bin
       percentile_disc(0.5) WITHIN GROUP (ORDER BY count) AS median
  -- Join bins and daily_counts
  FROM bins
       LEFT JOIN daily_counts
       -- Where the day is between the bin bounds
       ON day >= lower
          AND day < upper
 -- Group by bin bounds
 GROUP BY lower, upper
 ORDER BY lower;
```

### Monthly average with missing dates
________________________
```sql
-- generate series with all days from 2016-01-01 to 2018-06-30
WITH all_days AS 
     (SELECT generate_series('2016-01-01',
                             '2018-06-30',
                             '1 day'::interval) AS date),
     -- Subquery to compute daily counts
     daily_count AS 
     (SELECT date_trunc('day', date_created) AS day,
             count(*) AS count
        FROM evanston311
       GROUP BY day)
-- Aggregate daily counts by month using date_trunc
SELECT date_trunc('month',date) AS month,
       -- Use coalesce to replace NULL count values with 0
       avg(coalesce(count, 0)) AS average
  FROM all_days
       LEFT JOIN daily_count
       -- Joining condition
       ON all_days.date=daily_count.day
 GROUP BY month
 ORDER BY month;
```

### Longest gap
________________________
```sql
-- Compute the gaps
WITH request_gaps AS (
        SELECT date_created,
               -- lead or lag
               lag(date_created) OVER (order by date_created) AS previous,
               -- compute gap as date_created minus lead or lag
               date_created - lag(date_created) OVER (order by date_created) AS gap
          FROM evanston311)
-- Select the row with the maximum gap
SELECT *
  FROM request_gaps
-- Subquery to select maximum gap from request_gaps
 WHERE gap = (SELECT max(gap)
                FROM request_gaps);
```

### Rats!
________________________
```sql
-- Truncate the time to complete requests to the day
SELECT date_trunc('day',date_completed-date_created) AS completion_time,
-- Count requests with each truncated time
       count(*)
  FROM evanston311
-- Where category is rats
 WHERE category = 'Rodents- Rats'
-- Group and order by the variable of interest
 GROUP BY completion_time
 ORDER BY count(*) desc;

SELECT category, 
       -- Compute average completion time per category
       avg(date_completed - date_created) AS avg_completion_time
  FROM evanston311
-- Where completion time is less than the 95th percentile value
 WHERE date_completed - date_created < 
-- Compute the 95th percentile of completion time in a subquery
         (SELECT percentile_disc(0.95) WITHIN GROUP (ORDER BY (date_completed - date_created))
            FROM evanston311)
 GROUP BY category
-- Order the results
 ORDER BY avg_completion_time DESC;

-- Compute correlation (corr) between 
-- avg_completion time and count from the subquery
SELECT corr(avg_completion, count)
  -- Convert date_created to its month with date_trunc
  FROM (SELECT date_trunc('month', date_created) AS month, 
               -- Compute average completion time in number of seconds           
               avg(EXTRACT(epoch FROM date_completed - date_created)) AS avg_completion, 
               -- Count requests per month
               count(*) AS count
          FROM evanston311
         -- Limit to rodents
         WHERE category='Rodents- Rats' 
         -- Group by month, created above
         GROUP BY month) 
         -- Required alias for subquery 
         AS monthly_avgs;

-- Compute monthly counts of requests created
WITH created AS (
       SELECT date_trunc('month',date_created) AS month,
              count(*) AS created_count
         FROM evanston311
        WHERE category='Rodents- Rats'
        GROUP BY month),
-- Compute monthly counts of requests completed
      completed AS (
       SELECT date_trunc('month',date_completed) AS month,
              count(*) AS completed_count
         FROM evanston311
        WHERE category='Rodents- Rats'
        GROUP BY month)
-- Join monthly created and completed counts
SELECT created.month, 
       created_count, 
       completed_count
  FROM created
       INNER JOIN completed
       ON created.month=completed.month
 ORDER BY created.month;
```

