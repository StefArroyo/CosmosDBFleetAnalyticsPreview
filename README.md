# Cosmos DB Fleet Analytics Preview

Below find sample queries to help you navigate your Cosmos DB Fleet Analytics data.

### **Transactions Per Dollar**

This query provides the analysis for Transactions Per Dollar (total database transactions per dollar spent (TPD)). Total database transactions per dollar is defined as transactions (count of document and sproc operations on the resource) divided by the cost charged to in dollars post EA and RI discounts.

```sql
DECLARE @StartDate DATE = '2024-05-01';
DECLARE @EndDate DATE = '2024-05-30';
WITH _ProfileAccounts AS (
    SELECT AccountName, SubscriptionId
    FROM Customer_FactAccountHourly AS a
    INNER JOIN Customer_DimResource AS r 
    on a.[ResourceId] = r.[ResourceId]
    AND (
        AccountName LIKE 'Account1%'
        OR AccountName LIKE 'Account2%'
        OR AccountName LIKE 'Account3%'
        OR AccountName LIKE 'Account4%'
        OR AccountName LIKE 'Account5%'
    )
    GROUP BY AccountName, SubscriptionId
),
requestCountInfo AS (
    SELECT DATEADD(MONTH, DATEDIFF(MONTH, 0, Timestamp), 0) AS Timestamp,
           SUM(TotalRequestCount) AS Transactions
    FROM Customer_FactRequestHourly AS f
    INNER JOIN Customer_DimResource AS r 
    on f.[ResourceId] = r.[ResourceId]
   -- WHERE Timestamp > 
    WHERE Timestamp >= @StartDate AND Timestamp < @EndDate
    AND (AccountName IN (SELECT AccountName FROM _ProfileAccounts))
    GROUP BY DATEADD(MONTH, DATEDIFF(MONTH, 0, Timestamp), 0)
),
billingInfo AS (
    SELECT DATEADD(MONTH, DATEDIFF(MONTH, 0, Timestamp), 0) AS Timestamp,
           SUM(CASE WHEN MeterName = 'Usage' THEN 0.1 * 0.008 * ConsumedUnits ELSE 0.48 * ConsumedUnits END) AS actualCost
    FROM Customer_FactMeterUsageHourly a
        INNER JOIN Customer_DimResource r 
        on a.[ResourceId] = r.[ResourceId]
        INNER JOIN Customer_DimMeter m
        on a.[MeterId] = m.[MeterId]  
    WHERE Timestamp >= @StartDate AND Timestamp < @EndDate
    AND (AccountName IN (SELECT AccountName FROM _ProfileAccounts))
    GROUP BY DATEADD(MONTH, DATEDIFF(MONTH, 0, Timestamp), 0)
)
SELECT RC.Timestamp,
       RC.Transactions,
       BI.actualCost,
       RC.Transactions / NULLIF(BI.actualCost, 0) AS transactionsPerDollar
FROM requestCountInfo RC
JOIN billingInfo BI ON RC.Timestamp = BI.Timestamp ;
```

## Reserved Capacity Purchase Recommendation
This query provides a recommendation for how much RI to purchase for your Cosmos DB estate. Note: This is a conservative recommendation below what your minimum provisioned throughput is.

```sql
SELECT *
FROM [Customer_FactRequestHourly] a
INNER JOIN [Customer_DimResource] r 
on a.[ResourceId] = r.[ResourceId]
/*WHERE a.[SubscriptionId] == ""*/
/*WHERE (AccountName LIKE ''
   OR AccountName LIKE 'Account1%'
   OR AccountName LIKE 'Account2%'
   OR AccountName LIKE 'Account3%'
   OR AccountName LIKE 'Account4%')*/
   --AND Timestamp >= 'start_date'
   -


## RequestsHourly
Provides a view of your request history by hour.
```sql
SELECT *
FROM [Customer_FactRequestHourly] a
INNER JOIN [Customer_DimResource] r 
on a.[ResourceId] = r.[ResourceId]
/*WHERE a.[SubscriptionId] == ""*/
/*WHERE (AccountName LIKE ''
   OR AccountName LIKE 'Account1%'
   OR AccountName LIKE 'Account2%'
   OR AccountName LIKE 'Account3%'
   OR AccountName LIKE 'Account4%')*/
   --AND Timestamp >= 'start_date'
   --AND Timestamp < 'end_date'
```


## Costs Hourly
Provides a view of your cost history by hour.
```sql
SELECT * 
FROM Customer_FactMeterUsageHourly a
INNER JOIN Customer_DimResource r 
on a.[ResourceId] = r.[ResourceId]
INNER JOIN Customer_DimMeter m
on a.[MeterId] = m.[MeterId]
/*WHERE a.[SubscriptionId] == ""*/
WHERE (AccountName LIKE '%-profile-cosmosdb-0'
   OR AccountName LIKE 'Account1%'
   OR AccountName LIKE 'Account2%'
   OR AccountName LIKE 'Account3%'
   OR AccountName LIKE 'Account4%')
   --AND Timestamp >= 'start_date'
   --AND Timestamp < 'end_date'
```



