
 

[SQL 과제](https://15445.courses.cs.cmu.edu/fall2018/homework1/)

### Quiz 2

> ### Solution & My Query

``` sql
SELECT city, COUNT(1) as count
FROM station
GROUP BY city
ORDER BY count, city;
```

### Quiz 3

> ### Solution

    On Postgres 10.6 : 7 secs + @

``` sql
select city_trip_cnt.city, round(city_trip_cnt.cnt * 1.0 / trip_cnt.cnt, 4) as ratio 
from (select city, count(distinct(id)) as cnt 
      from trip, station 
      where station_id = start_station_id or station_id = end_station_id 
      group by city) as city_trip_cnt, 
     (select count(*) as cnt from trip) as trip_cnt 
order by ratio desc, city asc;
```

> ### My Query

    On Postgres 10.6 : 1 secs + @

``` sql
WITH cities(start_city, end_city) AS (
	SELECT a.city, b.city
	FROM (SELECT tr.id, st.city
		  FROM station st JOIN trip tr
	      ON st.station_id = tr.start_station_id) a 
          JOIN
	     (SELECT tr.id, st.city
	      FROM station st JOIN trip tr
	      ON st.station_id = tr.end_station_id) b
          ON a.id = b.id), 
visited(city) AS (
	SELECT start_city
	FROM cities
	UNION ALL
	SELECT end_city
	FROM cities
	WHERE start_city != end_city)

SELECT city, COUNT(1) as cnt, ROUND(1.0*COUNT(1) / (SELECT COUNT(1) FROM trip), 4) AS ratio
FROM visited
GROUP BY city
ORDER BY ratio DESC, city;
```

먼저, station이 위치한 city와 조인이 필요함

###  Quiz 4

> ### Solution

    On Postgres 10.6 : 2 secs + @

``` sql
with visit(station_id, station_name, city, cnt) as (
    select station_id, station_name, city, count(id) as cnt 
    from trip, station 
    where station_id = start_station_id or station_id = end_station_id 
    group by station_id)

select visit.city, visit.station_name, visit.cnt 
from visit where visit.cnt = (select max(cnt) from visit as max_visit 
                              where max_visit.city = visit.city) 
order by city;
```

> ### My Query

    On Postgres 10.6 : 1 secs + @

``` sql
WITH stations(station_id) AS (
    SELECT start_station_id
    FROM trip
    UNION ALL
    SELECT end_station_id
    FROM trip
    WHERE start_station_id != end_station_id),
stations_count AS (
    SELECT st_info.station_name, st.station_id, st_info.city, COUNT(1) as cnt
    FROM stations st JOIN station st_info
    ON st.station_id = st_info.station_id
    GROUP BY st_info.station_name, st.station_id, st_info.city
)

SELECT city, station_name, cnt
FROM (SELECT *, 
      ROW_NUMBER() OVER (PARTITION BY city ORDER by cnt DESC) AS ranking
      FROM stations_count) a
WHERE ranking = 1;
```

### Quiz 7

> ### Solution

    On Postgres 10.6 : 6 secs + @

``` sql
select bike_id, count(distinct(city)) as cnt 
from trip, station 
where start_station_id = station_id or end_station_id = station_id 
group by bike_id 
having count(distinct(city)) > 1 
order by cnt desc, bike_id asc;
```

> ### My Query

    On Postgres 10.6 : 1 secs + @

``` sql
WITH stations(bike_id, station_id) AS (
    SELECT bike_id, start_station_id
    FROM trip
    UNION ALL
    SELECT bike_id, end_station_id
    FROM trip),
bikes_city(bike_id, visited) AS (
    SELECT st.bike_id , COUNT(DISTINCT city)
    FROM stations st JOIN station st_info
    ON st.station_id = st_info.station_id
    GROUP BY st.bike_id
)


SELECT *
FROM bikes_city
WHERE visited > 1
ORDER BY visited DESC, bike_id;
```

### Quiz 8

> ### Solution


```python
with weather_days(events, cnt) as (
    select w.events, count(distinct w.date) as cnt from weather w group by w.events
)
select w.events, round(cast(count(distinct t.id) as numeric)/cast(wd.cnt as numeric), 4) as avg 
from trip t, station s, weather w, weather_days wd
where t.start_station_id = s.station_id and s.zip_code = w.zip_code and date(t.start_time) = w.date and w.events = wd.events
group by w.events, wd.cnt
order by avg desc, w.events asc;
```
