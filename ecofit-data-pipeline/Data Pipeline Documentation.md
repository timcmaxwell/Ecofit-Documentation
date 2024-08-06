## **Overview**
The Ecofit data pipeline is a highly automated Extract, Transform, Load (ETL) service taking raw Energy Performance Certificate (EPC) data collected from the Scottish, Northern Irish and England and Wales Government, and with minimal user interaction adding it to the Ecofit Database. The pipeline has been developed to remove the abnormalities of EPC data, add new data fields to allow for further manipulation and calculation by users, and retain a schema allowing for future scalability without loss of data.

## **Good to know**
Prefixes for file names, tools, and functions are important and follow a designated pattern throughout the pipeline. This is useful for readability, to differentiate between jobs, workflows, and crawlers, and to automate running the pipeline:

- Workflows start with `w_`
- Jobs start with `j_`
- Crawlers start with `c_`
- Triggers start with `t_`
- Cleaning functions start with `clean_`

## **Architecture**

While the pipeline is a collection of jobs and crawlers on AWS, these are collated into three distinct workflows for each nation: `w_extract_{nation}`, `w_transform_{nation}`, and `w_load_{nation}`.

### Pre-Pipeline Configuration

All data required to be run against the pipeline should be uploaded to the `raw-epc-data` bucket on AWS S3. For each nation, the EPC data can be collected from:

- **England and Wales**
    - [EPC Search API](https://www.api.gov.uk/dluhc/energy-performance-certificates-display-public-buildings-certificates-search-api)
    - [EPC Open Data](https://epc.opendatacommunities.org/)
- **Scotland**
    - [Scottish EPC Data](https://statistics.gov.scot/data/domestic-energy-performance-certificates)
    - Raw Scottish data contains two header fields. The `preprocess_raw_scotland_csvs.py` script in the `ecofit-utils` repository should be used to correct this before it is uploaded to AWS S3.
- **Northern Ireland** *(Pipeline in development)*
    - Contact within NI government: 
    - Anna.McNulty@finance-ni.gov.uk
    - Ainsley.Mitchell@finance-ni.gov.uk
    - Kevin.Collins@finance-ni.gov.uk
    - Info.EPB@finance-ni.gov.uk
    - The data can be accessed from emailing one of the above addresses with a request to access the latest Northern Irish EPC Domestic Dataset. They will provide a link on Box (https://www.box.com) to access a csv file that will contain all data from 2008 until now. This database continuously updates. It is recommended the database is updated every quarter in line with the Scotland. 


Data for England and Wales is currently uploaded on the last working day of every month, whereas Scottish data is released quarterly. A separate Lambda function is run to check for updates for both nations every day between 09:00 and 09:15. An email alert is sent to the distribution list every morning indicating data changes or not.

It is important to ensure that any document not containing EPC data is not uploaded to AWS S3, as this may cause the crawlers to have unintended consequences. Specifically, these are commonly the licence agreements or data glossaries. This will cause errors in the crawlers.

### Extract

The first stage of the pipeline is Extract. This can be run for each nation using the `w_extract_{NATION}` workflow. These workflows differ slightly for the different nations:

### England

- `t_crawl_raw_england`: Trigger to run crawlers.
- `c_england_certificates`: Crawler for raw crawl certificates data and update the Glue catalog table.
- `c_england_recommendations`: Crawler for raw crawl recommendations data and update the Glue catalog table.
- `t_england_extract_job`: Trigger to run extract job. Waits for both crawlers to succeed before running.
- `j_extract_england`: Glue Job to:
    - Map column names and data types to the Ecofit standard.
    - Join separate Certificate and Recommendation data into a nested `ArrayType` for each entry. English data does not include a typical saving at this time, so is set as an empty string at this point.
    - Save the data to the `transformed-epc-data` AWS S3 bucket.
- `t_england_cleaning_preparation`: Trigger to run cleaning preparation job.
- `j_cleaning_preparation`: Glue job to:
    - Remove unchanged EPCs.
    - Check for new values in the categorical columns and write them to S3.
    - New improvements’ costs and texts are checked for new values and output to S3.
    - Data is saved to the `prepared-data` bucket on AWS S3 for use in the next stage of the pipeline.

### Wales

- `t_crawl_raw_wales`: Trigger to run crawlers.
- `c_wales_certificates`: Crawler for raw certificates data and update the Glue catalog table.
- `c_wales_recommendations`: Crawler for raw crawl recommendations data and update the Glue catalog table.
- `t_wales_extract_job`: Trigger to run extract job. Waits for both crawlers to succeed before running.
- `j_extract_wales`: Glue Job to:
    - Map column names and data types to the Ecofit standard.
    - Join separate Certificate and Recommendation data into a nested `ArrayType` for each entry. English data does not include a typical saving at this time, so is set as an empty string at this point.
    - Save the data to the `transformed-epc-data` AWS S3 bucket.
- `t_wales_cleaning_preparation`: Trigger to run cleaning preparation job.
- `j_cleaning_preparation`: Glue job to:
    - Remove unchanged EPCs.
    - Check for new values in the categorical columns and write them to S3.
    - New improvements’ costs and texts are checked for new values and output to S3.
    - Data is saved to the `prepared-data` bucket on AWS S3 for use in the next stage of the pipeline.

### Scotland

- `t_crawl_raw_scotland`: Trigger to run crawlers.
- `c_scotland_raw`: Crawler for raw data to update the Glue catalog table.
- `t_scotland_extract_job`: Trigger to run extract job.
- `j_extract_scotland`: Glue Job to:
    - Map column names and data types to the Ecofit standard.
    - Transform the empty `mains_gas_flag` column to indicate whether a property is on mains gas with a string value.
    - Run a utility function to transform columns with mixed data types. Currently this is only used on `open_fireplaces_count` which contains strings and integers.
    - Process improvements to extract and standardise required data.
    - Save the data to the `transformed-epc-data` AWS S3 bucket.
- `t_scotland_cleaning_preparation`: Trigger to run cleaning preparation job.
- `j_cleaning_preparation`: Glue job to:
    - Remove unchanged EPCs.
    - Check for new values in the categorical columns and write them to S3.
    - New improvements’ costs and texts are checked for new values and output to S3.
    - Data is saved to the `prepared-data` bucket on AWS S3 for use in the next stage of the pipeline.

Individual nation scripts' concurrency limit is 1 (e.g., `j_extract_scotland`), whereas shared parameterised scripts (e.g., `j_cleaning_preparation`) limit is 3.

`j_cleaning_preparation` is computationally expensive as it involves checking every EPC in the input data against every record in the Ecofit database, so this is configured to run with the maximum available DPUs when the job is being run 3 times concurrently. To improve speed, if fewer than 3 nations are running, the number of DPUs can be adjusted.

> 💡 **UPDATE:** `j_cleaning_preparation` has been updated to use the date of the previous EPC release cut off to look for new EPCs and not check the database. Therefore, the `CUTOFF_DATE` parameter needs updating in this script before the pipeline is run.

### Manual Preparation

Following successful completion of the Cleaning Preparation jobs for each nation, the new mappings values identified must be added to the AWS RDS MySQL. These can be collected locally using the AWS CLI, or downloaded from the AWS Console. The `collect.py` script in `ecofit-utils` is useful for identifying and collating new and unique values.

The `epc-value-mappings` database is within a VPC on AWS, with incoming traffic security rules and can only be accessed by whitelisted IPs. A local SQL client can be used for adding new values to the mappings tables.

### Transform

Following the updating of the RDS database, it is useful to manually inspect each nation in AWS Athena to ensure that no unexpected errors have occurred. Future development will create a script here to ensure that all new values have been mapped before the rest of the pipeline is run. Following this, the `w_transform_{nation}` workflows can be run.

Presently each nation’s `w_transform_{NATION}` workflow contains 2 tasks:

- `t_{nation}_data_cleaning`: Trigger to run data cleaning Glue Job.
- `j_data_cleaning`: Parameterised Glue Job taking a nation name as a parameter:
    - Remove outliers in numerical columns as identified by the Ecofit identification process.
    - Transform nested data categories for energy and environmental efficiency columns.
    - Map categorical values to those in AWS S3.
    - Sort data types to ensure appropriate use of dates, strings, integers, floats, and booleans.
    - Deal with Null and NA values in all columns. Calculate and add custom Ecofit data and data on the use of renewable technologies.
    - Restructure improvement information and extract key data into new attributes.
    - Structure data to prepare for loading into a document-based database (AWS DocumentDB).

The job can be run concurrently up to 3 times, with up to 30 workers for each job. Run times vary from 10 minutes to 4.5 hours depending on the volume of data being processed. Cleaned data is written to the AWS S3 `cleaned_data` bucket. At this stage, the cleaned data should be verified for each nation in AWS Athena or another parquet viewer tool to ensure successful completion of the cleaning process.

### Load

The last workflow is used to load the data into the required database. It contains a trigger and a job:

- `t_{nation}_load_job`: Trigger to run load data job.
- `j_data_loading`: Parameterised Glue Job taking a nation name as a parameter:
    - Loads data into the `ecofit-database-dev` on AWS DocumentDB for further testing.
    - Contains a Connection to AWS DocumentDB to connect to the dev-database, which stops this job from being parameterised for the choice of database.

Currently, a separate script is run for loading data between databases, but in future it would be useful to use `j_data_loading`.

## **Ecofit Retrofit Databases**

There are three editions of the Ecofit Retrofit database, previously hosted in MongoDB Atlas and now in separate clusters on AWS DocumentDB.

Each database contains 3 collections:

- **Property** - containing cleaned Ecofit data collected from the pipeline.
- **LocationRenewableCost** - containing data collected in agreement with MCS on Renewable Energy installation costs for each local authority.
- **User** - containing user information for each database. All passwords are required to be hashed in the database and plain text passwords will throw an error.

Furthermore, there is an index for UPRNs and a compound index for addresses and postcodes in the Property collection of each database.
