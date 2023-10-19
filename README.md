# Example Fitbit Queries

## Looking at summaries

### Sleep summary query

```sql
SELECT participantidentifier,
        date_format(min(startdate), '%Y-%m-%d') as start_date,
        date_format(max(enddate),'%Y-%m-%d') as end_date,
        count(DISTINCT(date_format(startdate, '%Y-%m-%d'))) as days_recorded,
        count(participantidentifier) as sleep_sessions,
        CAST(split_part(CAST(max(enddate) - min(startdate) as varchar), ' ', 1) as INT) + 1 as calendar_day,
        round(avg(cast(efficiency as INT)), 2) as average_eff
            FROM fitbitsleeplogs 
            WHERE 
              duration > 0 
              and startdate > date('2023-02-01')
              and enddate < date('2023-05-01')
            GROUP BY participantidentifier
            ORDER BY start_date ASC
```

### Activity summary query 

```sql
SELECT participantidentifier,
        date_format(min(date), '%Y-%m-%d') as start_date,
        date_format(max(date), '%Y-%m-%d') as end_date,
        round(avg(steps), 2) as average_steps,
        count(date) as days_recorded,
        CAST(split_part(CAST(max(date) - min(date) as varchar), ' ', 1) as int) + 1 as calendar_days
            FROM fitbitdailydata
            WHERE 
              steps > 0
              and date > date('2023-02-01')
              and date < date('2023-05-01')
            GROUP BY participantidentifier
```

### HRV summary query 

```sql
SELECT participantidentifier,
        date_format(min(date), '%Y-%m-%d') as start_date,
        date_format(max(date), '%Y-%m-%d') as end_date,
        count(date) as days_recorded,
        CAST(split_part(CAST(max(date) - min(date) as varchar), ' ', 1) as int) + 1 as calendar_days,
        avg(hrvdailyrmssd) as daily_avg_hrv, avg(hrvdeeprmssd) as deep_avg_hrv,
        min(hrvdailyrmssd) as daily_min_hrv, min(hrvdeeprmssd) as deep_min_hrv,
        max(hrvdailyrmssd) as daily_max_hrv, max(hrvdeeprmssd) as deep_max_hrv
            FROM fitbitdailydata
            WHERE 
							hrvdailyrmssd IS NOT NULL
							and date > date('2023-02-01')
              and date < date('2023-05-01')
            GROUP BY participantidentifier
```

## Diving into sleep

### Sleep Effeciency query 

```sql
select participantidentifier, startdate, cast(efficiency as int) efficiency 
  from fitbitsleeplogs 
  where 
     duration > 0 
     and startdate > date('2023-02-01')
     and enddate < date('2023-05-01')
  order by participantidentifier, startdate asc
```

### Only night time sleep filter

```sql
select participantidentifier, 
  startdate, 
  cast(efficiency as int) efficiency
  from fitbitsleeplogs 
  where 
     duration > 0 
     and startdate > date('2023-02-01')
     and enddate < date('2023-05-01')
     and (hour(startdate) >= 20 or hour(startdate) <= 3)
  order by participantidentifier, startdate asc
```

### Sleep with duration and efficiency both.

```sql
select participantidentifier, 
  startdate, 
  cast(efficiency as int) efficiency,
  round(duration / 1000.0 / 60 / 60 , 2) duration_h
  from fitbitsleeplogs 
  where 
     duration > 0 
     and startdate > date('2023-02-01')
     and enddate < date('2023-05-01')
     and (hour(startdate) >= 20 or hour(startdate) <= 3)
  order by participantidentifier, startdate asc
```

## Activity + Sleep

### Activity Queries  - Start with this

```sql
 select concat(participantidentifier, ':', cast(date_format(date, '%y-%m-%d') as varchar)) pid, 
  date_format(date, '%y-%m-%d') date, 
  activitycalories 
  from fitbitdailydata
```

### Sleep Query

```sql
select concat(participantidentifier, ':', cast(date_format(startdate, '%y-%m-%d') as varchar)) pid,
  date_format(startdate, '%y-%m-%d') date, 
  cast(efficiency as int) efficiency
  from fitbitsleeplogs 
  where 
     duration > 0 
     and startdate > date('2023-02-01')
     and enddate < date('2023-05-01')
     and (hour(startdate) >= 20 or hour(startdate) <= 3)
  order by participantidentifier, startdate asc
```

### Put it all together in a single table

```sql
with activity as (
select concat(participantidentifier, ':', cast(date_format(date, '%y-%m-%d') as varchar)) pid, 
  date_format(date, '%y-%m-%d') date, 
  activitycalories 
  from fitbitdailydata 
),
sleep as (
 select concat(participantidentifier, ':', cast(date_format(startdate, '%y-%m-%d') as varchar)) pid,
  date_format(startdate, '%y-%m-%d') date, 
  cast(efficiency as int) efficiency
  from fitbitsleeplogs 
  where 
     duration > 0 
     and startdate > date('2023-02-01')
     and enddate < date('2023-05-01')
     and (hour(startdate) >= 20 or hour(startdate) <= 3)
  order by participantidentifier, startdate asc
)
select sleep.pid, sleep.date, sleep.efficiency,
  activity.date, activity.activitycalories
  from sleep 
  inner join activity 
  on sleep.pid = activity.pid
```

### Clean up the table and get it back to normal.

```sql
with activity as (
select concat(participantidentifier, ':', cast(date_format(date, '%y-%m-%d') as varchar)) pid, 
  date_format(date, '%y-%m-%d') date, 
  activitycalories 
  from fitbitdailydata 
),
sleep as (
 select concat(participantidentifier, ':', cast(date_format(startdate, '%y-%m-%d') as varchar)) pid,
  date_format(startdate, '%y-%m-%d') date, 
  cast(efficiency as int) efficiency
  from fitbitsleeplogs 
  where 
     duration > 0 
     and startdate > date('2023-02-01')
     and enddate < date('2023-05-01')
     and (hour(startdate) >= 20 or hour(startdate) <= 3)
  order by participantidentifier, startdate asc
)
select split_part(sleep.pid, ':', 1) participantidentifier, 
  sleep.date, 
  sleep.efficiency, 
  activity.activitycalories
  from sleep 
  inner join activity 
  on sleep.pid = activity.pid
```

## Other cool queries

### Get a summary of all the activities performed by an individual along with their HR

```sql
with ihr as (
	select "participantidentifier",
		count(value) hr_counts
	from "myintradayhr"
	where "value" is not null
		and datetime > date('2023-01-01')
	group by "participantidentifier"
),
dd as (
	select "participantidentifier",
		avg("steps") avg_steps
	from "fitbitdailydata"
	where steps > 0
		and date > date('2023-01-01')
	group by "participantidentifier"
),
fd as (
	select "participantidentifier",
		"device"
	from "fitbitdevices"
),
fa as (
	select "participantidentifier",
		array_agg(
			distinct(
				concat_ws(':', "activityname", cast(avg_hr as varchar))
			)
		) activities_hr
	from (
			select "participantidentifier",
				"activityname",
				"averageheartrate",
				cast(
					avg("averageheartrate") over (
						partition by "participantidentifier",
						"activityname"
						order by "participantidentifier" asc
					) as int
				) avg_hr
			from "fitbitactivitylogs"
			where "startdate" > date('2023-01-01')
			order by "activityname" asc
		)
	group by "participantidentifier"
),
fs as (
    select "participantidentifier", avg(cast("efficiency" as double)) avg_eff
        from "fitbitsleeplogs"
        where "startdate" > date('2023-01-01')
        group by "participantidentifier"
)
select dd."participantidentifier", dd.avg_steps, ihr.hr_counts, fs.avg_eff,
    fd."device", fa.activities_hr
    from dd
    left join ihr
    on dd."participantidentifier" = ihr."participantidentifier"
    left join fs 
    on dd."participantidentifier" = fs."participantidentifier"
    left join fd 
    on dd."participantidentifier" = fd."participantidentifier"
    left join fa
    on dd."participantidentifier" = fa."participantidentifier"
```

### Get resting HR from Fitbit data.

```sql
with fd as (
	select "participantidentifier",
		count("restingheartrate") "rh_daily_count",
		round(avg("restingheartrate"), 2) "rh_daily_avg",
		round(stddev("restingheartrate"), 2) "rh_daily_std"
	from "fitbitdailydata"
	where "restingheartrate" is not null
	group by "participantidentifier"
	order by "participantidentifier"
),
rr as (
	select "participantidentifier",
		count("restingheartrate") "rh_resting_count",
		round(avg("restingheartrate"),2) "rh_resting_avg",
		round(stddev("restingheartrate"),2) "rh_resting_std"
	from "fitbitrestingheartrates"
	where "restingheartrate" is not null
	group by "participantidentifier"
	order by "participantidentifier"
)
select fd."participantidentifier",
	fd.rh_daily_count, rr.rh_resting_count,
	fd.rh_daily_avg, rr.rh_resting_avg,
	fd.rh_daily_std, rr.rh_resting_std
from fd
	left join rr on fd."participantidentifier" = rr."participantidentifier
```
