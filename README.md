# **ML-Inventory-Activity**

> Deploys a Jenkins Pipeline which automates the daily aggregation and transformation of data and insight relating to inventory of active listings on the cars.com platform via automic. Built in a python enviroment using Spark via the pySpark API, S3 spot instances are leveraged to pull from various AWS data storage solutions. 

## Table of contents
1. config
    - There are several config files, for dev, prod, stage, and forge. 
        - The forge file is used for spinning up a forge instance on S3.
        - The other config files are used for identifying the various filepaths in the data-lake from which the data is being pulled from.
2. resources
    - The resource files have two main functions, one is defining the Automic enviroment for the workflow and serving as config files for the python scripts by providing locations for various required tables. 
3. scripts
    - The shell scripts found here are used for initating the workflow, creating s3 clusters and initating their enviroments and starting the python scripts. They also do error handling, basic logging functions, and hold the looping logic for completing backfill jobs.
4. src
    - This source folder contains all of the python scripts used by this workflow. Shared definitions are stored within the functions.py file.
        - inventory_activity_main holds the main function which executes the ETL process in a linear path.
        - active_listing_tmp extracts current data from VEHICLE_TEMP and filters it against data from vehicle_history_temp to create a subset of unique active inventory records in a dataframe object.
        - listing_activity pulls weighted impressions and leads to build an accurate activity tracking metric per listing.
        - active_listing_transform_tmp aggregates the listing into natural groups based on physical qualaties of the vehicles and by soft characteristics such as location (by zipcode).
        - listing_activity_load - -
5. miscellaneous
    - Jenkinsfile is a groovy script used to build the Pipeline and link it to nescessary dependencies. 

## Process


Once the environment is built, the python scripts are initated.
Taken as parameteres from run_main.sh, the python scrips are initated with a starting context of date and a yaml file. The main function loads the yaml data as an object which is used as the configs. This object, as well as the spark instance and the date are passesd to active_listing_tmp.main. 

The main function here passes the s3 location of the given day's current partition of the vehicle table (pulled from configs) as well as the spark object to the read_vehicles function. This function reads and returns the data (parquet file) in a dataframe object. The function then creates a new dataframe using spark from the historical partition of the same vehicle table. The two dataframes are then passed to get_veh_df to return a consolidated df which contains the union of all active listings back to the main function as active_listing_temp_df. 

The df is passed into listing_activity_temp.main along with the configs and the date. From here, imp_daily and dw_lead are read into two temp tables which are used to create vehicle_temp. A series of intermediate dfs are built in the following order; srp_imp_temp, als_temp_df, imp_temp1-2-3, lead_temp. These intermediate temp dfs are used to create tmp_al_df. This df contains {?} and is returned to the original main function as listing_activity_tmp_df. 

Listing_activity_tmp_df is then passed to active_listing_transform_tmp.main which creates several temp tables pulling directly from s3 {make, make_model, model_year, vehicle_trim, stock_type, bodystyle, zip_code, dma_zip, dealer_review_rollup, dealer_review, vehicle_definition}. Zip_code and dma_zip are are joined on zip_code_id to create a new fd and purchase_flag is used to create a df which contains {?}. The two new dfs are then registered as temp tables and joined together with the previously created temp tables to create veh.df which holds the physical and soft features of each active listing as veh_df which is returned to the main function as active_listing_transform_df. 

This df along with listing_activity_tmp_df, the date, and the configs file are passed to inventory_activity_load.main which registers them and a few tables from s3 as temp tables {vehicle_daily_temp, vehicle_review_rollup_temp, customer_temp, activity_customer_temp, dealer_sales_locations}. veh_reviews is used to create a df that contains reviews for each vehicle based on make and model which is passed into the final_join_df function with the current date. This creates the final inventory activity table that joins all features regarding the currently active listings. A new schema is then created using lists of pre-defined columns sorted by data-type. Data from carguru, autotrader and truecar are joined to the final df via the vin identifier of each vehicle. Daily attributes of each vehicle are then joined to the final df. The dataframe is then partitioned into 50 smaller files and they are written to s3a://cars-data-lake-landing-dev/ml_usr-dev@cars.com/inventory_activity/filedate={date}. 

If this process is being run as a backfill job, the start.sh file will rerun the program for the following day, if this is run as the daily workflow or is the last day in a backfill job, the s3 clusters will be destroyed and a followup job will move the files from landing to core. 

## Business Use

This workflow generates a lot of relevant information on active listings that are used for reporting purposes by the business intelligence team and a lot of the data is also consumed by models created by the data science team. This job is highly essential as it has a lot of dependent process on the analytics side and also impacts the frontend of the cars.com platform from the data it gathers. 

#### This repository was built by and for the MLaaS team at Cars.com. 

