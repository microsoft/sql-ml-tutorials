---
layout: page-steps
language: R
title: TBD
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

SELECT * FROM rental_rx_models;```

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
GO```

1.Execute the stored procedure to predict rental count for new data  

```sql
--Execute the predict_rentals stored proc and pass the modelname and a query string with a set of features we want to use to predict the rental count
EXEC dbo.predict_rentalcount_new @model = 'rxDTree',
       @q ='SELECT CONVERT(INT, 3) AS Month, CONVERT(INT, 24) AS Day, CONVERT(INT, 4) AS WeekDay, CONVERT(INT, 1) AS Snow, CONVERT(INT, 1) AS Holiday';
GO
```

> Congrats you just deployed a predictive model in SQL Server using R! 
