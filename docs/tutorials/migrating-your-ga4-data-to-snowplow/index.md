---
title: "Migrating your historical GA4 data to Snowplow"
description: "How to use the GA4 Migrator Tool"
date: "2023-07-10"
---
```mdx-code-block
import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';
```
The GA4 to Snowplow Migrator tool will take your historical GA4 data from BigQuery and convert it into Snowplow-formatted events. 
The tool runs locally on your machine, therefore no credentials or information is sent to Snowplow.

## Step 1 - Create a Service Account in Google Cloud Platform

The tool requires certain permissions in order to access and export your GA4 data. There are two options for creating the service account, either through the command line (CLI) or with Terraform. 

<Tabs groupId="service-account" queryString>
<TabItem value="CLI" label="CLI" default>
To create your service account using the cli, follow the steps below:  

#### 1 - Install gcloud

See https://cloud.google.com/sdk/docs/install-sdk for installation instructions

#### 2 - Authorize gcloud

```shell
gcloud auth login
```

#### 3 - Set the following variables in your terminal

- `PROJECT_ID`: the name of the GCP project in which the GA4 data is stored
- `SERVICE_ACCOUNT_NAME`: The name to give the new service account
- `ROLE_NAME`: The name to give the new custom role
- `KEY_FILE_NAME`: The name to give the the service account key file that is downloaded. Use the .json extension
e.g.
```shell
export PROJECT_ID=project_123_abc
export SERVICE_ACCOUNT_NAME=migrator-service-account
export ROLE_NAME=ga4_exporter_role
export KEY_FILE_NAME=migrator_service_account_key.json
```

#### 4 - Create a service account 

```shell
gcloud iam service-accounts create $SERVICE_ACCOUNT_NAME --display-name $SERVICE_ACCOUNT_NAME
```
*Note: this will error if the service account name uses underscores, or if the service account already exists*

#### 5 - Create a custom role

```shell
gcloud iam roles create $ROLE_NAME --project=$PROJECT_ID --permissions=bigQuery.tables.export,bigQuery.tables.getData,bigQuery.tables.get,bigQuery.tables.create,bigQuery.tables.update,bigQuery.tables.updateData,bigQuery.tables.list,bigQuery.tables.delete,bigQuery.jobs.create,storage.objects.create,storage.objects.get,storage.objects.list,storage.objects.delete,storage.buckets.getIamPolicy,storage.buckets.setIamPolicy,storage.buckets.create,storage.buckets.get,bigQuery.datasets.create,bigQuery.routines.create,bigQuery.routines.get
```
*Note this will error if you already have a custom role with this name, or your role name contains dashes*


#### 6 - Grant the custom role to the service account

```shell
gcloud projects add-iam-policy-binding $PROJECT_ID --member serviceAccount:$SERVICE_ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com --role projects/$PROJECT_ID/roles/$ROLE_NAME
```

#### 7 - Create a key for the service account

```shell
gcloud iam service-accounts keys create $KEY_FILE_NAME --iam-account $SERVICE_ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com
```
*This will download the key to your local machine in the same folder in which you are running the command*
</TabItem>

<TabItem value="Terraform" label="Terraform">

#### 1 - Install gcloud

See https://cloud.google.com/sdk/docs/install-sdk for installation instructions

#### 2 - Authorize gcloud

```shell
gcloud auth application-default login
```

#### 3 - Install Terraform

See https://learn.hashicorp.com/tutorials/terraform/install-cli for installation instructions. If you have previously run one of our quickstarts, you will already have Terraform installed.

#### 4 - Run Terraform

There is a file in the root directory of the migrator tool called `manage_service_account.tf`. This file contains the Terraform code to create the service account.
To create the service account in the root directory of the migrator tool, run the following commands:
```shell
terraform init
terraform apply
```
You will be prompted to enter values for role ID, service account ID and project ID. The role ID is the name of the custom IAM role that will be created. The service account ID is the name of the service account that will be created.

You can create a `terraform.tfvars` file if you do not want to re-enter the variables each time you run Terraform.
E.g.
```shell
project_id = "project-123-abc"
service_account_id = "ga4-migrator-service-account"
role_id = "ga4_migrator_role"
```
The keyfile will be created in the same directory you ran Terraform.
</TabItem>
</Tabs>

## Step 2 - How to run the GA4 Migrator docker container

Set the environment variable `SERVICE_ACCOUNT_KEY_FILE` to the full path for the GCP service account key file.
```shell
export SERVICE_ACCOUNT_KEY_FILE=[path/to/service_account/keyfile.json]
```
<details>
If you are also going to run the export from BigQuery to Snowflake, set the following environment variables. This user must have ACCOUNTADMIN permissions in Snowflake.
Also ensure that the GCS bucket you are using for the export already exists and is in the same location as your BigQuery table.

```shell
export SNOWFLAKE_USER=[snowflake_user]
export SNOWFLAKE_PASSWORD=[snowflake_password]
```
</details>

If you do not already have Docker, installation instructions can be found [here](https://docs.docker.com/get-docker/).
To run the docker container using docker compose. Run the following command:
```shell
docker compose up --build
```

## Step 3 - Using the migrator tool UI

Once the docker container is running, the URI to access it will be printed to your terminal.
There are 4 steps to the UI:
### Step 1 - Provide your GCP configuration

The following information is required:
- `Dataset location`: The location of your GA4 dataset in BigQuery
- `Project ID`: The name of the GCP project in which the GA4 data is stored
- `Dataset`: The name of your GA4 dataset in BigQuery. 
- `Table`: The name of your GA4 table in BigQuery. The default value is set to `events_*`. If you only want to import a certain time period (not all historical data) leave this as `events_*` and provide the specific date range below. This field should only be changed if you have a custom table name.
- `Start date`: The first date that you want to query data from. This is inclusive.
- `End date`: The date that you want to query data until. This is inclusive. If you only want to query one date, set both start and end dates to that date.

### Step 2 - Map Snowplow events

Although GA4 and Snowplow both track events, GA4 does not have a concept of [entities](https://docs.snowplow.io/docs/understanding-your-pipeline/entities/). As such, the migrator gathers the parameters from each of your GA4 events, and adds similar parameters to a Snowplow entity definition. For example, if you have two events, `add_to_cart` and `view_cart`, with each containing a property called `page_title`, a Snowplow entity called context_vendor_page_1_0_0 will be created, and the `page_title` parameter will be added to it. In addition, for each of your events, a Snowplow self-describing event will also be created, containing the parameters from each GA4 event.

The interface in the UI allows you to modify any of the pre-created entities, or create your own entities. You can remove parameters from the events (for example if you wish to add the parameter to an entity), but you cannot add any new parameters to events. Events will be migrated contatining the same parameters as they had in GA4, minus any that you remove.

When you create a new entity, you can choose parameters from the events that are shown in the UI. Note that you only have to add a parameter to an entity once, and it will be filled in the entity column from any event that has that parameter.
For example, if you move the `page_title` parameter (which exists in both `add_to_cart` and `view_cart` events) from the `add_to_cart` event to a new `cart` entity, in the newly created table the `page_title` parameter will be filled in on both the `add_to_cart` and `view_cart` events. You don't have to copy the parameter from both events to the new entity.

*Note: When naming new entities, only use alphanumeric characters and underscores. Do not use spaces or special characters.*

### Step 3 - Map additional columns

In this step, the mappings from GA4 to the native Snowplow columns are being defined. These definitions will be passed to dbt which will create and run the SQL for your new Snowplow table.
There are defaults that are set, but these can be customized if you wish.
`app_id`, for example, must be changed, substituting the placeholder values for the correct values for your GA4 data.

The SQL written for the mapping will be placed directly into the code, so it must be valid as a standalone single-line expression:
```sql
...
<your input SQL> AS snowplow_field
...
```
For example, the following inputs:
| Snowplow_field | mapping_sql |
| --------------- | ----------- |
| page_urlscheme  | regexp_extract(page_url, r'(.*?):') |
| page_urlhost    | {{dbt_utils.get_url_host(field='page_url')}} |
| geo_zipcode     | NULL        |
| app_id          | CASE WHEN stream_id = '4271243942' THEN 'Snowplow' ELSE NULL END |

Will be inserted into the SQL code as follows:
```SQL
...
regexp_extract(page_url, r'(.*?):') AS page_urlscheme,
{{dbt_utils.get_url_host(field='page_url')}} AS page_urlhost,
"NULL" AS geo_zipcode,
CASE WHEN stream_id = '4271243942' THEN 'Snowplow' ELSE NULL END AS app_id,
...
```
- Some Snowplow fields have been pre-extracted from the GA4 data to make the SQL more readable, and to make using these fields in further calculations easier. These are:
    - `domain_sessionidx`
    - `page_url`
    - `page_title`
    - `page_referrer`
    - `mkt_medium`
    - `mkt_source`
    - `mkt_term`
    - `mkt_content`
    - `mkt_campaign`
    - `domain_sessionid`
    
    As such, when writing SQL in the mapping, you can use these fields directly, e.g.:

    | Snowplow_field | mapping_sql |
    | --------------- | ----------- |
    | page_urlquery   | regexp_extract(page_url, r'(?:.*)\\?(.*)')|
    
- If you wish to use a parameter from `event_params`, the migrator has a [UDF](https://cloud.google.com/bigquery/docs/user-defined-functions) that will extract the parameter from the event_params column. The UDF is called `ga4_migrator_parse_params`. To use it, the SQL must be written in the following format:
    ```sql
    JSON_VALUE(ga4_migrator_parse_params(event_params), '$.parameter_to_extract')
    ```
    E.g.:
    ```sql
    JSON_VALUE(ga4_migrator_parse_params(event_params), '$.page_title')
    ```
    
The SQL that you input must conform to other strict requirements:
- When creating or updating any of the mapping fields and using a dbt package, it must be written in the following format, including the curly braces and the package reference: `{{dbt_package.function('column_name')}}`, e.g. `{{dbt_utils.get_url_host('page_referrer')}}`.

- In the mapping table, there are two fields that should not be altered:
`geo_country_temp` and `geo_region_temp`. These are required for the `geo_country` and `geo_region` fields to be populated correctly.

### Step 4 - Transform data

Click the `Run transformation` button to run the migration. Behind the scenes, this runs dbt. dbt will create the UDF that extracts the event_params, seed two tables that are used for geographical mapping, and create the final table that contains the migrated data.

### Step 5 - Import to Snowflake

This step is optional, and is only if you wish to import the data into Snowflake. If you do not wish to import the data, you can skip this step.
Provide the following parameters:
- `Region`: The region in which your Snowflake account is located
- `Account identifier`: The account identifier for your Snowflake account
- `Storage integration name`: The name you would like to give the storage integration that will be created
- `Stage name`: The name you would like to give the stage that will be created
- `Warehouse name`: The name of the warehouse that you would like to use to import the data
- `Database name`: The name of the database that you would like to the new table to be created in
- `Schema name`: The name of the schema that you would like to the new table to be created in
- `Table name`: The name of the table that will be created

Then click `Run import` and the data will be imported to Snowflake. The data will be imported to the table that you specified in the `Table name` field. If the table already exists, it will be dropped and recreated.