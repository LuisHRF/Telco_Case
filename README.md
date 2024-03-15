## **Telco Case Study: Why are we losing clients?**
A fictitious case where a telecom company called Telco suffers a drop in customers after joining another larger company.


First of all: I have used a fictitious project case that I found on Kagle. Thank you for making it public so that people like me who are learning can practice. You can find the link [here](https://www.kaggle.com/datasets/datacertlaboratoria/sql-proyecto-2-prdida-de-clientes-en-telco/data).


## Scenario
As a data analyst for the telecommunications company TELCO, the company's CEO, Amalia, has asked us to conduct a data study to determine the possible reasons why we are losing customers. This is because the current CTO, Sebastian, has his own hypothesis: 1) the decrease in customers is due to an increase in competition 2) The problem lies in month-to-month contracts, wanting to convert them into long-therm contratcs.

This is why Amalia has asked us to: 1) carry out a general study of the clients (gender, marital status, etc.) 2) check Sebastián's hypothesis by checking the status of clients with monthly contracts 3) look for common points of "high flight risk" clients

## What am I going to use?

I'm going to use BigQuery, Google Cloud Platform's SQL tool. This is because I want to continue practicing my skill with this tool and improve how I work in SQL.

## First steps

The first thing was to download the **.csv** files that the company provided us and upload them to the BigQuery project that I created (called data_telco). I leave here the name of each file and how I have saved each table in BigQuery:

> - Telco_customer_churn_location.csv = churn_locaiton
> - Telco_customer_churn_population.csv = churn_population
> - Telco_customer_churn_services.csv = churn_services
> - Telco_customer_churn_status.csv = churn_status

The first thing was to check that all the tables were uploaded correctly with simple queries like the following:

```sql
SELECT
Customer_ID,
Country,
State,
Married,
Gender,
FROM `lost-clients-portfolio.data_telco.churn_location`
LIMIT 6
```

After making sure that the tables had been uploaded correctly, I created a fifth table with the information from the previous ones. I have done this to be able to work more easily and with a single source of information:

```sql
CREATE TABLE `lost-clients-portfolio.data_telco.tabla`  AS (
SELECT a.* EXCEPT(Zip_Code),
        b.Gender,b.Age,b.Under_30,b.Senior_Citizen,
        b.Married,b.Dependents,b.Number_of_Dependents,
        c.Quarter,c.Referred_a_Friend,c.Number_of_Referrals,
        c.Tenure_in_Months,c.Offer,c.Phone_Service,
        c.Avg_Monthly_Long_Distance_Charges,c.Multiple_Lines,
        c.Internet_Service,c.Internet_Type,c.Avg_Monthly_GB_Download,
        c.Online_Security,c.Online_Backup,c.Device_Protection_Plan,
        c.Premium_Tech_Support,c.Streaming_TV,c.Streaming_Movies,
        c.Streaming_Music,c.Unlimited_Data,c.Contract,c.Paperless_Billing,
        c.Payment_Method,c.Monthly_Charge,c.Total_Charges,c.Total_Refunds,
        c.Total_Extra_Data_Charges,c.Total_Long_Distance_Charges,c.Total_Revenue,
        d.Customer_Status,d.Churn_Label,d.Churn_Value,
        d.Churn_Category,d.Churn_Reason 
FROM `lost-clients-portfolio.data_telco.churn_location` a
LEFT JOIN `lost-clients-portfolio.data_telco.churn_demographics` b
ON a.Customer_ID=b.Customer_ID
LEFT JOIN `lost-clients-portfolio.data_telco.churn_services` c
ON b.Customer_ID=c.Customer_ID
LEFT JOIN `lost-clients-portfolio.data_telco.churn_status` d
ON c.Customer_ID=d.Customer_ID
)
```

Once the table has been created, we check that the number of rows is appropriate (7043) using **COUNT**.

```sql
SELECT
  COUNT(*)
FROM `lost-clients-portfolio.data_telco.table` a
LIMIT 10
```

From now on, I check some data to see possible anomalous data and also get some general information:

> 1. The minimum (19), maximum (19) and average age of the clients (47.46). I find an anomalous data in the maximum age (119), it is saved to clean later.

```sql
SELECT
MIN(Age) AS min_age,
MAX(Age) AS max_age,
AVG(Age) AS avg_age
FROM `lost-clients-portfolio.data_telco.tabla`
```

> 2. Based on the key **Monthly_Charge**, I get the minimum (-10), maximum (118.75) and average (65.60). In this way I obtain the data on how much is charged to clients monthly; and again, I get an abnormal data of -10.

```sql
SELECT
MIN(Monthly_Charge) AS min_rev,
MAX(Monthly_Charge) AS max_rev,
AVG(Monthly_Charge) AS avg_rev
FROM `lost-clients-portfolio.data_telco.tabla`
```

> 3. I also list the five cities with the highest number of clients: Los Angeles (283), San Diego (275), San José (108), Sacramento (106) and San Francisco (102).

```sql
SELECT 
City, 
COUNT(Customer_ID) AS clientes_totales
FROM `lost-clients-portfolio.data_telco.tabla`
GROUP BY 1
ORDER BY 2 DESC 
LIMIT 5
```

> 4. I collect client data by gender and marital status (married or single). When I run the query, I realize that the data has been misspelled as there are six four genders: Male, M, Female and F. That is, some data has been named wrong.

```sql
SELECT 
 Gender, 
 Married,
  COUNT(Gender) as gender_clients,
  COUNT(Married) as status_clients
FROM `lost-clients-portfolio.data_telco.tabla`
GROUP BY 
 Gender,
 Married
```

After these queries, I decide to update the table by cleaning the data. First of all, I indicate that the 'M' is Male and the 'F' is Female. Secondly, I indicate that the age must be less than or equal to 90 years. And thirdly, that the monthly charges must always be greater than 0.

```sql
CREATE OR REPLACE TABLE 
`lost-clients-portfolio.data_telco.tabla` AS (
SELECT 
* EXCEPT(gender),
CASE
WHEN gender = 'M' THEN 'Male'
WHEN gender = 'F' THEN 'Female'
ELSE gender END as gender
FROM `lost-clients-portfolio.data_telco.tabla`
WHERE Age<=90
AND Monthly_Charge > 0
)
```

