# Data Analytics Challenge - 99
This repository contains the case resolution for job opportunity in 99

## Part I - Technical

First, I checked for duplicate IDs in the trip table, the answer is NO! So let's go.

### 1. What is the average trip cost of holidays? How does it compare to non-holidays?
~~~sql
SELECT
  AVG(CASE WHEN calendar.holiday = 1 THEN trip.trip_fare END) AS trip_fare_holiday,
  AVG(CASE WHEN calendar.holiday = 0 THEN trip.trip_fare END) AS trip_fare_non_holiday,
  ((AVG(CASE WHEN calendar.holiday = 1 THEN trip.trip_fare END)-
    AVG(CASE WHEN calendar.holiday = 0 THEN trip.trip_fare END))/
    AVG(CASE WHEN calendar.holiday = 0 THEN trip.trip_fare END))*100 AS percent_change
FROM trip 
LEFT JOIN calendar ON DATE(trip.call_time) = calendar.calendar_date
~~~

#### Answer: 

![image](https://user-images.githubusercontent.com/61919405/129113526-8ca82f1b-7454-4477-9d3c-5d4b0550973c.png)

### 2. Find the average call time of the first time passengers make a trip.
~~~sql
SELECT
  AVG(EXTRACT(EPOCH FROM (finish_time - call_time))/60) AS avg_first_call_time
FROM trip
JOIN passenger ON (trip.passenger_id = passenger.id AND passenger.first_call_time = trip.call_time)
~~~

#### Answer: **23.52 minutes**

### 3. Find the average number of trips per driver for every week day.
Here I have two differents approaches:
We have 2,318,357 diferents drivers, I think we have to see the average number of trips by driver one by one OR the average total. 
For the first approach:
~~~sql
SELECT
  calendar.week_day,
  COUNT(DISTINCT trip.id)/COUNT(DISTINCT trip.driver_id) AS trip_per_driver
FROM trip
JOIN calendar ON DATE(trip.call_time) = calendar.calendar_date
GROUP BY 1
~~~

#### Answer:

![image](https://user-images.githubusercontent.com/61919405/129116275-f2c19993-a285-442b-bdba-cdb055cf013f.png)

For the second one:
~~~sql
SELECT
  trip.driver_id,
  AVG(CASE WHEN calendar.week_day = 'Sunday' THEN count_trip END) AS avg_number_of_trips_sunday,
  AVG(CASE WHEN calendar.week_day = 'Monday' THEN count_trip END) AS avg_number_of_trips_monday,
  AVG(CASE WHEN calendar.week_day = 'Tuesday' THEN count_trip END) AS avg_number_of_trips_tuesday,
  AVG(CASE WHEN calendar.week_day = 'Wednesday' THEN count_trip END) AS avg_number_of_trips_wednesday,
  AVG(CASE WHEN calendar.week_day = 'Thursday' THEN count_trip END) AS avg_number_of_trips_thursday,
  AVG(CASE WHEN calendar.week_day = 'Friday' THEN count_trip END) AS avg_number_of_trips_friday,
  AVG(CASE WHEN calendar.week_day = 'Saturday' THEN count_trip END) AS avg_number_of_trips_saturday
FROM (
    SELECT driver_id, DATE(trip.call_time) AS call_time, COUNT(id) AS count_trip 
    FROM trip GROUP BY 1, 2) AS trip
JOIN calendar ON DATE(trip.call_time) = calendar.calendar_date
GROUP BY 1
~~~

### 4. Which day of the week drivers usually drive the most distance on average?
~~~sql
SELECT
  calendar.week_day,
  AVG(trip.trip_distance) AS avg_trip_distance
FROM trip
JOIN calendar ON DATE(trip.call_time) = calendar.calendar_date
GROUP BY 1
ORDER BY 2 DESC
~~~

#### Answer:

![image](https://user-images.githubusercontent.com/61919405/129116600-27705a09-d153-454e-b175-56a06aeb7cbc.png)

### 5. What was the growth percentage of rides month over month?

~~~sql
WITH trips_per_month AS
(SELECT 
  year_month,
  COUNT(id) AS num_of_trips
FROM (SELECT DATE(DATE_TRUNC('month', call_time)) AS year_month, id FROM trip) AS trip
GROUP BY 1)

SELECT 
  year_month,
  num_of_trips,
  ROUND(((num_of_trips - CAST(LAG(num_of_trips, 1) OVER (ORDER BY year_month) AS decimal)) / CAST(LAG(num_of_trips, 1) OVER (ORDER BY year_month) AS decimal))*100, 2) AS perc_growth
FROM trips_per_month
~~~

#### Answer:

![image](https://user-images.githubusercontent.com/61919405/129116713-38073f98-2980-4aba-a394-d26aa733fc64.png)
Obs.: September is not finished. 


### 6. Optional. List the top 5 drivers per number of trips in the top 5 largest cities.

Here, I consider the five largest cities are the cities with greater average travelled distance.

~~~sql
WITH cities AS (SELECT
  city_id,
  city.name,
  RANK() OVER (ORDER BY AVG(trip_distance) DESC) AS rank
FROM trip
JOIN city ON CAST(trip.city_id AS bigint) = city.id
GROUP BY 1, 2 ORDER BY 3
),

drivers AS
(SELECT
  driver_id,
  COUNT(trip.id) AS number_of_trip,
  RANK() OVER (ORDER BY COUNT(trip.id) DESC) AS rank
FROM trip
JOIN cities ON trip.city_id = cities.city_id
WHERE rank <= 5
GROUP BY 1
)

SELECT
  rank,
  driver_id,
  number_of_trip
FROM drivers WHERE rank <= 5
ORDER BY 1
~~~

#### Answer:

![image](https://user-images.githubusercontent.com/61919405/129117305-a79898fb-2a68-4298-b927-38890f88dac7.png)


## Part II - Analytical

### 1. Let's say it's 2019-09-23 and a new Operations manager for The Shire was just hired. She has 5 minutes during the Ops weekly meeting to present an overview of the business in the city, and since she's just arrived, she asked your help to do it. What would you prepare for this 5 minutes presentation?
[Weekly Follow-up Presentation](https://github.com/priscilaborges/data_analytics_challenge_99/blob/main/OPs%20Weekly.pptx)


### 2. She also mentioned she has a budget to invest in promoting the business. What kind of metrics and performance indicators would you use in order to help her decide if she should invest it into the passenger side or the driver side? Extra point if you provide data-backed recommendations.

Following the number of passenger by week to invest in **bonus weekly per user**.
~~~sql
WITH trips_per_month AS
(SELECT 
  week,
  count(distinct passenger_id) AS passenger_id
FROM (
    SELECT DATE(DATE_TRUNC('week', call_time)) AS week, passenger_id FROM trip
    JOIN city ON CAST(trip.city_id AS bigint) = city.id
WHERE city.name = 'The Shire') AS trip
GROUP BY 1)

SELECT 
  week,
  passenger_id,
  ((passenger_id - CAST(LAG(passenger_id, 1) OVER (ORDER BY week) AS decimal)) / CAST(LAG(passenger_id, 1) OVER (ORDER BY week) AS decimal))*100 AS perc_growth
FROM trips_per_month
~~~

And follow the **average of trips by driver**: here we can offer benefits and bonus for the firts 10 drivers with higher numbers of trips per week.

~~~sql
WITH trips_per_month AS
(SELECT 
  week,
  avg(distinct sum_trip) AS trip
FROM (
    SELECT DATE(DATE_TRUNC('week', call_time)) AS week, driver_id, count(trip.id) as sum_trip FROM trip
    JOIN city ON CAST(trip.city_id AS bigint) = city.id
WHERE city.name = 'The Shire' group by 1, 2) AS trip
GROUP BY 1)

SELECT 
  week,
  trip,
  ((trip - CAST(LAG(trip, 1) OVER (ORDER BY week) AS decimal)) / CAST(LAG(trip, 1) OVER ( ORDER BY week) AS decimal))*100 AS perc_growth
FROM trips_per_month
~~~

### 3. One month later, she comes back, super grateful for all the helpful insights you have given her. And says she is anticipating a driver supply shortage due to a major concert that is going to take place the next day and also a 3 day city holiday that is coming the next month. What would you do to help her analyze the best course of action to either prevent or minimize the problem in each case?

In this case, I would like to analyze if there is historic where we obtained a major concert day and holidays, then we could compare the number of calls/trips and prepare thinking about price increase and bonus to available drivers.

### Optional. We want to build up a model to predict “Possible Churn Users” (e.g.: no trips in the past 4 weeks). List all features that you can think about and the data mining or machine learning model or other methods you may use for this case.

Logistic regression is a statistical analysis method that could to predict "possible churn users". We would need a data with churn users to train the model and some variables like
number of trips by week per user, trip fare, waiting time, first call time, last call time.
 
