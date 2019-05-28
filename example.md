**Dashboard Preview**

![Dashboard](https://github.com/matthewgoerg/hive-rstudio/blob/master/dashboard.png)

# Machine Learning and Dashboards on the Cloud

> Using Google Cloud Platform, Hive, Spark, and RStudio

- Find 100GB of real click-through data
- Download the data into Google's Cloud Storage
- Create a Hive/Spark cluster hosted on the Google Cloud Platform
- Query the data using Hive to prepare it for analysis
- Install RStudio on the cluster
- Build an interactive dashboard using R Shiny
- Execute a machine learning pipeline using the R interface for Spark's Machine Learning Library (MLlib)

[![License](http://img.shields.io/:license-mit-blue.svg?style=flat-square)](http://badges.mit-license.org)



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

The HQL code is not very exciting in this project because there is only one table and the 
column names are masked so it is difficult to formulate interesting analytic questions.

I will add the instructions to set up the next part soon, but it involves moving the data 
from the flat files to GCP's BigQuery. Once the data is there, we can use it to build a Shiny 
app.

```r

```


---

## Installation

- All the `code` required to get started
- Images of what it should look like

### Clone

- Clone this repo to your local machine using `https://github.com/fvcproductions/SOMEREPO`

### Setup

- If you want more syntax highlighting, format your code like this:

> update and install this package first

```shell
$ brew update
$ brew install fvcproductions
```
Go to your browser and navigate to http://localhost:8787. 
> now install npm and bower packages

```shell
$ npm install
$ bower install
```

- For all the possible languages that support syntax highlithing on GitHub (which is basically all of them), refer <a href="https://github.com/github/linguist/blob/master/lib/linguist/languages.yml" target="_blank">here</a>.

---

## Features
## Usage (Optional)
## Documentation (Optional)
## Tests (Optional)

- Going into more detail on code and technologies used
- I utilized this nifty <a href="https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet" target="_blank">Markdown Cheatsheet</a> for this sample `README`.

---

## Contributing

> To get started...

### Step 1

- **Option 1**
    - 🍴 Fork this repo!

- **Option 2**
    - 👯 Clone this repo to your local machine using `https://github.com/joanaz/HireDot2.git`

### Step 2

- **HACK AWAY!** 🔨🔨🔨

### Step 3

- 🔃 Create a new pull request using <a href="https://github.com/joanaz/HireDot2/compare/" target="_blank">`https://github.com/joanaz/HireDot2/compare/`</a>.

---

## Team

> Or Contributors/People

| <a href="http://fvcproductions.com" target="_blank">**FVCproductions**</a> | <a href="http://fvcproductions.com" target="_blank">**FVCproductions**</a> | <a href="http://fvcproductions.com" target="_blank">**FVCproductions**</a> |
| :---: |:---:| :---:|
| [![FVCproductions](https://avatars1.githubusercontent.com/u/4284691?v=3&s=200)](http://fvcproductions.com)    | [![FVCproductions](https://avatars1.githubusercontent.com/u/4284691?v=3&s=200)](http://fvcproductions.com) | [![FVCproductions](https://avatars1.githubusercontent.com/u/4284691?v=3&s=200)](http://fvcproductions.com)  |
| <a href="http://github.com/fvcproductions" target="_blank">`github.com/fvcproductions`</a> | <a href="http://github.com/fvcproductions" target="_blank">`github.com/fvcproductions`</a> | <a href="http://github.com/fvcproductions" target="_blank">`github.com/fvcproductions`</a> |

- You can just grab their GitHub profile image URL
- You should probably resize their picture using `?s=200` at the end of the image URL.

---

## FAQ

- **How do I do *specifically* so and so?**
    - No problem! Just do this.

---

## Support

Reach out to me at one of the following places!

- Website at <a href="http://fvcproductions.com" target="_blank">`fvcproductions.com`</a>
- Twitter at <a href="http://twitter.com/fvcproductions" target="_blank">`@fvcproductions`</a>
- Insert more social links here.

---

## Donations (Optional)

- You could include a <a href="https://cdn.rawgit.com/gratipay/gratipay-badge/2.3.0/dist/gratipay.png" target="_blank">Gratipay</a> link as well.

[![Support via Gratipay](https://cdn.rawgit.com/gratipay/gratipay-badge/2.3.0/dist/gratipay.png)](https://gratipay.com/fvcproductions/)


---

## License

[![License](http://img.shields.io/:license-mit-blue.svg?style=flat-square)](http://badges.mit-license.org)

- **[MIT license](http://opensource.org/licenses/mit-license.php)**
- Copyright 2015 © <a href="http://fvcproductions.com" target="_blank">FVCproductions</a>.