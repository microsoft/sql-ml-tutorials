---
layout: page-steps
language: R
title: Build a predictive model
permalink: /R/rentalprediction/step/2
---

>After getting SQL Server with ML Services installed and your R IDE configured on your machine, you can now proceed to **train** a predictive model with R.
>R is a programming language that makes statistical and math computation easy, and is very useful for any machine learning/predictive analytics/statistics work.
There are lots of tutorials out there on R. This is one example of a [tutorial](https://www.tutorialspoint.com/r/) to learn more about R.
                
>In this specific scenario, we own a ski rental business, and we want to predict the number of rentals that
we will have on a future date. This information will help us to get ready from a stock, staff and facilities perspective.


>During **model training**, you create and train a predictive model by showing it sample data along with the outcomes. Then you save this model so that you can use it later when you want to make predictions against new data.

## Step 2.1 Restore the sample DB 


The dataset used in this tutorial is hosted in a SQL Server table.The table contains rental data from previous years.
1.Download the backup (.bak) file [here](https://sqlchoice.blob.core.windows.net/sqlchoice/TutorialDB.bak), and save it on a location that SQL Server can access.
For example in the folder where SQL Server is installed.
Sample path: C:\Program Files\Microsoft SQL Server\MSSQL13.MSSQLSERVER\MSSQL\Backup


2.Once you have the file saved, open SSMS and a new query window to run the following commands to restore the DB.
Make sure to modify the file paths and server name in the script.

```sql
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


```sql
USE tutorialdb;
SELECT * FROM [dbo].[rental_data];
```

>You now have the database and the data to use for training the model.

## Step 2.2 Access the data from SQL Server using R

Loading data from SQL Server to R is easy. So let's try it out.
Open a new RScript in your R development tool and run the following script.
Just don't forget to replace "MYSQLSERVER" with the name of your database instance.

```r
#Connection string to connect to SQL Server named instance
connStr <- paste("Driver=SQL Server; Server=", "MYSQLSERVER", 
                ";Database=", "Tutorialdb", ";Trusted_Connection=true;", sep = "");

#Get the data from SQL Server Table
SQL_rentaldata <- RxSqlServerData(table = "dbo.rental_data",
                              connectionString = connStr, returnDataFrame = TRUE);

#Import the data into a data frame
rentaldata <- rxImport(SQL_rentaldata);

#Let's see the structure of the data and the top rows
# Ski rental data, giving the number of ski rentals on a given date
head(rentaldata);
str(rentaldata);
```

```results
Year Month Day RentalCount WeekDay Holiday Snow
1 2014     1  20         445       2       1    0
2 2014     2  13          40       5       0    0
3 2013     3  10         456       1       0    0
4 2014     3  31          38       2       0    0
5 2014     4  24          23       5       0    0
6 2015     2  11          42       4       0    0
'data.frame':       453 obs. of  7 variables:
$ Year       : int  2014 2014 2013 2014 2014 2015 2013 2014 2013 2015 ...
$ Month      : num  1 2 3 3 4 2 4 3 4 3 ...
$ Day        : num  20 13 10 31 24 11 28 8 5 29 ...
$ RentalCount: num  445 40 456 38 23 42 310 240 22 360 ...
$ WeekDay    : num  2 5 1 2 5 4 1 7 6 1 ...
$ Holiday    : int  1 0 0 0 0 0 0 0 0 0 ...
$ Snow       : num  0 0 0 0 0 0 0 0 0 0 ...
```
>You have now read the data from SQL Server to R and explored it.

## Step 2.3 Prepare the data

Often times, the data needs to be prepared and cleaned before we start calling the functions. In this tutorial, most of the preparations have already been done.
We are just going to change the type of 3 columns to factors.

```r
#Changing the three factor columns to factor types
#This helps when building the model because we are explicitly saying that these values are categorical
rentaldata$Holiday <- factor(rentaldata$Holiday);
rentaldata$Snow <- factor(rentaldata$Snow);
rentaldata$WeekDay <- factor(rentaldata$WeekDay);

#Visualize the dataset after the change
str(rentaldata);
```

```results
data.frame':      453 obs. of  7 variables:
$ Year       : int  2014 2014 2013 2014 2014 2015 2013 2014 2013 2015 ...
$ Month      : num  1 2 3 3 4 2 4 3 4 3 ...
$ Day        : num  20 13 10 31 24 11 28 8 5 29 ...
$ RentalCount: num  445 40 456 38 23 42 310 240 22 360 ...
$ WeekDay    : Factor w/ 7 levels "1","2","3","4",..: 2 5 1 2 5 4 1 7 6 1 ...
$ Holiday    : Factor w/ 2 levels "0","1": 2 1 1 1 1 1 1 1 1 1 ...
$ Snow       : Factor w/ 2 levels "0","1": 1 1 1 1 1 1 1 1 1 1 ...
```
>The data is now prepared for training.

## Step 2.4 Train a model
In order to predict, we have to first find a function (model) that best describes the dependency between the variables in our dataset. This step is called training the model. The training dataset will be a subset of the entire dataset.
We are going to create two different models and see which one is predicting more accurately.

```r
#Now let's split the dataset into 2 different sets
#One set for training the model and the other for validating it
train_data = rentaldata[rentaldata$Year < 2015,];
test_data = rentaldata[rentaldata$Year == 2015,];

#Use this column to check the quality of the prediction against actual values
actual_counts <- test_data$RentalCount;

#Model 1: Use rxLinMod to create a linear regression model. We are training the data using the training data set
model_linmod <- rxLinMod(RentalCount ~  Month + Day + WeekDay + Snow + Holiday, data = train_data);

#Model 2: Use rxDTree to create a decision tree model. We are training the data using the training data set
model_dtree <- rxDTree(RentalCount ~ Month + Day + WeekDay + Snow + Holiday, data = train_data);
```
>Now we have trained and created two different models! Let's use them to predict and see which model is the most accurate.



## Step 2.5 prediction
We are now going to use a predict function to predict the Rental Counts using our two models. Finding the right type of model for a specific problem requires some experimentation. 
This [cheat sheet](https://azure.microsoft.com/en-us/documentation/articles/machine-learning-algorithm-choice/#the-machine-learning-algorithm-cheat-sheet) can be nice to have as a guide.

```r
#Use the models we just created to predict using the test data set.
#That enables us to compare actual values of RentalCount from the two models and compare to the actual values in the test data set
predict_linmod <- rxPredict(model_linmod, test_data, writeModelVars = TRUE, extraVarsToWrite = c("Year"));

predict_dtree <- rxPredict(model_dtree, test_data, writeModelVars = TRUE, extraVarsToWrite = c("Year"));

#Look at the top rows of the two prediction data sets.
head(predict_linmod);
head(predict_dtree);
```

```results
RentalCount_Pred RentalCount Month Day WeekDay Snow Holiday
1         27.45858          42     2  11       4    0       0
2        387.29344         360     3  29       1    0       0
3         16.37349          20     4  22       4    0       0
4         31.07058          42     3   6       6    0       0
5        463.97263         405     2  28       7    1       0
6        102.21695          38     1  12       2    1       0
  RentalCount_Pred RentalCount Month Day WeekDay Snow Holiday
1          40.0000          42     2  11       4    0       0
2         332.5714         360     3  29       1    0       0
3          27.7500          20     4  22       4    0       0
4          34.2500          42     3   6       6    0       0
5         645.7059         405     2  28       7    1       0
6          40.0000          38     1  12       2    1       0
```

>Congrats you just created two models with R! 

## Step 2.6 Compare results

Now let's see which of the models gives the best predictions. To be able to find out, we are going to plot the difference between the predicted and actual values.
R is a great language for quickly and easily visualizing data. We are going to use a basic plotting function to plot 2 graphs.


```r
#Now we will use the plotting functionality in R to viusalize the results from the predictions
#We are plotting the difference between actual and predicted values for both models to compare accuracy
par(mfrow = c(2, 1));
plot(predict_linmod$RentalCount_Pred - predict_linmod$RentalCount, main = "Difference between actual and predicted. rxLinmod");
plot(predict_dtree$RentalCount_Pred - predict_dtree$RentalCount, main = "Difference between actual and predicted. rxDTree");
```

![alt text](https://sqlchoice.blob.core.windows.net/sqlchoice/RLANG_CompareModels.JPG "Comparing the two models")

It looks like the decision tree model is more accurate. We have a quite accurate predictor and we feel confident to use it to predict
what is going to happen on a given situation in the future. 

> Congrats, you have decided on which model to use! Let us now deploy our R code by moving it to SQL Server.
