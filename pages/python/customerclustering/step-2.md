---
layout: page-steps
language: Python
title: Perform customer clustering 
permalink: /python/customerclustering/step/2
---


>After getting SQL Server 2017 with Machine Learning Services installed and your Python IDE up on your machine, you can now proceed and perform clustering using Python.

>Python is a programming language that can be used for machine learning and predictive analytics. 
                
>In this specific scenario, we have a store and we want to group customers based on their order and return behaviour. 
This information will help us target marketing efforts towards certain groups of customers. Before we go into how you can use R to perform this type of customer grouping using clustering in SQL Server 2017, we will look at the scenario in Python.


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


## Step 2.2 Access the data from SQL Server using Python

Loading data from SQL Server to Python is easy. So let's try it out.

Open a new Python Script in your Python IDE and run the following script.
Just don't forget to replace "MYSQLSERVER" with the name of your database instance.

In the query we are using to select data from SQL Server, we are separating customers along the following dimensions:
- *return frequency*
- *return order ratio (total number of orders partially or fully returned versus the total number of orders)*
- *return item ratio (total number of items returned versus the number of items purchased)*
- *return amount ration (total monetary amount of items returned versus the amount purchased)*                 

```Python
# Load packages.
import matplotlib.pyplot as plt
import numpy as np
import pandas as pd
import revoscalepy as revoscale
from scipy.spatial import distance as sci_distance
from sklearn import cluster as sk_cluster



def perform_clustering():
    ################################################################################################

    ##	Connect to DB and select data

    ################################################################################################

    # Connection string to connect to SQL Server named instance.
    conn_str = 'Driver=SQL Server;Server=localhost;Database=tpcxbb_1gb;Trusted_Connection=True;'

    input_query = '''SELECT
    ss_customer_sk AS customer,
    ROUND(COALESCE(returns_count / NULLIF(1.0*orders_count, 0), 0), 7) AS orderRatio,
    ROUND(COALESCE(returns_items / NULLIF(1.0*orders_items, 0), 0), 7) AS itemsRatio,
    ROUND(COALESCE(returns_money / NULLIF(1.0*orders_money, 0), 0), 7) AS monetaryRatio,
    COALESCE(returns_count, 0) AS frequency
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
    GROUP BY sr_customer_sk ) returned ON ss_customer_sk=sr_customer_sk'''


    # Define the columns we wish to import.
    column_info = {
        "customer": {"type": "integer"},
        "orderRatio": {"type": "integer"},
        "itemsRatio": {"type": "integer"},
        "frequency": {"type": "integer"}
    }

    ```

Results from the query are returned to Python using the revoscalepy RxSqlServerData function. 
This is also where we provide column info, to make sure that the types are correctly transferred.

```Python
data_source = revoscale.RxSqlServerData(sql_query=input_query, column_Info=column_info,
                                              connection_string=conn_str)
    revoscale.RxInSqlServer(connection_string=conn_str, num_tasks=1, auto_cleanup=False)
    # import data source and convert to pandas dataframe.
    customer_data = pd.DataFrame(revoscalepy.rx_import(data_source))
    print("Data frame:", customer_data.head(n=5))
```

```results
Rows Read: 37336, Total Rows Processed: 37336, Total Chunk Time: 0.172 seconds 
Data frame:     customer  orderRatio  itemsRatio  monetaryRatio  frequency
0    29727.0    0.000000    0.000000       0.000000          0
1    97643.0    0.068182    0.078176       0.037034          3
2    57247.0    0.000000    0.000000       0.000000          0
3    32549.0    0.086957    0.068657       0.031281          4
4     2040.0    0.000000    0.000000       0.000000          0
```

>You have now read the data from SQL Server to Python.

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
Or we can use Python to evaluate which number of clusters is best for our dataset. Let's determine the number of clusters using Python and the Elbow method!             

```Python
	################################################################################################

    ##	Determine number of clusters using the Elbow method

    ################################################################################################

    cdata = customer_data
    K = range(1, 20)
    KM = (sk_cluster.KMeans(n_clusters=k).fit(cdata) for k in K)
    centroids = (k.cluster_centers_ for k in KM)

    D_k = (sci_distance.cdist(cdata, cent, 'euclidean') for cent in centroids)
    dist = (np.min(D, axis=1) for D in D_k)
    avgWithinSS = [sum(d) / cdata.shape[0] for d in dist]
    plt.plot(K, avgWithinSS, 'b*-')
    plt.grid(True)
    plt.xlabel('Number of clusters')
    plt.ylabel('Average within-cluster sum of squares')
    plt.title('Elbow for KMeans clustering')
    plt.show()
```

![Elbow graph](https://sqlchoice.blob.core.windows.net/sqlchoice/static/images/Python_Elbow_Graph.png)
Finding the right number of clusters using an elbow graph
                
Based on the graph, it looks like *k = 4* would be a good value to try. That will group our customers into 4 clusters.

>Now we have derived the number of clusters to use when clustering!


## Step 2.4 Perform Clustering
It is now time to use Kmeans. In this sample, we will be using the KMeans function from the sklearn package.

```Python
	################################################################################################

    ##	Perform clustering using Kmeans

    ################################################################################################

    # It looks like k=4 is a good number to use based on the elbow graph.
    n_clusters = 4

    means_cluster = sk_cluster.KMeans(n_clusters=n_clusters, random_state=111)
    columns = ["orderRatio", "itemsRatio", "monetaryRatio", "frequency"]
    est = means_cluster.fit(customer_data[columns])
    clusters = est.labels_
    customer_data['cluster'] = clusters

    # Print some data about the clusters:

    # For each cluster, count the members.
    for c in range(n_clusters):
        cluster_members=customer_data[customer_data['cluster'] == c][:]
        print('Cluster{}(n={}):'.format(c, len(cluster_members)))
        print('-'* 17)

    # Print mean values per cluster.
    print(customer_data.groupby(['cluster']).mean())


perform_clustering()
```

>Great, now you have performed clustering in Python!

## Step 2.5 Analyze clusters

Now that we have done the clustering using Kmeans, we need to analyze the clusters and see if we can learn anything from that. 
The clustering mean values and the cluster sizes we just printed could tell us something about our data.

```results

Cluster0(n=31675):
-------------------
Cluster1(n=4989):
-------------------
Cluster2(n=1):
-------------------
Cluster3(n=671):
-------------------

         customer  orderRatio  itemsRatio  monetaryRatio  frequency
cluster                                                                
0        50854.809882    0.000000    0.000000       0.000000   0.000000
1        51332.535779    0.721604    0.453365       0.307721   1.097815
2        57044.000000    1.000000    2.000000     108.719154   1.000000
3        48516.023845    0.136277    0.078346       0.044497   4.271237
```

Focusing on the cluster mean values, it seems like we can actually interpret something. 
Just to refresh our memory, here are the definitions of our variables:

* *frequency = return frequency*
* *orderRatio = return order ratio (total number of orders partially or fully returned versus the total number of orders)*
* *itemsRatio = return item ratio (total number of items returned versus the number of items purchased)*
* *monetaryRatio = return amount ratio (total monetary amount of items returned versus the amount purchased)*

Some examples of what the mean values tell us?

* Well, cluster 0 (the largest cluster) seems to be a group of customers that are very inactive. All values are zero.
* And cluster 3 seems to be a group that stands out in terms of return behaviour.
                                  
Data mining using Kmeans often requires further analysis of the results, and further steps to better understand each cluster, 
but it provides some very good leads. Cluster 0 is a set of customers who are clearly not active. Perhaps we can target marketing efforts towards this group to trigger an interest for purchases? 
In the next step, we will query the database for the email addresses of customers in cluster 0, so that we can send a marketing email to them.
             
> Congrats you just performed clustering in Python! Let us now deploy our Python code by moving it to SQL Server.
