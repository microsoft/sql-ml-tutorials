---
layout: page-steps
language: R
title: Perform customer clustering 
permalink: /R/customerclustering/step/2
---


>After getting SQL Server with Machine Learning Services installed and your R IDE configured on your machine, you can now proceed and perform clustering using R.

>R is a programming language that makes statistical and math computation easy, and is very useful for any machine learning/predictive analytics/statistics work. There are lots of tutorials out there on R. This is one example of a [tutorial](https://www.tutorialspoint.com/r/) to learn more about R.
                
>In this specific scenario, we have an online store and we want to group customers based on their order and return behaviour. 
This information will help us target marketing efforts towards certain groups of customers. Before we go into how you can use R to perform this type of customer grouping using clustering in SQL Server 2016, we will look at the scenario in R.


## Step 2.1 Load the sample data 

**Restore the sample DB**
The dataset used in this tutorial is hosted in several SQL Server tables.The tables contain purchasing and return data based on orders.

1. Download the backup (.bak) file [here](https://sqlchoice.blob.core.windows.net/sqlchoice/static/tpcxbb_1gb.bak), and save it on a location that SQL Server can access.
For example in the folder where SQL Server is installed.
Sample path: C:\Program Files\Microsoft SQL Server\MSSQL13.MSSQLSERVER\MSSQL\Backup


2. Once you have the file saved, open SSMS and a new query window to run the following commands to restore the DB.
Make sure to modify the file paths and server name in the script.

```sql
USE master;  
GO  
RESTORE DATABASE tpcxbb_1gb  
   FROM DISK = 'C:\Program Files\Microsoft SQL Server\MSSQL13.MSSQLSERVER\MSSQL\Backup\tpcxbb_1gb.bak'
   WITH 
                MOVE 'tpcxbb_1gb' TO 'C:\Program Files\Microsoft SQL Server\MSSQL13.MSSQLSERVER\MSSQL\DATA\tpcxbb_1gb.mdf'
                ,MOVE 'tpcxbb_1gb_log' TO 'C:\Program Files\Microsoft SQL Server\MSSQL13.MSSQLSERVER\MSSQL\DATA\tpcxbb_1gb.ldf';  
GO  
```

You should now be able to see the database and tables store_sales and store_returns in the Object Explorer in SSMS.
You can also verify this by querying the tables in SSMS. Open a new query window and select the following.

```sql
USE tpcxbb_1gb;
SELECT TOP (100) * FROM [dbo].[store_sales];
SELECT TOP (100) * FROM [dbo].[store_returns];
```

>You now have the database and the data to perform clustering.


## Step 2.2 Access the data from SQL Server using R

Loading data from SQL Server to R is easy. So let's try it out.

Open a new RScript in your R development tool and run the following script.
Just don't forget to replace "MYSQLSERVER" with the name of your database instance.

In the query we are using to select data from SQL Server, we are separating customers along the following dimensions:
- *return frequency*
- *return order ratio (total number of orders partially or fully returned versus the total number of orders)*
- *return item ratio (total number of items returned versus the number of items purchased)*
- *return amount ration (total monetary amount of items returned versus the amount purchased)*                 

```r
#Connection string to connect to SQL Server. Don't forget to replace MyServer with the name of your SQL Server instance
connStr <- paste("Driver=SQL Server;Server=", " MyServer", ";Database=" , "tpcxbb_1gb" , ";Trusted_Connection=true;" , sep="" );

#The query that we are using to select data from SQL Server
input_query <- "
	SELECT
  ss_customer_sk AS customer,
  round(CASE WHEN ((orders_count = 0) OR (returns_count IS NULL) OR (orders_count IS NULL) OR ((returns_count / orders_count) IS NULL) ) THEN 0.0 ELSE (cast(returns_count as nchar(10)) / orders_count) END, 7) AS orderRatio,
  round(CASE WHEN ((orders_items = 0) OR(returns_items IS NULL) OR (orders_items IS NULL) OR ((returns_items / orders_items) IS NULL) ) THEN 0.0 ELSE (cast(returns_items as nchar(10)) / orders_items) END, 7) AS itemsRatio,
  round(CASE WHEN ((orders_money = 0) OR (returns_money IS NULL) OR (orders_money IS NULL) OR ((returns_money / orders_money) IS NULL) ) THEN 0.0 ELSE (cast(returns_money as nchar(10)) / orders_money) END, 7) AS monetaryRatio,
  round(CASE WHEN ( returns_count IS NULL                                                                        ) THEN 0.0 ELSE  returns_count                 END, 0) AS frequency
  --cast(round(cast(CASE WHEN (returns_count IS NULL) THEN 0.0 ELSE  returns_count END as double)) as integer) AS frequency
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
"
```

Results from the query are returned to R using the rxSqlServerData function. 
This is also where we define the type for columns we want to select (using colClasses), to make sure that the types are correctly transferred to R.

```r
#Querying SQL Server using the query above and and getting the results back to data frame customer_returns
#This is also where we define the type for columns we want to select (using colClasses), to make sure that the types are correctly transferred to R
customer_returns <- RxSqlServerData(sqlQuery=input_query,colClasses=c(customer ="numeric" , orderRatio="numeric" , itemsRatio="numeric" , monetaryRatio="numeric" , frequency="numeric" ),connectionString=connStr);
                
# Transform the data from an input dataset to an output dataset
customer_data <- rxDataStep(customer_returns);
                
#Look at the data we just loaded from SQL Server
head(customer_data, n = 5);
```
```results
customer orderRatio itemsRatio monetaryRatio frequency
1    75675          0          0      0.071633         4
2    86750          0          0      0.000000         0
3    75343          0          0      0.000000         0
4    49280          0          0      0.000000         0
5    89406          0          0      0.000000         0
6    72426          0          0      0.000000         0
```

>You have now read the data from SQL Server to R and explored it.

## Step 2.3 Determine number of clusters

Using the clustering algorithm **Kmeans**, is one of the simplest and most well known ways of grouping data.
Now that we have our selected data, we can group the data into clusters using the iterative data mining algorithm called Kmeans.

The algorithm accepts two inputs: The data itself, and a predefined number "k", the number of clusters.
The output is k clusters with input data partitioned among them.
                    
The goal of K-means is to group the items into k clusters such that all items in same cluster are as similar to each other as possible. And items not in same cluster are as different as possible.
It uses the distance measures to calculate similarity and dissimilarity.

This is how the algorithm works:

1. It randomly chooses k points and make them the initial centroids (each cluster has a centroid which basically is the "center" of the cluster)
2. For each point, it finds the nearest centroid and assigns the point to the cluster associated with the nearest centroid
3. Updates the centroid of each cluster based on members in that cluster. Typically, a new centroid will be the average of all members in the cluster
4. Repeats steps 2 and 3, until the clusters are stable

The number of clusters has to be predefined and the quality of the clusters is heavily dependent on the correctness of the k value specified.
You could just randomly pick a number of clusters, run Kmeans and iterate your way to a good number. 
Or we can use R to evaluate which number of clusters is best for our dataset. Let's determine the number of clusters using R!             
```r
#Determine number of clusters
#Using a plot of the within groups sum of squares, by number of clusters extracted, can help determine the appropriate number of clusters
#We are looking for a bend in the plot. It is at this "elbow" in the plot that we have the appropriate number of clusters 
wss <- (nrow(customer_data) - 1) * sum(apply(customer_data, 2, var))
       for (i in 2:20)
       wss[i] <- sum(kmeans(customer_data, centers = i)$withinss)
plot(1:20, wss, type = "b", xlab = "Number of Clusters", ylab = "Within groups sum of squares")
```

![Elbow graph](https://sqlchoice.blob.core.windows.net/sqlchoice/static/images/Elbow_graph.JPG)
Finding the right number of clusters using an elbow graph
                
Based on the graph above, it looks like *k = 4* would be a good value to try. That will group our customers into 4 clusters.

>Now we have derived the number of clusters to use when clustering.


## Step 2.4 Perform Clustering
Now it is time to use Kmeans. In this sample, we will be using the function rxKmeans which is the Kmeans function in the RevoScaleR package.

```r
# Output table to hold the customer group mappings. This is a table where the cluster mappings will be saved in the database. 
# This table is generated from R
return_cluster = RxSqlServerData(table = "return_cluster", connectionString = connStr);
# Set.seed for random number generator for predictability
set.seed(10);
# Generate clusters using rxKmeans and output key / cluster to a table in SQL Server called return_cluster
clust <- rxKmeans( ~ orderRatio + itemsRatio + monetaryRatio + frequency, customer_returns, numClusters=4
         , outFile=return_cluster, outColName="cluster" , extraVarsToWrite=c("customer"), overwrite=TRUE);

# Read the customer returns cluster table from the DB
customer_cluster <- rxDataStep(return_cluster);

```

>Great, now you have performed clustering in R!

## Step 2.5 Analyze results - Plot clusters

R is very useful since it's easy to plot and visualize data for analysis. Now that we have done the clustering using Kmeans, we need to analyze it and see if we can learn anything from it. Sometimes plotting the clusters might be helpful, so let's do that.

```r
#Plot the clusters (you need to install R library "cluster". If you don't have that installed, just uncomment the next line of code)
#install.packages("cluster")
library("cluster");
clusplot(customer_data, customer_cluster$cluster, color=TRUE, shade=TRUE, labels=4, lines=0, plotchar = TRUE);
```
![Plot of the clusters](https://sqlchoice.blob.core.windows.net/sqlchoice/static/images/Cluster_plot.JPG)

Unfortunaltely, this plot does not really tell us anything about how the different clusters are different from each other. 
We had 4 different variables and we need to know something about the different characteristics of each cluster to make conclusions.
                
Using Kmeans is just a data mining method and you still need to spend time on analyzing the results. 
Depending on your data, you might need to use different methods to analyze the result. Here we just want to show that plotting can be one way.
Let's try something else and see if we can get some more insight from that.

>Now you have explored plotting as a method to evaluate the clusters.


## Step 2.6 Analyze cluster means

The clust object contains the results from our Kmeans clustering. We are going to look at some mean values.

```r
#Look at the clustering details and analyze results
clust
```

```results
Call:
rxKmeans(formula = ~orderRatio + itemsRatio + monetaryRatio + 
    frequency, data = customer_returns, outFile = return_cluster, 
    outColName = "cluster", extraVarsToWrite = c("customer"), 
    overwrite = TRUE, numClusters = 4)
Data: customer_returns
Number of valid observations: 37336
Number of missing observations: 0 
Clustering algorithm:  
 
K-means clustering with 4 clusters of sizes 31675, 671, 2851, 2139
Cluster means:
   orderRatio   itemsRatio monetaryRatio frequency
1 0.000000000 0.0000000000    0.00000000  0.000000
2 0.007451565 0.0000000000    0.04449653  4.271237
3 1.008067345 0.2707821817    0.49515232  1.031568
4 0.000000000 0.0004675082    0.10858272  1.186068
Within cluster sum of squares by cluster:
         1          2          3          4 
    0.0000  1329.0160 18561.3157   363.2188 
    
```

Focusing on the cluster mean values, it seems like we can actually interpret something. 
Just to refresh our memory, here are the definitions of our variables:

* *frequency = return frequency*
* *orderRatio = return order ratio (total number of orders partially or fully returned versus the total number of orders)*
* *itemsRatio = return item ratio (total number of items returned versus the number of items purchased)*
* *monetaryRatio = return amount ratio (total monetary amount of items returned versus the amount purchased)*

Some examples of what the mean values tell us?

* Well, cluster 1 (the largest cluster) seems to be a group of customers that are not active. All values are zero.
*And cluster 3 seems to be a group that stands out in terms of return behaviour.
                                  
Data mining using Kmeans often requires further analysis of the results, and further steps to better understand each cluster, 
but it provides some very good leads. Cluster 1 is a set of customers who are clearly not active. Perhaps we can target marketing efforts towards this group to trigger an interest for purchases? 
Let's query the database for their email addresses so that we can send a marketing email to them.
             
Use the following select statement to use the cluster data to select email addresses to customers in a specific cluster from te tables in SQL Server. 

Open a new query in SSMS and run the following select statement:

```sql
USE [tpcxbb_1gb]
SELECT customer.[c_email_address], customer.c_customer_sk
  FROM dbo.customer
  JOIN 
  [dbo].[return_cluster] as r
  ON r.customer = customer.c_customer_sk
  WHERE r.cluster = 3
```

> Congrats you just performed clustering with R! Let us now deploy our R code by moving it to SQL Server.
