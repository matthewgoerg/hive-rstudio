# Hive

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

[RStudio Environment](https://github.com/matthewgoerg/hive-rstudio/blob/master/r.md)