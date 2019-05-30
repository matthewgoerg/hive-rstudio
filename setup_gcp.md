# Set up GCP project

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

After you enter your credentials, you will get a command line. We need to get the data from the web and put 
it in a bucket. We will start with one day's worth of click data. First we download it:

```clustercommandline
gsutil cp http://azuremlsampleexperiments.blob.core.windows.net/criteo/day_1.gz gs://data-science-gcp-224906/
```

Then unzip it and move it into a bucket:

```clustercommandline
gunzip day_2.zip 

export BUCKET=${BUCKET:=<bucket-name>} 
echo "Uploading to bucket $BUCKET..." 
gsutil -m cp day_2 gs://$BUCKET/click/
```

We want to perform the data ingestion and preprocessing in Hive. Enter this line to launch the Hive environment:

```clustercommandline
beeline -u "jdbc:hive2://localhost:10000"
```

[Preprocessing with Hive](https://github.com/matthewgoerg/hive-rstudio/blob/master/hive.md)