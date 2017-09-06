---
layout: page-steps
language: R
title: Build a predictive model
permalink: /R/rentalprediction/
redirect_from:
  - /R/
  - /R/rentalprediction/step/
  - /R/rentalprediction/step/1
---  


Predictive modeling is a powerful way to add intelligence to your application. It enables applications to predict outcomes against new data. The act of incorporating predictive analytics into your applications involves two major phases: **model training** and **model deployment**

In this tutorial, you will learn how to create a predictive model in R and deploy it with SQL Server 2016 (and above) Machine Learning Services.

Here is a [video](https://www.youtube.com/watch?v=YCyj9cdi4Nk&feature=youtu.be) that introduces in-database analytics and predictive modeling with R in SQL Server.

You can copy code as you follow this tutorial. All code is also available on [GitHub](https://github.com/NelGson/sql-server-samples/tree/master/samples/features/r-services/getting-started/rental-prediction)

## Step 1.1 Install SQL Server with in-database R / Machine Learning Services
{% include partials/install_sql_server_windows_ML.md %}

## Step 1.2 Install SQL Server Management Studio (SSMS)
Download and install SQL Server Management studio: [SSMS](https://msdn.microsoft.com/en-us/library/mt238290.aspx)

>Now you have installed a tool you can use to easily manage your database objects and scripts.

## Step 1.3 Enable external script execution              
Run SSMS and open a new query window. Then execute the script below to enable your instance to run R scripts in SQL Server.

```sql
 EXEC sp_configure 'external scripts enabled', 1;
RECONFIGURE WITH OVERRIDE
```
You can read more about configuring Machine Learning Services [here](https://docs.microsoft.com/en-us/sql/advanced-analytics/r-services/set-up-sql-server-r-services-in-database).
**Don't forget to restart your SQL Server Instance after the configuration!** You can restart in SSMS by right clicking on the instance name in the Object Explorer and choose *Restart*.
 
Optional: If you want, you can also [download SSMS custom reports](https://github.com/Microsoft/sql-server-samples/blob/master/samples/features/r-services/ssms-custom-reports/R%20Services%20-%20Configuration.rdl) available on github. 
The report "R Services - Configuration.rdl" for example provides an overview of the R runtime parameters and gives you an option to configure your instance with a button click.
To import a report in SSMS, right click on Server Objects in the SSMS Object Explorer and choose Reports -> Custom reports. Upload the .rdl file.

>Now you have enabled external script execution so that you can run R code inside SQL Server!

## Step 1.4 Install and configure your R development environment   
1. You need to install an R IDE. Here are some suggestions:

*R Tools for Visual Studio (RTVS) [Download](https://www.visualstudio.com/vs/rtvs)

*RStudio [Download](https://www.rstudio.com)

2. To be able to use some of the functions in this tutorial, you need to configure your R IDE to point to Microsoft R Client, which is an R Runtime provided by Microsoft. This runtime contains MS R Open packages.

Follow the steps 1 and 2 [here](https://msdn.microsoft.com/en-us/microsoft-r/r-client-get-started#configure-ide) to install R client and configure your R IDE tool.

>Terrific, now your SQL Server instance is able to host and run R code and you have the necessary development tools installed and configured! 
The next section will walk you through creating a predictive model using R.
    
