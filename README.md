# **ML-Inventory-Activity**

> Deploys a Jenkins Pipeline which automates the daily aggregation and transformation of data and insight relating to inventory of active listings on the cars.com platform via automic. Built in a python enviroment using Spark via the pySpark API, S3 spot instances are leveraged to pull from various AWS data storage solutions. 

## Table of contents
1. config
    - There are several config files, for dev, prod, stage, and forge. 
        - The forge file is used for spinning up a forge instance on S3.
        - The other config files are used for identifying the various filepaths in the data-lake from which the data is being pulled from.
2. resources
    - The resource files have two main functions, one is defining the Automic enviroment for the workflow and defining the schema of the inventory_activity schema.  
3. scripts
    - The shell scripts found here are used for initating the workflow, creating s3 clusters and initating their enviroments and starting the python scripts. They also do error handling, basic logging functions, and hold the looping logic for completing backfill jobs.
4. src
    - This source folder contains all of the python scripts used by this workflow. Shared definitions are stored within the functions.py file.
        - inventory_activity_main holds the main function which executes the ETL process in a linear path.
        - active_listing_tmp extracts current data from VEHICLE_TEMP and filters it against data from vehicle_history_temp to create a subset of unique active inventory records in a dataframe object.
        - listing_activity pulls weighted impressions and leads to build an accurate activity tracking metric per listing.
        - active_listing_transform_tmp aggregates the listing into natural groups based on physical qualaties of the vehicles and by soft characteristics such as location (by zipcode).
5. miscellaneous
    - Jenkinsfile is a groovy script used to build the Pipeline and link it to nescessary dependencies. 
    

#### This repository was built by and for the MLaaS team at Cars.com. 

