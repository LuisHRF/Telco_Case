## **Telco Case Study: Why are we losing clients?**
A fictitious case where a telecom company called Telco suffers a drop in customers after joining another larger company.


First of all: I have used a fictitious project case that I found on Kagle. Thank you for making it public so that people like me who are learning can practice. You can find the link [here](https://www.kaggle.com/datasets/datacertlaboratoria/sql-proyecto-2-prdida-de-clientes-en-telco/data).


## Scenario
As a data analyst for the telecommunications company TELCO, the company's CEO, Amalia, has asked us to conduct a data study to determine the possible reasons why we are losing customers. This is because the current CTO, Sebastian, has his own hypothesis: 1) the decrease in customers is due to an increase in competition 2) The problem lies in month-to-month contracts, wanting to convert them into long-therm contratcs.

This is why Amalia has asked us to: 1) carry out a general study of the clients (gender, marital status, etc.) 2) check Sebastián's hypothesis by checking the status of clients with monthly contracts 3) look for common points of "high flight risk" clients

## What am I going to use?

I'm going to use BigQuery, Google Cloud Platform's SQL tool. This is because I want to continue practicing my skill with this tool and improve how I work in SQL.

## Transforming and cleaning data

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
CREATE TABLE `lost-clients-portfolio.data_telco.table_general`  AS (
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
FROM `lost-clients-portfolio.data_telco.table_general` a
LIMIT 10
```

From now on, I check some data to see possible anomalous data and also get some general information:

> 1. The minimum (19), maximum (19) and average age of the clients (47.46). I find an anomalous data in the maximum age (119), it is saved to clean later.

```sql
SELECT
MIN(Age) AS min_age,
MAX(Age) AS max_age,
AVG(Age) AS avg_age
FROM `lost-clients-portfolio.data_telco.table_general`
```

> 2. Based on the key **Monthly_Charge**, I get the minimum (-10), maximum (118.75) and average (65.60). In this way I obtain the data on how much is charged to clients monthly; and again, I get an anormal data of -10.

```sql
SELECT
MIN(Monthly_Charge) AS min_rev,
MAX(Monthly_Charge) AS max_rev,
AVG(Monthly_Charge) AS avg_rev
FROM `lost-clients-portfolio.data_telco.table_general`
```

> 3. I also list the five cities with the highest number of clients: Los Angeles (283), San Diego (275), San José (108), Sacramento (106) and San Francisco (102).

```sql
SELECT 
City, 
COUNT(Customer_ID) AS clientes_totales
FROM `lost-clients-portfolio.data_telco.table_general`
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
FROM `lost-clients-portfolio.data_telco.table_general`
GROUP BY 
 Gender,
 Married
```

After these queries, I decide to update the table by cleaning the data. First of all, I indicate that the 'M' is Male and the 'F' is Female. Secondly, I indicate that the age must be less than or equal to 90 years. And thirdly, that the monthly charges must always be greater than 0.

```sql
CREATE OR REPLACE TABLE 
`lost-clients-portfolio.data_telco.table_general` AS (
SELECT 
* EXCEPT(gender),
CASE
WHEN gender = 'M' THEN 'Male'
WHEN gender = 'F' THEN 'Female'
ELSE gender END as gender
FROM `lost-clients-portfolio.data_telco.table_general`
WHERE Age<=90
AND Monthly_Charge > 0
)
```

## Analysis

From this moment on, once the table is clean and I have verified that it is functional, I began to create visualizations of the most notable data for analysis.

The first thing is to create some new variables that I think are important in order to perform the analysis and share the data:

> 1. A range of ages: This will help me group clients in a cleaner way. Also, my "boss" (Sebastián) talked about age ranges, so it is a way to start working with data that superiors need.

```sql
CREATE OR REPLACE TABLE `lost-clients-portfolio.data_telco.table_general`AS (
 SELECT *,
    CASE 
    WHEN Age < 21 THEN '0 to 20 years'
    WHEN Age BETWEEN 21 AND 60 THEN '21 to 60 years' 
    ELSE 'More than 61 years' 
    END AS age_range
    FROM `lost-clients-portfolio.data_telco.table_general`)
```

> 2. A range of "referrals": One of Amalia's requests was to check the statistics of the number of referrals in relation to marital status. So I made a query to be able to extract this data from **Number_of_Referrals**.

```sql
CREATE OR REPLACE TABLE `lost-clients-portfolio.data_telco.table_general` AS (
 SELECT *,
    CASE 
    WHEN Number_of_Referrals = 0 THEN 'Any referral'
    WHEN Number_of_Referrals BETWEEN 1 AND 4 THEN '1 to 4 referral' 
    WHEN Number_of_Referrals BETWEEN 5 AND 8 THEN '5 to 8 referral' 
    ELSE 'More than 8 referrals' 
    END AS referrals_range
    FROM `lost-clients-portfolio.data_telco.table_general`
)
```

> 3. Groupings by "risk of drain" and avg_antiquity: Sebastián's big concern was that clients who contracted the monthly modality (Month-to-Month) were the ones with the greatest risk of flight. This is why I establish, first of all, four groups according to Sebastián's hypothesis:

> G1 (very high risk group): over 64 years old with a monthly contract; G2 (high risk group): those under 64 years of age with a monthly contract with one or more referrals; G3 (medium risk group): over 64 years of age with another type of contract (non-monthly) and one or more referrals; and G4 (low risk group): clients with less than 40 months of membership and non-monthly contracts.

> At this point I had a small problem: when I created the table, the rest of the variables were eliminated. This is why in this code I applied **EXCEPT()** to the variables I had in mind.


```sql
CREATE OR REPLACE  TABLE `lost-clients-portfolio.data_telco.table_general` AS (
SELECT
a.* EXCEPT(Customer_ID, Contract, Churn_Value, Total_Revenue),
a.Customer_ID,
a.Contract,
a.Churn_Value,a.Total_Revenue,
       CASE WHEN a.Contract = 'Month-to-Month' AND a.Age > 64 THEN 'G1'  
       ELSE
       CASE WHEN a.Contract = 'Month-to-Month' AND a.Age < 64 AND a.Number_of_Referrals <= 1 THEN 'G2' 
       ELSE 
       CASE  WHEN a.Contract != 'Month-to-Month' AND a.Age > 64 AND a.Number_of_Referrals <= 1 THEN 'G3'  
       ELSE 
       CASE  WHEN a.Contract != 'Month-to-Month' AND a.Tenure_in_Months < 40 THEN 'G4' 
       ELSE
       'Not risk_group' END END END END  AS risk_group
FROM `lost-clients-portfolio.data_telco.table_general` a
)
```
> At the same time, I made a query to be able to create a variable that would refer to the months that clients have been with the company. This way, I used the **Tenure_in_Months** variable to create a per-client average called **avg_antiquity**.

```sql
CREATE OR REPLACE  TABLE `lost-clients-portfolio.data_telco.table_general` AS(
WITH antiquity_base AS (
        SELECT Contract,
        AVG(Tenure_in_Months) AS avg_antiquity
        FROM `lost-clients-portfolio.data_telco.table_general`
        GROUP BY 1
)
SELECT a.*,
        b.avg_antiquity
FROM `lost-clients-portfolio.data_telco.table_general` a
LEFT JOIN antiquity_base b
ON a.Contract = b.Contract
)
```
> 4. Some operations: Here I already took advantage of the new variables to extract some data. The first of them was the estimated historical payment per client (**estimated_pay**), multiplying the average seniority and the total monthly revenue.

```sql
CREATE OR REPLACE  TABLE `lost-clients-portfolio.data_telco.table_general` AS(
SELECT a.*,
a.avg_antiquity*Total_Revenue/3 AS estimated_pay
FROM `lost-clients-portfolio.data_telco.table_general` a
)
```

> I also made an estimate through quartiles to better organize the risk groups. In this case, I used an operator that I had already used in a previous exercise, I just substituted some variables.

```sql
CREATE OR REPLACE TABLE `lost-clients-portfolio.data_telco.table_general` AS (
    SELECT *,NTILE(4) OVER( PARTITION BY Contract 
                    ORDER BY estimated_income ASC) AS estimated_quartile
    FROM  `lost-clients-portfolio.data_telco.tabla`
    WHERE Churn_Value=0
)
```

##  Dataviz using Power BI

Now I will share the codes of the visualizations made with Power BI (I will upload the images of these over time, since now I still do not understand very well how to do it) and the results of the data analysis of the Telco case.

> 1. **General Issues**: The first thing was to create visualizations that show an overview of customers. **image 1** corresponds to a graph that shows the five cities with the highest number of clients:

![](https://github.com/LuisHRF/Telco_Case/blob/main/cities_clients.PNG)

> At the same time, I have made a bar graph that combines the number of clients according to their marital status (single or married) and the total number of referrals by city. In this way, we see that the majority of clients are still married couples in most cities.

![](https://github.com/LuisHRF/Telco_Case/blob/main/Clients_By_Married_And_City%C3%A7.PNG)

> 2. **Sebastian's hypothesis**: Now I have to test Sebastian's hypothesis with data to see if it can really be an indication of customer loss. Let us remember that the CTO argued that the loss of clients was due, in addition to competition, to month-to-month contracts, which were the ones that retained the smallest number of clients.
>
> The first graph you create is one that shows the number of customers based on the type of contract they have (month-to-month, one year, or two years) with the average tenure. This will show which customers tend to stay with the company the longest. As can be seen, monthly customers have a much lower average retention rate than the rest: 20 months compared to 54 for bi-annual contracts and 40 for annual contracts.

![](https://github.com/LuisHRF/Telco_Case/blob/main/Clients_By_Contract_Antiquity.PNG)

> The next step I have followed is to review the risk groups. To remember, the risk groups are divided into four: G1, G2, G3 and G4; being from highest to lowest risk. The visualization shows the percentage of membership in these risk groups according to the age of the client, so that we determine which age range is the most "fleeting".

> As can be seen, the clients most at risk of unsubscribing are those over 64 years of age.

![](https://github.com/LuisHRF/Telco_Case/blob/main/risk_by_age.PNG)

> From these two graphs, I have created a third that brings together the number of clients by age range with the type of contract and their risk of unsubscribing. Again, this shows that the only age range that belongs to G1 is the month-to-month customer over 64 years old. At the same time, monthly clients are the ones most at risk of absconding.

![](https://github.com/LuisHRF/Telco_Case/blob/main/risk_by_contract_age.PNG)

> Finally, I created a graph that shows the specific number of clients at risk of churn by contract type. In this way it can be seen visually and very quickly that the majority of clients at risk are in monthly contracts, while annual and bi-annual contracts are much safer.

![](https://github.com/LuisHRF/Telco_Case/blob/main/risk_lvl_clients.PNG)

## Analysis and proposal

Sebastián's hypothesis is demonstrated, therefore we can determine that:

- The majority of clients are between month-to-month and two-year contracts
- The greatest risk of "leakage" is found in monthly contracts, with 73% less permanence with the company
- G1, the highest risk group, is only found in monthly clients who are over 61 years of age
- G2, the risk group, is abundant in the age range between 21 to 64 years among monthly clients

Now yes: it is time to act. Based on this data, my recommendation as a data analyst are the following proposals:

- We must try to displace the G1 of those over 61 years of age, so a specific annual contract would be created for this type of client. With this we could adapt the contract to the specific type of client
- Many of them with a low number of references, we could focus on creating annual plans for nursing homes. In this way we would expand the spectrum and be able to add higher contracts and avoid the loss of clients.
- Since the average length of stay is less than 20 months (1 year and 8 months) in monthly contracts and the largest number of clients are between 21 and 60 years old, it would create a type of promotion to improve the plan. For example, if you complete a year with the monthly contract, you can upgrade it to the annual or bi-annual contract with some type of advantage

## Thanks

If you've made it this far, thank you very much! I continue on my path to be able to dedicate myself to data analysis. In this case, so complex for me, I have used two tools: SQL (BigQuery) and Power BI (DataViz), with which I am becoming familiar.

Any advice or recommendation is welcome. A hug!
