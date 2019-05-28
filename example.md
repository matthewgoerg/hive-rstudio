**Dashboard preview**

![Dashboard](https://github.com/matthewgoerg/hive-rstudio/blob/master/dashboard.png)

**ML model comparison Preview**

![Dashboard](https://github.com/matthewgoerg/hive-rstudio/blob/master/lift_clicks.PNG)

# Machine Learning and Dashboards on the Cloud

> Using Google Cloud Platform, Hive, Spark, and RStudio

In this project we will:
- Find 100GB of real click-through data
- Download the data into Google's Cloud Storage
- Create a Hive/Spark cluster hosted on the Google Cloud Platform
- Query the data using Hive to prepare it for analysis
- Install RStudio on the cluster
- Build an interactive dashboard using R Shiny
- Execute a machine learning pipeline using the R interface for Spark's Machine Learning Library (MLlib)

**Technology stack:**

- Google Cloud Platform
- HDFS
- Apache Hive
- Apache Spark
- RStudio
- sparklyr
- Shiny
- BigQuery

---

## Table of Contents (Optional)

> If you're `README` has a lot of info, section headers might be nice.

- [Set up GCP project](#gcp)
- [Hive](#hive)
- [RStudio environment](#rstudio)
- [Dashboards](#rstudio)
- [Machine learning](#machinelearning)
- [Discussion](#discussion)
- [Support](#support)
- [License](#license)


---

## Set up GCP project

For this project, you will need a Google Cloud Platform account and project. The instructions to do this can be found here. 
Once you have the account and project, click on Cloud Shell and enter the following commands to create a DataProc cluster:

```cloudshell
export REGION=us-central1
export ZONE=us-central1-a
export PROJECT=<your-project-name>
gcloud config set compute/zone $ZONE

gcloud services enable dataproc.googleapis.com sqladmin.googleapis.com

export PROJECT=$(gcloud info --format='value(config.project)')
gsutil mb -l $REGION gs://$PROJECT-warehouse

gcloud sql instances create hive-metastore2 \
	--database-version="MYSQL_5_7" \
	--activation-policy=ALWAYS \
	--gce-zone $ZONE

gcloud dataproc clusters create hive-cluster \
	--scopes sql-admin \
	--image-version 1.3 \
	--initialization-actions 

gs://dataproc-initialization-actions/cloud-sql-proxy/cloud-sql-proxy.sh \
	--properties hive:hive.metastore.warehouse.dir=gs://$PROJECT-warehouse/datasets \
	--metadata "hive-metastore-instance=$PROJECT:$REGION:hive-metastore2"
```

Now download the Google Cloud SDK onto your machine. You will also need a SSH client. I am using PuTTY. 
Enter the following code to launch PuTTY and SSH into your cluster.

```googlecloudsdk
gcloud compute ssh ^
	--zone=us-central1-a ^
	--project=$PROJECT ^
 	hive-cluster-m -- ^
	-L 8787:localhost:8787
```

After you enter your credentials, you will get a command line. Enter this line to launch the Hive environment.

```clustercommandline
beeline -u jdbc:hive2://localhost:10000/default -n *rstudio*@*hive-cluster-m* -d org.apache.hive.jdbc.HiveDriver
```

## Hive

The Hive query language (HQL) is substantially similar to standard SQL. For example, to view all the tables in 
your environment, type this:

```hiveql
SHOW TABLES;
```
Let's move our data from Google Cloud Storage (GCS) to Hive.

```hiveql
DROP TABLE IF EXISTS click_data;

CREATE EXTERNAL TABLE click_data 
(label INT, 
	int_feature_01 INT,
	int_feature_02 INT,
	int_feature_03 INT,
	int_feature_04 INT,
	int_feature_05 INT,
	int_feature_06 INT,
	int_feature_07 INT,
	int_feature_08 INT,
	int_feature_09 INT,
	int_feature_10 INT,
	int_feature_11 INT,
	int_feature_12 INT,
	int_feature_13 INT,
	cat_feature_01 VARCHAR(8),
	cat_feature_02 VARCHAR(8),
	cat_feature_03 VARCHAR(8),
	cat_feature_04 VARCHAR(8),
	cat_feature_05 VARCHAR(8),
	cat_feature_06 VARCHAR(8),
	cat_feature_07 VARCHAR(8),
	cat_feature_08 VARCHAR(8),
	cat_feature_09 VARCHAR(8),
	cat_feature_10 VARCHAR(8),
	cat_feature_11 VARCHAR(8),
	cat_feature_12 VARCHAR(8),
	cat_feature_13 VARCHAR(8),
	cat_feature_14 VARCHAR(8),
	cat_feature_15 VARCHAR(8),
	cat_feature_16 VARCHAR(8),
	cat_feature_17 VARCHAR(8),
	cat_feature_18 VARCHAR(8),
	cat_feature_19 VARCHAR(8),
	cat_feature_20 VARCHAR(8),
	cat_feature_21 VARCHAR(8),
	cat_feature_22 VARCHAR(8),
	cat_feature_23 VARCHAR(8),
	cat_feature_24 VARCHAR(8),
	cat_feature_25 VARCHAR(8),
	cat_feature_26 VARCHAR(8))
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
LOCATION 'gs://$PROJECT/click/';
```

```hiveql
SELECT COUNT(*) FROM click_data;
```

This table is 192 million rows long. Even though we built a cluster, it is a good idea to 
test out our machine learning pipeline on a subset of the data. Once, we know it's working,
we can go back and run it on our entire data. The following code will randomly sample 10,000
rows from click_data into a new table called click_sample.

```hiveql
DROP TABLE IF EXISTS click_sample;

CREATE TABLE click_sample
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
STORED AS textfile
AS SELECT
	CASE WHEN label IS NULL THEN 0 ELSE label END AS label,
	CASE WHEN int_feature_01 IS NULL THEN 0 ELSE int_feature_01 END AS int_feature_01,
	CASE WHEN int_feature_02 IS NULL THEN 0 ELSE int_feature_02 END AS int_feature_02,
	CASE WHEN int_feature_03 IS NULL THEN 0 ELSE int_feature_03 END AS int_feature_03,
	CASE WHEN int_feature_04 IS NULL THEN 0 ELSE int_feature_04 END AS int_feature_04,
	CASE WHEN int_feature_05 IS NULL THEN 0 ELSE int_feature_05 END AS int_feature_05,
	CASE WHEN int_feature_06 IS NULL THEN 0 ELSE int_feature_06 END AS int_feature_06,
	CASE WHEN int_feature_07 IS NULL THEN 0 ELSE int_feature_07 END AS int_feature_07,
	CASE WHEN int_feature_08 IS NULL THEN 0 ELSE int_feature_08 END AS int_feature_08,
	CASE WHEN int_feature_09 IS NULL THEN 0 ELSE int_feature_09 END AS int_feature_09,
	CASE WHEN int_feature_10 IS NULL THEN 0 ELSE int_feature_10 END AS int_feature_10,
	CASE WHEN int_feature_11 IS NULL THEN 0 ELSE int_feature_11 END AS int_feature_11,
	CASE WHEN int_feature_12 IS NULL THEN 0 ELSE int_feature_12 END AS int_feature_12,
	CASE WHEN int_feature_13 IS NULL THEN 0 ELSE int_feature_13 END AS int_feature_13,
	CASE WHEN cat_feature_01 = '' THEN 'EMPTY' ELSE cat_feature_01 END AS cat_feature_01,
	CASE WHEN cat_feature_02 = '' THEN 'EMPTY' ELSE cat_feature_02 END AS cat_feature_02,
	CASE WHEN cat_feature_03 = '' THEN 'EMPTY' ELSE cat_feature_03 END AS cat_feature_03,
	CASE WHEN cat_feature_04 = '' THEN 'EMPTY' ELSE cat_feature_04 END AS cat_feature_04,
	CASE WHEN cat_feature_05 = '' THEN 'EMPTY' ELSE cat_feature_05 END AS cat_feature_05,
	CASE WHEN cat_feature_06 = '' THEN 'EMPTY' ELSE cat_feature_06 END AS cat_feature_06,
	CASE WHEN cat_feature_07 = '' THEN 'EMPTY' ELSE cat_feature_07 END AS cat_feature_07,
	CASE WHEN cat_feature_08 = '' THEN 'EMPTY' ELSE cat_feature_08 END AS cat_feature_08,
	CASE WHEN cat_feature_09 = '' THEN 'EMPTY' ELSE cat_feature_09 END AS cat_feature_09,
	CASE WHEN cat_feature_10 = '' THEN 'EMPTY' ELSE cat_feature_10 END AS cat_feature_10,
	CASE WHEN cat_feature_11 = '' THEN 'EMPTY' ELSE cat_feature_11 END AS cat_feature_11,
	CASE WHEN cat_feature_12 = '' THEN 'EMPTY' ELSE cat_feature_12 END AS cat_feature_12,
	CASE WHEN cat_feature_13 = '' THEN 'EMPTY' ELSE cat_feature_13 END AS cat_feature_13,
	CASE WHEN cat_feature_14 = '' THEN 'EMPTY' ELSE cat_feature_14 END AS cat_feature_14,
	CASE WHEN cat_feature_15 = '' THEN 'EMPTY' ELSE cat_feature_15 END AS cat_feature_15,
	CASE WHEN cat_feature_16 = '' THEN 'EMPTY' ELSE cat_feature_16 END AS cat_feature_16,
	CASE WHEN cat_feature_17 = '' THEN 'EMPTY' ELSE cat_feature_17 END AS cat_feature_17,
	CASE WHEN cat_feature_18 = '' THEN 'EMPTY' ELSE cat_feature_18 END AS cat_feature_18,
	CASE WHEN cat_feature_19 = '' THEN 'EMPTY' ELSE cat_feature_19 END AS cat_feature_19,
	CASE WHEN cat_feature_20 = '' THEN 'EMPTY' ELSE cat_feature_20 END AS cat_feature_20,
	CASE WHEN cat_feature_21 = '' THEN 'EMPTY' ELSE cat_feature_21 END AS cat_feature_21,
	CASE WHEN cat_feature_22 = '' THEN 'EMPTY' ELSE cat_feature_22 END AS cat_feature_22,
	CASE WHEN cat_feature_23 = '' THEN 'EMPTY' ELSE cat_feature_23 END AS cat_feature_23,
	CASE WHEN cat_feature_24 = '' THEN 'EMPTY' ELSE cat_feature_24 END AS cat_feature_24,
	CASE WHEN cat_feature_25 = '' THEN 'EMPTY' ELSE cat_feature_25 END AS cat_feature_25,
	CASE WHEN cat_feature_26 = '' THEN 'EMPTY' ELSE cat_feature_26 END AS cat_feature_26
 FROM click_data
 WHERE rand() <= 0.0006
 DISTRIBUTE BY rand()
 SORT BY rand()
 LIMIT 10000;
```

## RStudio Environment

Go back to the command line on the master node and run this to install R on your cluster:

```clustercommandline
sudo apt-get update
sudo apt-get install -y \
	r-base r-base-dev \
	libcurl4-openssl-dev libssl-dev libxml2-dev
```

Open your browser and type http://localhost:8787 in the address bar. Enter your user name and password. 
Once you are in, open a new R script.

The first order of business is to install all of the packages we need. Installing packages in R 
can take much longer on GCP than the desktop versions we're accustom to.

```r
install.packages("sparklyr")
install.packages("dplyr")
install.packages("ggplot2")
install.packages("tidyr")
install.packages("lsei")
install.packages("stringr")
install.packages("rlang")
install.packages("MLmetrics")

library(sparklyr)
library(dplyr)
library(ggplot2)
library(tidyr)
library(lsei)
library(stringr)
library(rlang)
library(MLmetrics)

sparklyr::spark_install()
```

Next, we will configure Spark and connect to it. Spark will run on our cluster in the background and 
sparklyr will translate the R code into Scala (or Java? I'm not sure) code to make our lives easier.
While we are here it would be helpful to point out potential problems with this approach. 
- There are far fewer users in the sparklyr community than the R community as a whole. This means that there 
is less support in terms of forums. 
- The code is being translated into a paradigm and code that we are not as familiar with as R. When things go 
wrong, it is difficult to read through the error messages and debug. It would be a good idea to read up on Spark 
and Hive to understand how they work.
As an example, when I was creating this project, I was reminded that Hive tables are not automatically ACID compliant. 
When I tried to mutate some columns in dplyr, I ran into errors. I was stuck until I realized that instead of updating 
the single column I was interested in, Hive was creating a new table with the updated column and re-assigning it back 
to the name of the old table.

```r-base
#config
Sys.setenv(SPARK_HOME="/usr/lib/spark")
config <- spark_config()

#connect
sc <- spark_connect(master = "local")
```

We need to convert the categorical variables into an expanded set of binary columns.

```r
click_sample <- tbl(sc, "click_sample") %>%
  select(starts_with("cat"))
```


## Dashboard

The HQL code is not very exciting in this project because there is only one table and the 
column names are masked so it is difficult to formulate interesting analytic questions.

I will add the instructions to set up the next part soon, but it involves moving the data 
from the flat files to GCP's BigQuery. Once the data is there, we can use it to build a Shiny 
app.

```r

```




---

## License

[![License](http://img.shields.io/:license-mit-blue.svg?style=flat-square)](http://badges.mit-license.org)

- **[MIT license](http://opensource.org/licenses/mit-license.php)**
- Copyright 2015 © <a href="http://fvcproductions.com" target="_blank">FVCproductions</a>.