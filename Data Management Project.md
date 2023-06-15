## Data Management Tasks

### Task 1: 
    Improve the request coverage in the fact table following the hints in the previous section and extending those with your own ideas. 
Because of the data type, the incident, zip, and agency data are trimmed to increase coverage.

In order to improve the coverage, first, we will use the view creation method, and put the data obtained by filtering with regular expressions, wildcards, etc., in the view, and the subsequent operations will be done by manipulating the corresponding view such as adding new definition description fields, querying, etc. to get the required data. This job uses a 32M data table.

- Create Views
The SQL statements used are as follows:
```SQL
CREATE OR REPLACE VIEW FACT_SERVICE_QUALITY_SANITISED AS
SELECT
"Unique_Key",
"Created_Date",
"Closed_Date",
"Agency",
UPPER(REGEXP_REPLACE("Agency", '[^[:alnum:]]+', '')) AS "Agency_Sanitised",
"Agency_Name",
"Complaint_Type",
UPPER(REGEXP_REPLACE("Complaint_Type", '[^[:alnum:]]+', '')) AS "Complaint_Type_Sanitised",
"Descriptor",
"Location_Type",
"Incident_Zip",
UPPER(REGEXP_REPLACE("Incident_Zip", '[^[:alnum:]]+', '')) AS "Incident_Zip_Sanitised",
"Incident_Address",
"City",
UPPER(REGEXP_REPLACE("City", '[^[:alnum:]]+', '')) AS "City_Sanitised",
"Status",
"Due_Date",
"Resolution_Description",
"Resolution_Action_Updated_Date",
"Community_Board",
BBL,
"Borough",
"X_Coordinate_(State Plane)",
"Y_Coordinate_(State Plane)",
"Open_Data_Channel_Type",
"Location"
FROM NYC311.SERVICE_REQUEST_32M;
```

The new fields in this view are Agency_Sanitised, Complaint_Type_Sanitised and Incident_Zip_Sanitised, the data type of these fields are characters, so they are prone to case and special character entry problems, so they are trimmed in the view, and the created_weak used for Created_Date and Closed_Date, which are used to calculate year_weak, are of type TIMESTAMP_NTZ(0), so there is little chance of problems, so they are not handled. In subsequent tasks, this view structure will be modified as needed to get the required data without modifying the original data.

The result of the query with the trimmed data:
```SQL
SELECT COUNT(DISTINCT sqs."Complaint_Type_Sanitised") AS CTS, 
       COUNT(DISTINCT sqs."Incident_Zip_Sanitised") AS IZS,
       COUNT(DISTINCT sqs."Complaint_Type") AS CT, 
       COUNT(DISTINCT sqs."Incident_Zip") AS IZ
FROM FACT_SERVICE_QUALITY_SANITISED sqs;
```
![task1_count](https://github.com/bluerpacific/DataManagement/blob/main/task1_count.png)

From the graph, we can see that the amount of data in distinct is reduced after trimming, which means that there is indeed data in the original data due to character input differences, which can improve the query coverage to some extent.


### Task2:
    Write a query to identify request types that are recorded in the dataset as handled by more than one agency. You can use either the original 32 M table or the fact service quality table for this purpose. 

```SQL
SELECT "type"
FROM FACT_SERVICE_QUALITY SQ
JOIN DIMENSION_COMPLAINT_TYPE DC ON SQ."type_id" = DC."type_id"
GROUP BY "type"
HAVING COUNT(DISTINCT "agency_id") > 1;
```
![task2_type](https://github.com/bluerpacific/DataManagement/blob/main/task2_type.png)
The answer is obtained by querying the number of agencies corresponding to different types.

### Task3:
     Write a query and produce a convincing chart/visualisation to show which NYC agencies are improving their service quality (based on faster request processing times) over time (months and years). Yes, you can modify the fact service quality table to include new columns. 

- Add a new field to the FACT_SERVICE_QUALITY table
```SQL
"year_month" VARCHAR(7),
```

- Then, in the update data statement, add the following statement:
```SQL
DATE_PART('year', "Created_Date") * 100 + DATE_PART('month', "Created_Date") AS "year_month",
...
GROUP BY DA."agency_id", DZ.ZIP, DC."type_id", DT."year_week", "year_month";
```
This converts the time of each record into the month of the year for storage.
![task3_insert](https://github.com/bluerpacific/DataManagement/blob/main/task3_insert.png)

- Query the number of cases per month and the average processing time per month for each organization, where the average processing time is calculated as SUM(average_days * total_count) / SUM(total_count).

- Query the number of cases per agency for each month
```SQL
SELECT "agency_id", "year_month", SUM(case_total) AS case_total
FROM (
    SELECT SQ."agency_id", SQ."year_month", COUNT(*) AS case_total
    FROM FACT_SERVICE_QUALITY SQ
    GROUP BY SQ."agency_id", SQ."year_month"
) subquery
GROUP BY "agency_id", "year_month"
ORDER BY "year_month" ASC;
```
- Query the average processing time of each agency
```SQL
SELECT SQ."agency_id", SQ."year_month", SUM(SQ."average_days" * SQ."total_count") / SUM(SQ."total_count") AS avg_time
FROM FACT_SERVICE_QUALITY SQ
GROUP BY SQ."agency_id", SQ."year_month"
ORDER BY SQ."agency_id" ASC, "year_month" ASC;
```
- Take agency 1023 as an example, show the number of cases, monthly average processing time and visualize the data.
```SQL
SELECT "agency_id", "year_month", SUM(case_total) AS case_total
FROM (
    SELECT SQ."agency_id", SQ."year_month", COUNT(*) AS case_total
    FROM FACT_SERVICE_QUALITY SQ
    WHERE SQ."agency_id" = '1023'
    GROUP BY SQ."agency_id", SQ."year_month"
) subquery
GROUP BY "agency_id", "year_month"
ORDER BY "year_month" ASC;

SELECT SQ."agency_id", SQ."year_month", SUM(SQ."average_days" * SQ."total_count") / SUM(SQ."total_count") AS avg_time
FROM FACT_SERVICE_QUALITY SQ
WHERE SQ."agency_id" = '1023'
GROUP BY SQ."agency_id", SQ."year_month"
ORDER BY SQ."agency_id" ASC, "year_month" ASC;
```

- Use Python to read the csv data and visualize it
![task3_case_total_in_month](https://github.com/bluerpacific/DataManagement/blob/main/task3_count_saccter.png)
![task3_avg_time_in_month](https://github.com/bluerpacific/DataManagement/blob/main/task3_time_line.png)

### Task4:
    Write a query and produce a convincing chart/visualisation to show which NYC boroughs may be functioning better than others and are improving over time. You may use a subset of agencies for this purpose, choosing only a few criteria. 

```SQL
WITH zip_max_total_count AS (
    SELECT ZIP, COUNT(*) AS total_count
    FROM FACT_SERVICE_QUALITY FQ
    JOIN DIMENSION_YEAR_WEEK DT ON FQ."year_week" = DT."year_week"
    GROUP BY ZIP
    ORDER BY total_count DESC
    LIMIT 1
)
SELECT DT."year_week", COUNT(*) AS total_count, AVG(FQ."average_days") AS avg_days, STDDEV(FQ."average_days") AS stddev_days
FROM FACT_SERVICE_QUALITY FQ
JOIN DIMENSION_YEAR_WEEK DT ON FQ."year_week" = DT."year_week"
JOIN zip_max_total_count ZMTC ON FQ.ZIP = ZMTC.ZIP
GROUP BY DT."year_week"
ORDER BY DT."year_week" ASC;
```
The sample data obtained is as follows:
![task4_query](https://github.com/bluerpacific/DataManagement/blob/main/task4_query.png)

The ZIP with the highest number of cases in 2015 is used as an example, and the query criteria is updated to restrict the query to data between 2015 and 2016.
```SQL
WHERE DT."year_week" between 201500 and 201600
```

The visualization chart is as follows:
![task4_2015](https://github.com/bluerpacific/DataManagement/blob/main/task4_2015.png)

From the graph, we can see that in 2015, the number of cases in this ZIP is decreasing as the year_week changes, but from the mean of the processing time and the standard deviation of the processing time, the mean of the processing time reflects the service quality of the agency in a region, but at the same time, the standard deviation of the processing time fluctuates greatly, so the service quality of the region in this year does not change with time. Therefore, the service quality of the region in this year did not become better with time.
