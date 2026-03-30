# **🚗 San Francisco Crash Accidents – Data Analysis Project**

This project presents an end-to-end data analysis of traffic accidents in San Francisco using SQL and Power BI. It follows a full data analytics workflow, starting from raw data ingestion, through data cleaning, transformation and Inight Analysis in SQL, and ending with analytical reporting in Power BI.

The dataset was first loaded into a SQL database from source files, where the entire data preparation process was performed. This included creating database tables, validating data integrity, and cleaning inconsistent or missing records.

**👉 Data transformation logic and preparation steps can be found in the Preparing Data section within the SQL Queries folder.**

After the data quality checks and transformations were completed, the cleaned and structured tables were used to build a dimensional data model that supports efficient analytical queries.

**👉 Business-oriented analysis and key insights are documented in the BusinessAnalyticsResults.md file.**

For better readability of the SQL code used in the analysis, all queries are also organized separately in the SQL Queries → Business Analytics section.

The goal of the project was to analyze accident patterns, identify key risk factors, and demonstrate practical analytical skills required in a Data Analyst / BI Analyst role.

The final output is a Gold-layer warehouse optimized for analytical consumption and a Power BI dashboard presenting the results.

________________________________________


# 🎯 Project Motivation & Business Context

### **The Challenge: Turning Chaos into Insights** 🌪️
Raw traffic data from San Francisco is often **fragmented and inconsistent**, making direct analysis unreliable. In its initial state, the dataset consists of disconnected entities (accidents, parties, victims) stored in separate structures that are not ready for analytical use.

Before any meaningful analysis could take place, several critical hurdles had to be cleared:
* 📑 **Inconsistent Schemas** ⮕ Different structures across multiple datasets.
* 🔍 **Quality Gaps** ⮕ Limited guarantees regarding data integrity and completeness.
* 🔗 **Broken Relations** ⮕ Missing or inconsistent identifiers between linked tables.
* 🏗️ **No Unified Layer** ⮕ Absence of a consolidated, analytics-ready data foundation.

---

### **The Solution: A Robust Data Pipeline** 🛠️
This project serves as a **simulation of real-world analytical workflows**. It demonstrates the transition from messy, raw data to a verified, structured environment suitable for both executive reporting and deep-dive SQL queries.

**The project focuses on:**
1. **Simple Data Engineering** ⮕ Designing schemas, defining data types, and performing robust ingestion.
2. **SQL Transformation** ⮕ Using **MySQL** to identify gaps, enforce referential integrity, and clean the data.
3. **Business Intelligence** ⮕ Building an interactive **Power BI** layer for high-level visualization.
4. **Ad-hoc Analytics** ⮕ Creating a stable environment for rapid SQL queries from stakeholders or management.

---

### **Why This Matters?** 💡
> "Without proper data preparation in **MySQL**, any further analysis in **Power BI** would be flawed and misleading."

By solving these technical challenges, this project demonstrates a core suite of **Data Analyst / BI Analyst** competencies:
* ✅ **Mid SQL Level** (Data cleaning & transformation)
* ✅ **Relational Modeling** (Primary/Foreign Key architecture in PowerBi)
* ✅ **Analytical Thinking** (Translating raw rows into business KPIs)
* ✅ **Data Storytelling** (Interactive Dashboards)

---

### **Summary of Achievements** 🚀
* **Consolidated** multiple sources into a single, unified model.
* **Enforced** data quality and referential integrity across all tables.
* **Provided** a stable, high-performance foundation for BI reporting.
* **Automated** the path from "Raw Data" to "Actionable Insight".

________________________________________


# 🔧 Tools & Technologies 

**SQL**
* Data ingestion from raw files
* Data validation and cleaning
* Creation of relational tables
* Analytical queries|
* Joins, aggregations and window functions

**Power BI**
* Data modeling
* DAX calculations
* Interactive dashboards
* KPI and trend analysis

**Data Warehouse Concepts**
* Star schema modeling
* Dimensional tables
* Separation of fact and dimension layers
* Preparation of a Gold analytical layer

________________________________________


# 📊 Power BI Dashboard


* Built on Gold layer; supports filtering by time, KPI's, party type, race, sex and others crash and victims attribute
* Interactive exploration of trips with weather and event context
* Link to the entire 6-sheet report on GoogleDrive: [SanFrancisco_Crash_Accidents](https://drive.google.com/drive/folders/1J4xnV9rc0f1PreJpv8apeQcehdENsBfy?usp=sharing) - Due to the file size.

<img width="1562" height="736" alt="Sample of Project" src="https://github.com/user-attachments/assets/1e72cf23-ffdc-4244-bb5a-d8f286adbe05" />

________________________________________

# 📊 Some Example of Business Analytics

/**1. Does seat belt use affect the number of road accident casualties? Analyze the data on casualties and determine whether seat belt 
use reduces the severity of accidents. For each safety equipment category, calculate:
- the number of people involved in accidents
- the total number of injuries
- the total number of fatalities
- the percentage of injuries
- the percentage of fatalities

Then compare the results between:
- the use of seat belts
- the lack of seat belt use
Finally, indicate in which cases the lack of seat belt use most increases the risk of death in an accident.**/

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

<img width="722" height="73" alt="Ekren screen " src="https://github.com/user-attachments/assets/a1af321f-e6da-44c3-b85c-92389a5efe81" />

# 📥 Data Sources

•	SanFraniscoCrashes: [Traffic Crashes Resulting_SanFrancisco](https://data.sfgov.org/Public-Safety/Traffic-Crashes-Resulting-in-Injury/ubvf-ztfx/about_data)

________________________________________


# 📂 Repository Structure 
```text
SanFrancisco_Crash_Accidents/
│
├── SQL Queries/                       # List of Queries used during the work
│   ├── Queries - Business Analytics/  # Queries used during simulation of real work on a database with business questions
│   ├── Queries - Preparing Data/      # Queries used during simulation of real work related to creating tables, transforming data and preparing it for further analysis
│
└── README.md                          # Project overview and documentation



