**Dashboard preview**

![Dashboard](https://github.com/matthewgoerg/hive-rstudio/blob/master/dashboard.png)

***Note:*** This is not my ideal approach for solving machine learning on the cloud. I think the 
future of this topic is in serverless machine learing using tools like ---- and ---- in Python and 
using GCP. Machine learning with HDFS is not 
 
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

gcloud sql instances create hive-metastore4 \
	--database-version="MYSQL_5_7" \
	--activation-policy=ALWAYS \
	--zone $ZONE

gcloud dataproc clusters create hive-cluster \
    --scopes sql-admin \
    --image-version 1.3 \
    --initialization-actions gs://dataproc-initialization-actions/cloud-sql-proxy/cloud-sql-proxy.sh \
    --properties hive:hive.metastore.warehouse.dir=gs://$PROJECT-warehouse/datasets \
    --metadata "hive-metastore-instance=$PROJECT:$REGION:hive-metastore4"

```

Now download the Google Cloud SDK onto your machine. You will also need a SSH client. I am using PuTTY. 
Enter the following code to launch PuTTY and SSH into your cluster.

```googlecloudsdk
gcloud compute ssh ^
	--zone=us-central1-a ^
	--project=<your-project-name> ^
 	hive-cluster-m -- ^
	-L 8787:localhost:8787
```

After you enter your credentials, you will get a command line. Enter this line to launch the Hive environment.

```clustercommandline
beeline -u "jdbc:hive2://localhost:10000"
```

## Hive

The Hive query language (HQL) is substantially similar to standard SQL. For example, to view all the tables in 
your environment, type this:

```hiveql
SHOW TABLES;
+-----------+
| tab_name  |
+-----------+
+-----------+
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
LOCATION 'gs://data-science-gcp-224906/click/';
```

```hiveql
SELECT COUNT(*) FROM click_data;
+------------+
|    _c0     |
+------------+
| 196792019  |
+------------+
```

This table is 197 million rows long. Even though we built a cluster, it is a good idea to 
test out our machine learning pipeline on a subset of the data. Once, we know it's working,
we can go back and run it on our entire data. The following code will randomly sample 100,000
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
 WHERE rand() <= 0.006
 DISTRIBUTE BY rand()
 SORT BY rand()
 LIMIT 1000000;
```
This took about 2.5 minutes.

I did not end up using the following code, but I am leaving it in because it shows how I approach a 
problem with Hive.

Eventually, we are going to need to convert the categorical columns to binary dummy variables so that 
we can run machine learing models. It would be nice to use all of the levels within each category, but 
we find that there are 1.1 million different levels for the first categorical variable alone. It would 
be too costly to have millions of variables for each observation. Therefore, we will follow ---- and 
find the top 20 most common levels for each variable and leave the rest as zeros. This should give us 
similar results to using all the levels at a far lower computational cost.

First step is to fin the most common levels for each variable. To accomplish this, we will use two small 
files and execute them from the cluster command line.

The first file is a bash script that loops through each categorical variable and launches a HQL script for 
each one. The HQL script is simply a query that returns a table of the top 20 most common variables for the 
given categorical field. These tables will come into play as we set up the ML pipeline. 

***loop_data.sh***
```bashscript
for flag in $(seq -f "%02g" 1 26);
do
  hive --hivevar mytable="table_cat_feature_"$flag --hivevar myvar="cat_feature_"$flag flag=$flag -f new_data.hql
done
```
***new_data.hql***
```hql
DROP TABLE IF EXISTS ${hivevar:mytable};

CREATE TABLE ${hivevar:mytable} AS
SELECT ${hivevar:myvar}, COUNT(*) AS count_
FROM click_sample
GROUP BY ${hivevar:myvar}
ORDER BY count_ DESC
LIMIT 20;
```

We are going to leave Hive now, but you can try some analytic queries here if you want.

## RStudio Environment

Go back to the command line on the master node and run this to install R on your cluster:

```clustercommandline
sudo apt-get update
sudo apt-get install -y \
    r-base r-base-dev \
    libcurl4-openssl-dev libssl-dev libxml2-dev

sudo apt-get install gdebi-core
wget https://download2.rstudio.org/server/debian9/x86_64/rstudio-server-1.2.1335-amd64.deb
sudo gdebi rstudio-server-1.2.1335-amd64.deb

sudo adduser [USER_NAME]
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

Let's connect to the click_sample table so we can modify it.

```r
click_sample <- tbl(sc, "click_sample")
```

We need to convert the categorical variables into an expanded set of binary columns. This loop 
goes through each of the 26 categorical variable columns and does three things:
- ft_string_indexer converts each of the masked labels within the column to a numeric index because 
the machine learning models won't take strings as inputs
- the first categorical column contains over a million different labels and this is going to 
be too much for our models to handle. I learned that if we use only the 20 most frequent labels 
we can get similar ML performance at a much lower computational cost. The mutate step turns all 
of the labels that are not in the 20 most frequent to 0.
- ft_one_hot_encoder converts the indicies to a column of vectors with binary values for each 
level in the column.

```r
for (i in 1:26) {

idx_col <- paste0('cat_idx_', str_pad(i, 2, pad = "0"))



click_sample <- click_sample %>%

  ft_string_indexer(input_col = paste0('cat_feature_', str_pad(i, 2, pad = "0")),

                    output_col = paste0('cat_idx_', str_pad(i, 2, pad = "0"))) %>%

  mutate(!!idx_col := ifelse(!!sym(idx_col) >= 21, 21, !!sym(idx_col))) %>%

  ft_one_hot_encoder(input_col = paste0('cat_idx_', str_pad(i, 2, pad = "0")),

                     output_col = paste0('cat_vec_', str_pad(i, 2, pad = "0")))

}
```

We want to drop all of the original and intermediate categorical columns.

```r
click_sample <- click_sample %>%

  select(-starts_with('cat_feat')) %>%

  select(-starts_with('cat_idx'))
```

The neural net model needs to know how many individual features are in the model. This is complicated 
because the interger columns are length 1 and the lengths for each of the categorical columns ranges 
between 2 and 21. Here's a loop that counts the lengths of each categorical column and adds 13 for the 
number of integer columns.

```r
preview <- as.data.frame(head(click_sample, 1))



num_features <- 13

for (i in 15:40) {

  num_features <- features + length(preview[1,i][[1]])

}
```

Now we're ready to start the actual machine learning. My approach for this project is an ensemble 
of four models and consists of three steps: Training on 60% of the sample, constructing weights for 
each model with 10% of the data, and validating on 40% of the data to see how well we predict on new 
data.

```r
click_data <- click_sample %>%

  sdf_random_split(train = 0.6, validation = 0.4, weight = 0.1, seed = 123)

train_tbl <- click_data$train

test_tbl <- click_data$validation

weight_tbl <- click_data$weight
```

Here's the formula we will use in each model. 

```r
ml_formula <- formula(label ~ .)
```

The four models I'm using are Decision Tree, Gradient Boosted Trees, Random Forest, and Neural Net. 
Below I'm training each one. I am not doing much with the hyperparameters but they will need to be 
tuned eventually. Let's try running with mostly defaults and see how they perform.

```r

ml_dt <- ml_decision_tree_classifier(x=train_tbl, formula=ml_formula)


ml_gbt <- ml_gbt_classifier(train_tbl, ml_formula, max_iter = 250, step_size = 0.01)


ml_rf <- ml_random_forest_classifier(train_tbl, ml_formula, num_trees = 250)


ml_nn <- ml_multilayer_perceptron_classifier(train_tbl, ml_formula, layers = c(num_features, 200, 100, 2))

```

I need to convert this code into an lapply function. This step collects predicted values for 
each observation in the weighting data set and merges them all into a single table.

```r
log_pred <- ml_predict(ml_log, weight_tbl) %>%
  select(probability_1) %>%
  mutate(log_prob = probability_1) %>%
  select(-probability_1)

gbt_pred <- ml_predict(ml_gbt, weight_tbl) %>%
  select(probability_1) %>%
  mutate(gbt_prob = probability_1) %>%
  select(-probability_1)

rf_pred <- ml_predict(ml_rf, weight_tbl) %>%
  select(probability_1)  %>%
  mutate(rf_prob = probability_1) %>%
  select(-probability_1)

nn_pred <- ml_predict(ml_nn, weight_tbl) %>%
  select(label, probability_1) %>%
  mutate(nn_prob = probability_1) %>%
  select(-probability_1)

combined <- as.data.frame(cbind(nn_pred, gbt_pred, rf_pred, log_pred))
```

This step regresses a matrix of predicted values for each of the 4 models 
against the observed label to determine how the models compare. I use the 
pnnls() instead of lm() because I want the sum of the coefficients to be 1.

```r
a <- as.matrix(combined[2:4])

b <- combined$label



my_coefs <- pnnls(a, b, sum = 1)$x
my_coefs
```

This step repeats the process for the validation (test) data set and 
collects them into a table.

```r
log_pred <- ml_predict(ml_log, test_tbl) %>%
  select(probability_1) %>%
  mutate(log_prob = probability_1) %>%
  select(-probability_1)

rf_pred <- ml_predict(ml_rf, test_tbl) %>%
  select(probability_1)  %>%
  mutate(rf_prob = probability_1) %>%
  select(-probability_1)

nn_pred <- ml_predict(ml_nn, test_tbl) %>%
  select(label, probability_1) %>%
  mutate(nn_prob = probability_1) %>%
  select(-probability_1)

combined_test <- as.data.frame(cbind(log_pred, nn_pred, gbt_pred, rf_pred))
```

Here I'm adding a column to the table of predicted values for the test set.
weighted_prob is the weighted average for each of the predicted models. 

If my coefficients from the previous step are log=0.1, nn=0.2, gbt=0.4, rf=0.3 
and my predicted values for an observation are log=0.02, nn=0.03, gbt=0.03, rf=0.01 
then the weighted probability for this observation is 
0.1*0.02 + 0.2*0.03 + 0.4*0.03 + 0.3*0.01 = 0.023. The weighted probability should be 
more accurate than any of the individual models alone.

```r
combined_test <- combined %>%

  mutate(weighted_prob = nn_prob*my_coefs[1]+gbt_prob*my_coefs[2]+rf_prob*my_coefs[3])
```

The log-loss for the weighted probability is 0.137086.

```r
LogLoss(combined_test$weighted_prob, combined_test$label)
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