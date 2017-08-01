---
layout: page-steps
language: Python
title: Build a predictive model 
permalink: /python/rentalprediction/step/2
---


>After getting SQL Server with ML Services installed and your Python IDE configured on your machine, you can now proceed to **train** a predictive model with Python.

>In this specific scenario, we own a ski rental business, and we want to predict the number of rentals that
we will have on a future date. This information will help us to get ready from a stock, staff and facilities perspective.


>During **model training**, you create and train a predictive model by showing it sample data along with the outcomes. Then you save this model so that you can use it later when you want to make predictions against new data.

## Step 2.1 Load the sample data 

**Restore the sample DB**
The dataset used in this tutorial is hosted in a SQL Server table.The table contains rental data from previous years.</p>
1. Download the backup (.bak) file [here](https://sqlchoice.blob.core.windows.net/sqlchoice/TutorialDB.bak), and save it on a location that SQL Server can access.
For example in the folder where SQL Server is installed.
Sample path: C:\Program Files\Microsoft SQL Server\MSSQL13.MSSQLSERVER\MSSQL\Backup


2.Once you have the file saved, open SSMS and a new query window to run the following commands to restore the DB.
Make sure to modify the file paths and server name in the script.

```SQL
USE master;  
GO  
RESTORE DATABASE TutorialDB  
   FROM DISK = 'C:\Program Files\Microsoft SQL Server\MSSQL13.MSSQLSERVER\MSSQL\Backup\TutorialDB.bak'
   WITH 
   MOVE 'TutorialDB' TO 'C:\Program Files\Microsoft SQL Server\MSSQL13.MSSQLSERVER\MSSQL\DATA\TutorialDB.mdf'
   ,MOVE 'TutorialDB_log' TO 'C:\Program Files\Microsoft SQL Server\MSSQL13.MSSQLSERVER\MSSQL\DATA\TutorialDB.ldf';  
GO 
```

A table named rental_data containing the dataset should exist in the restored SQL Server database.
You can verify this by querying the table in SSMS.


```SQL
USE tutorialdb;
SELECT * FROM [dbo].[rental_data];
```

>You now have the database and the data to use for training the model.

## Step 2.2 Explore the data with Python

Loading data from SQL Server to Python is easy. So let's try it out.

Open a new Python script in your IDE and run the following script.
Just don't forget to replace "MYSQLSERVER" with the name of your database instance.

```python
import pandas as pd
from sklearn.linear_model import LinearRegression
from sklearn.metrics import mean_squared_error

#If you are running SQL Server 2017 RC1 and above:
from revoscalepy import RxComputeContext, RxInSqlServer, RxSqlServerData
from revoscalepy import rx_import

#Connection string to connect to SQL Server named instance
conn_str = 'Driver=SQL Server;Server=MYSQLSERVER;Database=TutorialDB;Trusted_Connection=True;'

#Define the columns we wish to import
 column_info = { 
         "Year" : { "type" : "integer" },
         "Month" : { "type" : "integer" }, 
         "Day" : { "type" : "integer" }, 
         "RentalCount" : { "type" : "integer" }, 
         "WeekDay" : { 
             "type" : "factor", 
             "levels" : ["1", "2", "3", "4", "5", "6", "7"]
         },
         "Holiday" : { 
             "type" : "factor", 
             "levels" : ["1", "0"]
         },
         "Snow" : { 
             "type" : "factor", 
             "levels" : ["1", "0"]
         }
     }

#Get the data from SQL Server Table
data_source = RxSqlServerData(table="dbo.rental_data",
                               connection_string=conn_str, column_info=column_info)
computeContext = RxInSqlServer(
     connection_string = conn_str,
     num_tasks = 1,
     auto_cleanup = False
)
     
    
RxInSqlServer(connection_string=conn_str, num_tasks=1, auto_cleanup=False)

 # import data source and convert to pandas dataframe
df = pd.DataFrame(rx_import(input_data = data_source))
print("Data frame:", df)
# Get all the columns from the dataframe.
columns = df.columns.tolist()
# Filter the columns to remove ones we don't want to use in the training
columns = [c for c in columns if c not in ["Year"]]
```

```results
Rows Processed: 453
Data frame:      Day  Holiday  Month  RentalCount  Snow  WeekDay  Year
0     20        1      1          445     2        2  2014
1     13        2      2           40     2        5  2014
2     10        2      3          456     2        1  2013
3     31        2      3           38     2        2  2014
4     24        2      4           23     2        5  2014
5     11        2      2           42     2        4  2015
6     28        2      4          310     2        1  2013
...
[453 rows x 7 columns]
```

>You have now read the data from SQL Server to Python and explored it.

## Step 2.3 Train a model
In order to predict, we first have to find a function (model) that best describes the dependency between the variables in our dataset. This step is called **training the model**. The training dataset will be a subset of the entire dataset.

We are going to create a model using a linear regression algorithm.

```python
 # Store the variable we'll be predicting on.
target = "RentalCount"
# Generate the training set.  Set random_state to be able to replicate results.
train = df.sample(frac=0.8, random_state=1)
# Select anything not in the training set and put it in the testing set.
test = df.loc[~df.index.isin(train.index)]
# Print the shapes of both sets.
print("Training set shape:", train.shape)
print("Testing set shape:", test.shape)
# Initialize the model class.
lin_model = LinearRegression()
# Fit the model to the training data.
lin_model.fit(train[columns], train[target])
```

```results
Training set shape: (362, 7)
Testing set shape: (91, 7)
```

>Now we have trained a linear regression model in Python! Let's use it to predict the rental count.

## Step 2.4 Prediction
We are now going to use a predict function to predict the Rental Counts using our two models. 

```python
# Generate our predictions for the test set.
lin_predictions = lin_model.predict(test[columns])
print("Predictions:", lin_predictions)
# Compute error between our test predictions and the actual values.
lin_mse = mean_squared_error(lin_predictions, test[target])
print("Computed error:", lin_mse)
```

```results
Predictions: [  40.   38.  240.   39.  514.   48.  297.   25.  507.   24.   30.   54.
   40.   26.   30.   34.   42.  390.  336.   37.   22.   35.   55.  350.
  252.  370.  499.   48.   37.  494.   46.   25.  312.  390.   35.   35.
  421.   39.  176.   21.   33.  452.   34.   28.   37.  260.   49.  577.
  312.   24.   24.  390.   34.   64.   26.   32.   33.  358.  348.   25.
   35.   48.   39.   44.   58.   24.  350.  651.   38.  468.   26.   42.
  310.  709.  155.   26.  648.  617.   26.  846.  729.   44.  432.   25.
   39.   28.  325.   46.   36.   50.   63.]
Computed error: 3.59831533436e-26
```

> Congrats you just created a model with Python! Let us now deploy our Python code by moving it to SQL Server.
