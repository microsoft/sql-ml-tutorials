---
layout: page-steps
language: R
title: Build a predictive model
permalink: /R/rentalprediction/step/3
---



>In this section, we will move the R code we just wrote to SQL Server and deploy our predictive model with the help of SQL Server Machine Learning Services.
>To deploy a model, you store the model in a hosting environment (like a database) and implement a prediction function that uses the model to predict.  That function can be called from applications.

SQL Server ML Services enables you to train and test predictive models in the context of SQL Server. You author T-SQL programs that contain embedded R scripts, and the SQL Server database engine takes care of the execution. Because it executes in SQL Server, your models can easily be trained against data stored in the database.
To deploy, you store your model in the database and create a stored procedure that predicts using the model.


## Step 3.1 Create a table for storing the model

In SSMS, launch a new query window and run the following T-SQL script:

```sql
USE TutorialDB;
DROP TABLE IF EXISTS rental_rx_models;
GO
CREATE TABLE rental_rx_models (
                model_name VARCHAR(30) NOT NULL DEFAULT('default model') PRIMARY KEY,
                model VARBINARY(MAX) NOT NULL
);
GO
```

## Step 3.2 Create stored procedure for generating the model

We are now going to create a stored procedure in SQL Server to use the R code we wrote in the previous module and generate the rxDTree model inside the database.
The R code will be embedded in the TSQL statement.

1.Create the stored procedure

```sql
-- Stored procedure that trains and generates an R model using the rental_data and a decision tree algorithm
DROP PROCEDURE IF EXISTS generate_rental_rx_model;
go
CREATE PROCEDURE generate_rental_rx_model (@trained_model varbinary(max) OUTPUT)
AS
BEGIN
    EXECUTE sp_execute_external_script
      @language = N'R'
    , @script = N'
        require("RevoScaleR");

			rental_train_data$Holiday = factor(rental_train_data$Holiday);
            rental_train_data$Snow = factor(rental_train_data$Snow);
            rental_train_data$WeekDay = factor(rental_train_data$WeekDay);

        #Create a dtree model and train it using the training data set
        model_dtree <- rxDTree(RentalCount ~ Month + Day + WeekDay + Snow + Holiday, data = rental_train_data);
        #Before saving the model to the DB table, we need to serialize it
        trained_model <- as.raw(serialize(model_dtree, connection=NULL));'

    , @input_data_1 = N'select "RentalCount", "Year", "Month", "Day", "WeekDay", "Snow", "Holiday" from dbo.rental_data where Year < 2015'
    , @input_data_1_name = N'rental_train_data'
    , @params = N'@trained_model varbinary(max) OUTPUT'
    , @trained_model = @trained_model OUTPUT;
END;
GO
```
1.Save the model to a table
```sql
-- Save model to table 
TRUNCATE TABLE rental_rx_models;

DECLARE @model VARBINARY(MAX);
EXEC generate_rental_rx_model @model OUTPUT;

INSERT INTO rental_rx_models (model_name, model) VALUES('rxDTree', @model);

SELECT * FROM rental_rx_models;
```

The model is now saved in the database as a binary object.

## Step 3.2 Create stored procedure for prediction

We are now very close to deploying our predicting model so that we can consume it from our applications.
This last step includes creating a stored procedure that uses our model to predict the rental count for new data.

1.Create a stored procedure that predicts using our model

```sql
--Stored procedure that takes model name and new data as input parameters and predicts the rental count for the new data
DROP PROCEDURE IF EXISTS predict_rentalcount_new;
GO
CREATE PROCEDURE predict_rentalcount_new (@model VARCHAR(100),@q NVARCHAR(MAX))
AS
BEGIN
    DECLARE @rx_model VARBINARY(MAX) = (SELECT model FROM rental_rx_models WHERE model_name = @model);
    EXECUTE sp_execute_external_script 
        @language = N'R'
        , @script = N'
            require("RevoScaleR");

            #The InputDataSet contains the new data passed to this stored proc. We will use this data to predict.
            rentals = InputDataSet;
            
        #Convert types to factors
            rentals$Holiday = factor(rentals$Holiday);
            rentals$Snow = factor(rentals$Snow);
            rentals$WeekDay = factor(rentals$WeekDay);

            #Before using the model to predict, we need to unserialize it
            rental_model = unserialize(rx_model);

            #Call prediction function
            rental_predictions = rxPredict(rental_model, rentals);'
                , @input_data_1 = @q
        , @output_data_1_name = N'rental_predictions'
                , @params = N'@rx_model varbinary(max)'
                , @rx_model = @rx_model
                WITH RESULT SETS (("RentalCount_Predicted" FLOAT));
   
END;
GO
```

1.Execute the stored procedure to predict rental count for new data  

```sql
--Execute the predict_rentals stored proc and pass the modelname and a query string with a set of features we want to use to predict the rental count
EXEC dbo.predict_rentalcount_new @model = 'rxDTree',
       @q ='SELECT CONVERT(INT, 3) AS Month, CONVERT(INT, 24) AS Day, CONVERT(INT, 4) AS WeekDay, CONVERT(INT, 1) AS Snow, CONVERT(INT, 1) AS Holiday';
GO
```

## Step 3.3 Predict using native scoring (New!)
In SQL Server 2017, we are introducing a native predict function in TSQL. The native PREDICT function allows you to perform faster scoring using certain RevoScaleR or revoscalepy models using a SQL query without invoking the R or Python runtime. The following code sample shows how you can train a model in R using RevoscaleR "Rx" functions, save the model to a table in the DB and predict using [native scoring](https://docs.microsoft.com/en-us/sql/advanced-analytics/sql-native-scoring).

```sql
USE TutorialDB;

--STEP 1 - Setup model table for storing the model
DROP TABLE IF EXISTS rental_models;
GO
CREATE TABLE rental_models (
                model_name VARCHAR(30) NOT NULL DEFAULT('default model'),
                lang VARCHAR(30),
				model VARBINARY(MAX),
				native_model VARBINARY(MAX),
				PRIMARY KEY (model_name, lang)
				
);
GO


--STEP 2 - Train model using RevoscaleR
DROP PROCEDURE IF EXISTS generate_rental_R_native_model;
go
CREATE PROCEDURE generate_rental_R_native_model (@model_type varchar(30), @trained_model varbinary(max) OUTPUT)
AS
BEGIN
    EXECUTE sp_execute_external_script
      @language = N'R'
    , @script = N'
require("RevoScaleR")

rental_train_data$Holiday = factor(rental_train_data$Holiday);
rental_train_data$Snow = factor(rental_train_data$Snow);
rental_train_data$WeekDay = factor(rental_train_data$WeekDay);

if(model_type == "linear") {
	#Create a dtree model and train it using the training data set
	model_dtree <- rxDTree(RentalCount ~ Month + Day + WeekDay + Snow + Holiday, data = rental_train_data);
	trained_model <- rxSerializeModel(model_dtree, realtimeScoringOnly = TRUE);
	}

if(model_type == "dtree") {
	model_linmod <- rxLinMod(RentalCount ~ Month + Day + WeekDay + Snow + Holiday, data = rental_train_data);
	#Before saving the model to the DB table, we need to serialize it. This time, as a native scoring model
	trained_model <- rxSerializeModel(model_linmod, realtimeScoringOnly = TRUE);
	}	
'

    , @input_data_1 = N'select "RentalCount", "Year", "Month", "Day", "WeekDay", "Snow", "Holiday" from dbo.rental_data where Year < 2015'
    , @input_data_1_name = N'rental_train_data'
    , @params = N'@trained_model varbinary(max) OUTPUT, @model_type varchar(30)'
	, @model_type = @model_type
    , @trained_model = @trained_model OUTPUT;
END;
GO

--STEP 3 - Save model to table

--Line of code to empty table with models
--TRUNCATE TABLE rental_models;

--Save Linear model to table
DECLARE @model VARBINARY(MAX);
EXEC generate_rental_R_native_model "linear", @model OUTPUT;
INSERT INTO rental_models (model_name, native_model, lang) VALUES('linear_model', @model, 'R');

--Save DTree model to table
DECLARE @model2 VARBINARY(MAX);
EXEC generate_rental_R_native_model "dtree", @model2 OUTPUT;
INSERT INTO rental_models (model_name, native_model, lang) VALUES('dtree_model', @model2, 'R');

-- Look at the models in the table
SELECT * FROM rental_models;

GO
-- STEP 4  - Use the native PREDICT (native scoring) to predict number of rentals for both models
--Now lets predict using native scoring with linear model
DECLARE @model VARBINARY(MAX) = (SELECT TOP(1) native_model FROM dbo.rental_models WHERE model_name = 'linear_model' AND lang = 'R');
SELECT d.*, p.* FROM PREDICT(MODEL = @model, DATA = dbo.rental_data AS d) WITH(RentalCount_Pred float) AS p;
GO

--Native scoring with dtree model
DECLARE @model VARBINARY(MAX) = (SELECT TOP(1) native_model FROM dbo.rental_models WHERE model_name = 'dtree_model' AND lang = 'R');
SELECT d.*, p.* FROM PREDICT(MODEL = @model, DATA = dbo.rental_data AS d) WITH(RentalCount_Pred float) AS p;
GO

```

> Congrats you just deployed a predictive model in SQL Server using R! 
