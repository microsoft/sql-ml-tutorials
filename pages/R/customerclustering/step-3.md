---
layout: page-steps
language: R
title: Perform customer clustering
permalink: /R/customerclustering/step/3
---


>In this section, we will move the R code we just wrote into SQL Server and deploy our clustering with the help of SQL Server Machine Learning Services.

>In order to perform clustering on a regular basis, as new customers are registering, we need to be able call our R script from any App. 
To do that, we can simply delploy the R Script in SQL Server. You basically put the R script inside a SQL stored procedure in the database. A stored procedure is like a function, that takes parameters and can return results, and the good thing with these procedures is that they can be called from any application. 


## Step 3.1 Create a stored procedure for clustering

In SSMS, launch a new query window and run the following T-SQL script to create the stored procedure:

```sql
USE [tpcxbb_1gb]
DROP PROC IF EXISTS generate_customer_return_clusters;
GO
CREATE procedure [dbo].[generate_customer_return_clusters]
AS
/* 
  This procedure uses R to classify customers into different groups based on their
  purchase & return history.
*/
BEGIN
	DECLARE @duration FLOAT
	, @instance_name NVARCHAR(100) = @@SERVERNAME
	, @database_name NVARCHAR(128) = db_name()
-- Input query to generate the purchase history & return metrics
	, @input_query NVARCHAR(MAX) = N'
SELECT
  ss_customer_sk AS customer,
  round(CASE WHEN ((orders_count = 0) OR (returns_count IS NULL) OR (orders_count IS NULL) OR ((returns_count / orders_count) IS NULL) ) THEN 0.0 ELSE (cast(returns_count as nchar(10)) / orders_count) END, 7) AS orderRatio,
  round(CASE WHEN ((orders_items = 0) OR(returns_items IS NULL) OR (orders_items IS NULL) OR ((returns_items / orders_items) IS NULL) ) THEN 0.0 ELSE (cast(returns_items as nchar(10)) / orders_items) END, 7) AS itemsRatio,
  round(CASE WHEN ((orders_money = 0) OR (returns_money IS NULL) OR (orders_money IS NULL) OR ((returns_money / orders_money) IS NULL) ) THEN 0.0 ELSE (cast(returns_money as nchar(10)) / orders_money) END, 7) AS monetaryRatio,
  round(CASE WHEN ( returns_count IS NULL                                                                        ) THEN 0.0 ELSE  returns_count                 END, 0) AS frequency
  
FROM
  (
    SELECT
      ss_customer_sk,
      -- return order ratio
      COUNT(distinct(ss_ticket_number)) AS orders_count,
      -- return ss_item_sk ratio
      COUNT(ss_item_sk) AS orders_items,
      -- return monetary amount ratio
      SUM( ss_net_paid ) AS orders_money
    FROM store_sales s
    GROUP BY ss_customer_sk
  ) orders
  LEFT OUTER JOIN
  (
    SELECT
      sr_customer_sk,
      -- return order ratio
      count(distinct(sr_ticket_number)) as returns_count,
      -- return ss_item_sk ratio
      COUNT(sr_item_sk) as returns_items,
      -- return monetary amount ratio
      SUM( sr_return_amt ) AS returns_money
    FROM store_returns
    GROUP BY sr_customer_sk
  ) returned ON ss_customer_sk=sr_customer_sk 
 '
EXEC sp_execute_external_script
	  @language = N'R'
	, @script = N'
# Define the connection string
connStr <- paste("Driver=SQL Server;Server=", instance_name, " ;Database=", database_name, " ;Trusted_Connection=true;", sep="" );

# Input customer data that needs to be classified. This is the result we get from our query
customer_returns <- RxSqlServerData(sqlQuery = input_query,
									colClasses = c(customer = "numeric", orderRatio = "numeric", itemsRatio = "numeric", monetaryRatio = "numeric", frequency = "numeric"),
									connectionString = connStr);

# Output table to hold the customer cluster mappings
return_cluster = RxSqlServerData(table = "customer_return_clusters", connectionString = connStr);

# set.seed for random number generator for predicatability
set.seed(10);

# generate clusters using rxKmeans and output clusters to a table called "customer_return_clusters".
clust <- rxKmeans( ~ orderRatio + itemsRatio + monetaryRatio + frequency, customer_returns, numClusters = 4
                    , outFile = return_cluster, outColName = "cluster", writeModelVars = TRUE , extraVarsToWrite = c("customer"), overwrite = TRUE);
'
	, @input_data_1 = N''
	, @params = N'@instance_name nvarchar(100), @database_name nvarchar(128), @input_query nvarchar(max), @duration float OUTPUT'
	, @instance_name = @instance_name
	, @database_name = @database_name
	, @input_query = @input_query
	, @duration = @duration OUTPUT;
END;

GO
```

>You have now created a stored procedure that contains the R script for clustering.


## Step 3.2 Perform clustering in SQL Server

We are now going to execute the stored procedure in SQL Server to perform the clustering 

```sql
--Empty table of the results before running the stored procedure
TRUNCATE TABLE customer_return_clusters;
--Execute the clustering. This will load the table customer_return_clusters with cluster mappings
EXEC [dbo].[generate_customer_return_clusters];
                               
```

Now let's verify that it works and that we actually have the list of customers and their cluster mappings

```sql             
--Now select data from table customer_return_clusters to verify that the clustering data was loaded
SELECT * FROM customer_return_clusters;

```

>Congrats, you have performed clustering with R inside SQL Server!


## Step 3.3 Why is it useful to deploy this in SQL Server?

We now have the clustering implemented in SQL Server. Why is that useful?

Well, imagine that you need to perform clustering on you cutomer data on a regular basis as new customers sign up to keep an updated understanding of customer behavior. In this example, we might want to send out promotion emails and can select the email addresses of customers in cluster 3 to send out a promotion.

You can also schedule jobs that run the stored procedure and automatically send the results to for example a CRM application or a reporting tool. 

The code below is selecting the email addresses of customers in cluster 3, for a promotion campaign:

```sql
USE [tpcxbb_1gb]
SELECT customer.[c_email_address], customer.c_customer_sk
  FROM dbo.customer
  JOIN 
  [dbo].[customer_return_clusters] as r
  ON r.customer = customer.c_customer_sk
  WHERE r.cluster = 3
```

> Congrats, you have now performed clustering in SQL Server with R using SQL Server Machine Learning Services! 
