# Business Questions and Insights

Below are business questions and their answers, including the most interesting database insights from the project.

---

## 1. Does seat belt use affect the number of road accident casualties?

Analyze the data on casualties and determine whether seat belt use reduces the severity of accidents. For each safety equipment category, calculate:
* The number of people involved in accidents
* The total number of injuries
* The total number of fatalities
* The percentage of injuries
* The percentage of fatalities

**Comparison:**
Then compare the results between the use of seat belts and the lack of seat belt use. Finally, indicate in which cases the lack of seat belt use most increases the risk of death in an accident.

### SQL Query
```sql
WITH Safety_Stats AS (
    SELECT 
        COUNT(*) AS Participant, 
        SUM(number_injured) AS Total_Injured, 
        SUM(number_killed) AS Total_Killed, 
        CONCAT(ROUND(SUM(number_injured)/NULLIF(COUNT(*),0)*100,2),'%') AS Injured_Ratio, 
        CONCAT(ROUND(SUM(number_killed)/NULLIF(COUNT(*),0)*100,2),'%') AS Killed_Ratio, 
        victim_safety_equip_1 
    FROM dim_victims_final 
    WHERE victim_safety_equip_1 = 'Lap/Shoulder Harness Not Used' 
       OR victim_safety_equip_1 = 'Lap/Shoulder Harness Used' 
    GROUP BY victim_safety_equip_1
)
SELECT * FROM Safety_Stats;
```
**SQL Results:**

<img width="722" height="73" alt="Ekren screen " src="https://github.com/user-attachments/assets/b377c843-99e8-4e3e-9ef0-890816e61958" />

<img width="292" height="57" alt="image" src="https://github.com/user-attachments/assets/e15e0623-4319-4b74-9f1a-caafb6298205" />


**Conlusions:**

Although almost every participant in both groups sustains some type of injury (over 99% of those injured), the key difference lies in the mortality rate:

Increased risk: The risk of death without a seat belt (0.73%) is over 10 times higher than with one (0.07%).

Do seat belts affect the number of casualties? Yes, primarily their final outcome. Seat belts don't prevent the accident itself, but they determine whether an injured passenger survives.
________________________________________


## 2. Which days of the week have the highest number of accidents and how their ranking changes from year to year

```sql
SELECT
year,
day,
Total_Accidents,
Ranking
FROM
(SELECT 
DENSE_RANK()OVER(PARTITION BY YEAR(dc.clean_date) ORDER BY COUNT(dc.case_id_pkey) DESC) AS Ranking,
COUNT(dc.case_id_pkey) as Total_Accidents,
YEAR(dc.clean_date) as year,
DAYNAME(dc.clean_date) as day
FROM dim_crashes dc
GROUP BY 
YEAR(dc.clean_date),DAYNAME(dc.clean_date)) tt
WHERE Ranking =1
ORDER BY year desc;
```

**SQL Results:**
<img width="352" height="411" alt="image" src="https://github.com/user-attachments/assets/4575de35-87d8-4a10-bb28-f3be87369892" />

**Conclusions:**

Friday is the most recurring "top-ranked" day for accidents, especially in recent years. It held the #1 ranking in:

Recent Period: 2024, 2025

Friday appears to be the most consistently dangerous day, likely due to increased traffic volume from commuters returning home and people starting weekend travel.
________________________________________

## 3. What are the top 3 times of day with the highest number of injuries and how has their ranking changed over time?

```sql
WITH TOP3_whole_range AS (
SELECT
DENSE_RANK()OVER(ORDER BY SUM(dv.number_injured) DESC) Ranking, 
SUM(dv.number_injured) as Totals,
dc.Time_Of_Day_Range
FROM dim_crashes dc
	 LEFT JOIN dim_victims_final dv
		ON dv.case_id_pkey=dc.case_id_pkey
GROUP BY dc.Time_Of_Day_Range
ORDER BY SUM(dv.number_injured) desc
),

Selection AS (
SELECT 
Time_Of_Day_Range,
Totals
FROM TOP3_whole_range
WHERE Ranking <=3),

Changes AS (
SELECT
DENSE_RANK()OVER(PARTITION BY YEAR(dc.clean_date) ORDER BY SUM(dv.number_injured) DESC) Ranking, 
SUM(dv.number_injured) as Total,
dc.Time_Of_Day_Range,
YEAR(dc.clean_date) as rok
FROM dim_crashes dc
	 LEFT JOIN dim_victims_final dv
		ON dv.case_id_pkey=dc.case_id_pkey
GROUP BY dc.Time_Of_Day_Range, YEAR(dc.clean_date)
ORDER BY SUM(dv.number_injured) desc
),

Prev_period AS (
SELECT
c.Time_Of_Day_Range,
c.rok,
c.Total,
c.Ranking,
LAG(Ranking) OVER (PARTITION BY Time_Of_Day_Range ORDER BY rok) AS PREV
FROM Changes c
	INNER JOIN Selection s
		ON c.Time_Of_Day_Range=s.Time_Of_Day_Range
ORDER BY rok desc)

SELECT 
Time_Of_Day_Range,
rok,
Total,
Ranking,
PREV,
CASE
	WHEN Ranking = PREV THEN "no change" ELSE "change"
END AS rank_diff
FROM Prev_period;
```
**SQL Results:**

<img width="496" height="415" alt="image" src="https://github.com/user-attachments/assets/855ded92-4e24-464c-9709-25539903ef31" />

**Conclusions:**

The table shows that the hazard hierarchy is very stable, but subject to change.

Afternoon: leader ("no change").

Noon vs. Morning: This is where the greatest competition occurs. In 2022 and 2023, the two "changes" occurred, suggesting that the morning accident rate in those years approached the afternoon level.
________________________________________


## 4.Which time of day and day of the week combinations are most dangerous in terms of the number of injuries?

```sql
WITH SUMIK AS (SELECT 
SUM(dv.number_injured) AS Total,
CONCAT(dc.time_of_day_range,' ','&',' ',DAYNAME(dc.clean_date)) AS Combination
FROM dim_crashes dc
	LEFT JOIN dim_victims_final dv
		ON dc.case_id_pkey=dv.case_id_pkey
GROUP BY 
CONCAT(dc.time_of_day_range,' ','&',' ',DAYNAME(dc.clean_date))
ORDER BY Total DESC),

Ranking AS (SELECT
Combination, 
DENSE_RANK() OVER(ORDER BY Total DESC) AS Ranks
FROM SUMIK)

SELECT *
FROM Ranking
WHERE Ranks <=3;
```
**SQL Results:**

<img width="252" height="91" alt="image" src="https://github.com/user-attachments/assets/59655609-045d-42ff-95a6-47e9e9b46c84" />

**Conclusions:**

Most Dangerous Combinations (Top 3)

Afternoon & Wednesday
Ranking: 1

Statistically, this is the riskiest time on the roads. The accumulation of the afternoon rush hour in the middle of the week results in the highest number of injuries.

Afternoon & Friday
Ranking: 2

Friday afternoon comes in second place. This is a particularly dangerous time due to the overlap between returning from work and weekend travel.

Afternoon & Tuesday
Ranking: 3

Tuesday afternoons round out the podium for the most accident-prone combinations.
________________________________________


## 5.Which vehicle types are most often involved in accidents resulting in death or serious injury?

```sql
WITH Agregate AS (SELECT
dp.stwd_vehicle_type,
dv.collision_severity,
SUM(dv.number_injured) AS Total_Injured,
SUM(dv.number_killed) AS Total_Killed
FROM dim_parties dp
inner JOIN dim_victims_final dv
ON dp.party_id=dv.party_id
WHERE dv.collision_severity IN ('Fatal','Injury (Severe)')
GROUP BY dp.stwd_vehicle_type,dv.collision_severity)

SELECT 
	stwd_vehicle_type,
    SUM(CASE 
			WHEN collision_severity = 'Injury (Severe)' THEN Total_Injured ELSE 0 END)AS Injury_Severe,
	SUM(CASE 
			WHEN collision_severity = 'Fatal' THEN Total_Killed ELSE 0 END)AS Fatal
	FROM Agregate
    GROUP BY stwd_vehicle_type;
```

**SQL Results:**

<img width="352" height="341" alt="image" src="https://github.com/user-attachments/assets/13782049-b6af-4e00-a3b5-dca1335aaef9" />


**Conclusions:**

Most At-Risk Groups (Top 3)
Pedestrian
Serious injuries: 1,805
Fatalities: 294
Conclusion: This is by far the most vulnerable group. The number of pedestrian deaths is more than six times higher than the next highest group.

Passenger Car
Serious injuries: 1,202
Fatalities: 49
Conclusion: The second largest group. Although the number of serious injuries is very high, passengers are protected by the bodywork, which is reflected in the significantly lower number of fatalities (relative to pedestrians).

Bicycle
Serious injuries: 834
Fatalities: 38
Conclusion: Cyclists constitute the third group in terms of the frequency of serious injuries.
________________________________________


## 6. Which party roles (driver, passenger, pedestrian) have the highest number of injured or killed in accidents?

```sql
WITH TopTotalInjuredPart AS (
SELECT 
dp.party_type,
SUM(COALESCE(dv.number_injured,0)) AS Total_Injured
FROM dim_parties dp
	LEFT JOIN dim_victims_final dv
		ON dp.party_id=dv.party_id
GROUP BY dp.party_type
ORDER BY Total_Injured DESC
LIMIT 1),

TopTotalKilledPart AS (
SELECT 
dp.party_type,
SUM(COALESCE(dv.number_killed,0)) AS Total_Killed
FROM dim_parties dp
	LEFT JOIN dim_victims_final dv
		ON dp.party_id=dv.party_id
GROUP BY dp.party_type
ORDER BY Total_Killed DESC
LIMIT 1)

SELECT 
    party_type, 
    Total_Killed AS Liczba, 
    'Killed' AS Statistic_Type
FROM TopTotalKilledPart

UNION ALL

SELECT 
    party_type, 
    Total_Injured AS Liczba,
    'Injured' AS Statistic_Type
FROM TopTotalInjuredPart;
```
**SQL Results:**

<img width="293" height="68" alt="image" src="https://github.com/user-attachments/assets/a0aa46fe-5436-40e9-97fb-b0ea1ddb070d" />

**Conclusions:**
Most Injured: Drivers. Because every vehicle has at least one driver, statistically they are most likely to be involved in accidents involving injuries.

Highest Fatality Rate (Killed): Pedestrians. Data shows that pedestrians are the most risky in terms of fatal accidents (confirmed by data on 294 fatalities, which is the dominant figure in fatality statistics).

________________________________________

## 7. Which 10 specific dates had the highest number of accidents throughout the period analyzed?

```sql
SELECT
	COUNT(DISTINCT dc.case_id_pkey) AS Total_Accidents,
    dd.date_day
FROM dim_crashes dc
	INNER JOIN dim_date dd
		ON dc.clean_date=dd.date_day
GROUP BY dd.date_day
ORDER BY COUNT(DISTINCT dc.case_id_pkey) DESC, dd.date_day ASC
LIMIT 10;
```
**SQL Results:**

<img width="237" height="242" alt="image" src="https://github.com/user-attachments/assets/e46af1f1-977d-4f68-9a70-e08e621aa646" />

**Conslucions:**

The most tragic day on the list was December 28, 2018 (23 accidents). It's worth noting that many of the dates on this list fall in the fall and winter months (October, November, December, January, and February), which may suggest that harsher weather conditions or holiday periods impact road safety.

________________________________________

## 8.How has the number of accidents involving drunk drivers changed over time?

```sql
WITH New_Sobriety AS (SELECT 
	case_id_pkey,
    party_sobriety,
    CASE 
		WHEN dp.party_sobriety LIKE 'Had Been Drinking%' THEN 'Had Been Drinking' ELSE 'Other sobriety' END New_party_sobriety
FROM dim_parties dp
WHERE party_type = 'Driver')

SELECT 
	COUNT(DISTINCT dc.case_id_pkey) AS Total_Accidents,
    dc.Year_Of_Crash,
    dns.New_party_sobriety
		FROM dim_crashes dc
			INNER JOIN New_Sobriety dns
				ON dc.case_id_pkey=dns.case_id_pkey
WHERE dns.New_party_sobriety = 'Had Been Drinking'
GROUP BY dc.Year_Of_Crash,dns.New_party_sobriety
ORDER BY dc.Year_Of_Crash DESC;
```

**SQL Results:**

<img width="403" height="371" alt="image" src="https://github.com/user-attachments/assets/37573883-7926-45a4-8f56-87906848b056" />

________________________________________

## 9.How many people were injured and how many died in accidents involving drunk drivers?Also provide the percentage of injured and killed victims in this group. Additionally, compare this to drivers who were sober.

```sql
WITH New_Sobriety AS (SELECT 
	party_id,
    party_sobriety,
    CASE 
		WHEN dp.party_sobriety LIKE 'Had Been Drinking%' THEN 'Had Been Drinking' ELSE 'Other sobriety' END New_party_sobriety_drinking,
	CASE 
		WHEN dp.party_sobriety LIKE 'Had Not Been Drinking%' THEN 'Had Not Been Drinking' ELSE 'Other sobriety' END New_party_sobriety_notdrinking
FROM dim_parties dp
WHERE party_type = 'Driver'),

Killed_Injured_Drinking AS (SELECT
	SUM(number_injured) AS Total_Injured,
    SUM(number_killed) AS Total_Killed
FROM dim_victims_final dvf
	LEFT JOIN New_Sobriety dns
		ON dvf.party_id=dns.party_id
WHERE New_party_sobriety_drinking = 'Had Been Drinking'), 

Killed_Injured_NotDrinking AS (SELECT
	SUM(number_injured) AS Total_Injured,
    SUM(number_killed) AS Total_Killed
FROM dim_victims_final dvf
	LEFT JOIN New_Sobriety dns
		ON dvf.party_id=dns.party_id
WHERE New_party_sobriety_notdrinking = 'Had Not Been Drinking')
	
SELECT
	'Had Been Drinking' AS Typ_Uczestnika,
	Total_Injured, 
    Total_Killed, 
    CONCAT(ROUND(Total_Injured/(Total_Injured+Total_Killed)*100,2),'%') AS Injured_Ratio,
    CONCAT(ROUND(Total_Killed/(Total_Injured+Total_Killed)*100,2),'%') AS Killed_Ratio
FROM Killed_Injured_Drinking

UNION ALL

SELECT
	'Had Not Been Drinking' AS Typ_Uczestnika,
	Total_Injured, 
    Total_Killed, 
    CONCAT(ROUND(Total_Injured/(Total_Injured+Total_Killed)*100,2),'%') AS Injured_Ratio,
    CONCAT(ROUND(Total_Killed/(Total_Injured+Total_Killed)*100,2),'%') AS Killed_Ratio
FROM Killed_Injured_NotDrinking;
```
**SQL Results:**

<img width="573" height="67" alt="image" src="https://github.com/user-attachments/assets/96d640ab-a2e4-4f05-b221-b238e8a42138" />

________________________________________

## 10.For each type of collision (e.g., rear-end, head-on, side-swipe, etc.), calculate: 
**- The number of accidents,**
**- The total number of injuries,**
**- The total number of fatalities,**
**- The average number of casualties per accident.**
**Finally, list the three types of collisions with the highest average number of casualties.**

```sql
SELECT 
	type_of_collision,
    COUNT(DISTINCT dc.case_id_pkey) AS Total_Accidents,
    SUM(dvf.number_injured) AS Total_Injured,
    SUM(dvf.number_killed) AS Total_Killed,
    SUM(dvf.number_injured)/COUNT(DISTINCT dc.case_id_pkey) AS Avg_Injured
FROM dim_crashes dc
	LEFT JOIN dim_parties dp 
		ON dc.case_id_pkey=dp.case_id_pkey
	LEFT JOIN dim_victims_final dvf
		ON dp.party_id=dvf.party_id
GROUP BY type_of_collision
ORDER BY SUM(dvf.number_injured)/COUNT(DISTINCT dc.case_id_pkey) DESC
LIMIT 3;

WITH victims_per_crash AS (
    SELECT
        case_id_pkey,
        SUM(number_injured) AS total_injured,
        SUM(number_killed) AS total_killed
    FROM dim_victims_final
    GROUP BY case_id_pkey
)

SELECT
    dc.type_of_collision,
    COUNT(DISTINCT dc.case_id_pkey) AS total_accidents,
    SUM(vpc.total_injured) AS total_injured,
    SUM(vpc.total_killed) AS total_killed,
    ROUND(
        (SUM(vpc.total_injured) + SUM(vpc.total_killed)) /
        COUNT(DISTINCT dc.case_id_pkey),
        2
    ) AS avg_victims_per_crash
FROM dim_crashes dc
LEFT JOIN victims_per_crash vpc
    ON dc.case_id_pkey = vpc.case_id_pkey
GROUP BY dc.type_of_collision
ORDER BY avg_victims_per_crash DESC
LIMIT 3;
```
**SQL Results:**

<img width="612" height="93" alt="image" src="https://github.com/user-attachments/assets/d4914c79-b1b2-4ba6-a8d3-3774a2911ab2" />
________________________________________


## 11. Do younger drivers cause accidents with more victims? Divide drivers into age groups (e.g., <25, 25–40, 41–60, 60+) and check:

**- How many accidents occurred in each group,**
**- How many total injuries and fatalities there were,**
**- What was the average number of victims per accident in each group.
Additionally, check whether the results differ between: male drivers and female drivers.**

```sql
WITH victims_per_crash AS (
    SELECT
        case_id_pkey,
        SUM(number_injured) AS total_injured,
        SUM(number_killed) AS total_killed
    FROM dim_victims_final
    GROUP BY case_id_pkey
),

drivers AS (
    SELECT
        case_id_pkey,
        party_sex,
        party_age,
        CASE
            WHEN party_age < 25 THEN '<25'
            WHEN party_age BETWEEN 25 AND 40 THEN '25-40'
            WHEN party_age BETWEEN 41 AND 60 THEN '41-60'
            ELSE '60+'
        END AS age_range
    FROM dim_parties
    WHERE party_type = 'Driver'
)

SELECT
    d.age_range,
    d.party_sex,
    COUNT(DISTINCT dc.case_id_pkey) AS total_accidents,
    SUM(vpc.total_injured) AS total_injured,
    SUM(vpc.total_killed) AS total_killed,
    ROUND(
        (SUM(vpc.total_injured) + SUM(vpc.total_killed)) /
        COUNT(DISTINCT dc.case_id_pkey),
        2
    ) AS avg_victims_per_crash
FROM dim_crashes dc
LEFT JOIN drivers d
    ON dc.case_id_pkey = d.case_id_pkey
LEFT JOIN victims_per_crash vpc
    ON dc.case_id_pkey = vpc.case_id_pkey
GROUP BY
    d.age_range,
    d.party_sex
ORDER BY
    avg_victims_per_crash DESC;
```
**SQL Results:**	

<img width="662" height="135" alt="image" src="https://github.com/user-attachments/assets/21d089c6-5dd9-4f14-831c-34fb8f5490ac" />




