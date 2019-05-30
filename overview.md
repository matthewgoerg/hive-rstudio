**Dashboard preview**

![Dashboard](https://github.com/matthewgoerg/hive-rstudio/blob/master/dashboard.png)

***Note:*** This is not my ideal approach for doing machine learning on the cloud. I think the 
solution involves serverless machine learing on GCP or AWS using Tensorflow in Python.
 
# Machine Learning and Dashboarding on the Cloud

Using Google Cloud Platform, Hive, Spark, and RStudio

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

[Set Up Google Cloud Project](https://github.com/matthewgoerg/hive-rstudio/blob/master/setup_gcp.md#set-up-gcp-project)