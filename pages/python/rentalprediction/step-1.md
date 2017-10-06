---
layout: page-steps
language: Python
title: Build a predictive model 
permalink: /python/rentalprediction/
redirect_from:
  - /python/
  - /python/rentalprediction/step/
  - /python/rentalprediction/step/1
---

>Predictive modeling is a powerful way to add intelligence to your application. It enables applications to predict outcomes against new data.
The act of incorporating predictive analytics into your applications involves two major phases: **model training** and **model deployment**

>In this tutorial, you will learn how to create a predictive model in Python and deploy it with SQL Server 2017 Machine Learning Services, **RC1 and above**.

>You can copy code as you follow this tutorial. All code is also available on [github](https://github.com/Microsoft/sql-server-samples/tree/master/samples/features/machine-learning-services/python/getting-started/rental-prediction).

## Step 1.1 Install SQL Server with in-database Machine Learning Services
{% include partials/install_sql_server_windows_ML.md %}

## Step 1.2 Install SQL Server Management Studio (SSMS)
Download and install SQL Server Management studio: [SSMS](https://msdn.microsoft.com/en-us/library/mt238290.aspx)

>Now you have installed a tool you can use to easily manage your database objects and scripts.


## Step 1.3 Enable external script execution              
Run SSMS and open a new query window. Then execute the script below to enable your instance to run Python scripts in SQL Server.

```sql
 EXEC sp_configure 'external scripts enabled', 1;
RECONFIGURE WITH OVERRIDE
```
You can read more about configuring Machine Learning Services [here](https://docs.microsoft.com/en-us/sql/advanced-analytics/r-services/set-up-sql-server-r-services-in-database).
**Don't forget to restart your SQL Server Instance after the configuration!** You can restart in SSMS by right clicking on the instance name in the Object Explorer and choose *Restart*.


>Now you have enabled external script execution so that you can run Python code inside SQL Server!

## Step 1.4 Install and configure your Python development environment   
1.You need to install a Python IDE. Here are some suggestions:

*Python Tools for Visual Studio (PTVS) [Download](https://microsoft.github.io/PTVS)

*PyCharm [Download](https://www.jetbrains.com/pycharm/)

2.To be able to use some of the functions in this tutorial, you need to point your Python environment to use the Python that comes with SQL Server 2017.
Point your Python environment to this path: *C:\Program Files\Microsoft SQL Server\YOURSQLSERVER\PYTHON_SERVICES*

>Terrific, now your SQL Server instance is able to host and run Python code and you have the necessary development tools installed and configured! 
The next section will walk you through creating a predictive model using Python.
    
