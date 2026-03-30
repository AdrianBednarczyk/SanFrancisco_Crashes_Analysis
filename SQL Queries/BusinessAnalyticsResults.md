/** Some fast cases to solved to show skills in SQL **/

# 1. Does seat belt use affect the number of road accident casualties?

Analyze the data on casualties and determine whether seat belt 
use reduces the severity of accidents. For each safety equipment category, calculate:
- the number of people involved in accidents
- the total number of injuries
- the total number of fatalities
- the percentage of injuries
- the percentage of fatalities

Then compare the results between:
- the use of seat belts
- the lack of seat belt use
Finally, indicate in which cases the lack of seat belt use most increases the risk of death in an accident.

WITH Safety_Stats AS (SELECT
	COUNT(*) AS Participant,
    SUM(number_injured) AS Total_Injured,
    SUM(number_killed) AS Total_Killed,
    CONCAT(ROUND(SUM(number_injured)/NULLIF(COUNT(*),0)*100,2),'%') AS Injured_Ratio,
    CONCAT(ROUND(SUM(number_killed)/NULLIF(COUNT(*),0)*100,2),'%') AS Killed_Ratio,
	victim_safety_equip_1
    FROM dim_victims_final
	WHERE victim_safety_equip_1 = 'Lap/Shoulder Harness Not Used' OR victim_safety_equip_1 ='Lap/Shoulder Harness Used' 
    GROUP BY victim_safety_equip_1)

SELECT
MAX(CASE WHEN victim_safety_equip_1 = 'Lap/Shoulder Harness Not Used'
    THEN Killed_Ratio END) AS Not_UsedHarness_Ratio,
MAX(CASE WHEN victim_safety_equip_1 = 'Lap/Shoulder Harness Used'
    THEN Killed_Ratio END) AS Not_Used_Ratio
FROM Safety_Stats;

<img width="722" height="73" alt="Ekren screen " src="https://github.com/user-attachments/assets/b377c843-99e8-4e3e-9ef0-890816e61958" />

<img width="292" height="57" alt="image" src="https://github.com/user-attachments/assets/e15e0623-4319-4b74-9f1a-caafb6298205" />


________________________________________


#2. Which days of the week have the highest number of accidents and how their ranking changes from year to year#

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

<img width="352" height="411" alt="image" src="https://github.com/user-attachments/assets/4575de35-87d8-4a10-bb28-f3be87369892" />






**3. What are the top 3 times of day with the highest number of injuries and how has their ranking changed over time?**


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

<img width="496" height="415" alt="image" src="https://github.com/user-attachments/assets/855ded92-4e24-464c-9709-25539903ef31" />






**4.Which time of day and day of the week combinations are most dangerous in terms of the number of injuries?**


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

<img width="252" height="91" alt="image" src="https://github.com/user-attachments/assets/59655609-045d-42ff-95a6-47e9e9b46c84" />






**5.Which vehicle types are most often involved in accidents resulting in death or serious injury?**


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

<img width="352" height="341" alt="image" src="https://github.com/user-attachments/assets/13782049-b6af-4e00-a3b5-dca1335aaef9" />





**6. Which party roles (driver, passenger, pedestrian) have the highest number of injured or killed in accidents?**

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

<img width="293" height="68" alt="image" src="https://github.com/user-attachments/assets/a0aa46fe-5436-40e9-97fb-b0ea1ddb070d" />




**7. Which 10 specific dates had the highest number of accidents throughout the period analyzed?**

SELECT
	COUNT(DISTINCT dc.case_id_pkey) AS Total_Accidents,
    dd.date_day
FROM dim_crashes dc
	INNER JOIN dim_date dd
		ON dc.clean_date=dd.date_day
GROUP BY dd.date_day
ORDER BY COUNT(DISTINCT dc.case_id_pkey) DESC, dd.date_day ASC
LIMIT 10;

<img width="237" height="242" alt="image" src="https://github.com/user-attachments/assets/e46af1f1-977d-4f68-9a70-e08e621aa646" />




**8.How has the number of accidents involving drunk drivers changed over time?**

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

<img width="403" height="371" alt="image" src="https://github.com/user-attachments/assets/37573883-7926-45a4-8f56-87906848b056" />


/**9.How many people were injured and how many died in accidents involving drunk drivers?Also provide the percentage of injured 
and killed victims in this group. Additionally, compare this to drivers who were sober.**/

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

<img width="573" height="67" alt="image" src="https://github.com/user-attachments/assets/96d640ab-a2e4-4f05-b221-b238e8a42138" />

/**10.For each type of collision (e.g., rear-end, head-on, side-swipe, etc.), calculate:
- The number of accidents,
- The total number of injuries,
- The total number of fatalities,
- The average number of casualties per accident.
Finally, list the three types of collisions with the highest average number of casualties.**/

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

<img width="612" height="93" alt="image" src="https://github.com/user-attachments/assets/d4914c79-b1b2-4ba6-a8d3-3774a2911ab2" />

/**11. Do younger drivers cause accidents with more victims?
Divide drivers into age groups (e.g., <25, 25–40, 41–60, 60+) and check:
- How many accidents occurred in each group,
- How many total injuries and fatalities there were,
- What was the average number of victims per accident in each group.
Additionally, check whether the results differ between: male drivers and female drivers.**/

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

	
<img width="662" height="135" alt="image" src="https://github.com/user-attachments/assets/21d089c6-5dd9-4f14-831c-34fb8f5490ac" />




