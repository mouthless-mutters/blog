+++
title = "Cloud Storage to BigQuery: Building an Event-Driven Pipeline on GCP"
date = "2026-02-18T09:00:00-05:00"
draft = false
+++

I wanted to build some things to learn more about GCP, so I decided to create a pipeline that will automatically take files from a Cloud Storage bucket (on upload) and end up in a BigQuery table. Since I enjoy Jiu-Jitsu, I am uploading BJJ match clips in order to eventually do some AI analysis.

Here are the steps I took.

## Step 1: Create a Cloud Storage Bucket

First, I created a bucket to hold my raw match clips.

    gcloud storage buckets create gs://bjj-video-raw --location=us-central1

This bucket acts as the entry point to the system. Any object uploaded here should trigger processing.

## Step 2: Create a BigQuery Dataset and Table

Next, I created a dataset:

    bq --location=us create -d bjj_analytics

Then I created a table for video metadata:

    bq mk --table bjj_analytics.video_metadata name:STRING,size:INTEGER,content_type:STRING,time_created:TIMESTAMP,updated:TIMESTAMP

The schema is intentionally simple:

-name
-size
-content_type
-time_created
-updated

## Step 3: Write the Cloud Function (Gen 2)

I used a 2nd-gen Cloud Function, which runs on Cloud Run and uses Eventarc under the hood.

Here is the function code (main.py):

```import functions_framework
from google.cloud import bigquery

client = bigquery.Client()

@functions_framework.cloud_event
def log_video_metadata(cloud_event):
    data = cloud_event.data

    table = client.get_table("project-<project-guid>.bjj_analytics.video_metadata")

    row = {
        "video_id": data["id"],
        "file_name": data["name"],
        "bucket": data["bucket"],
        "size_bytes": int(data["size"]),
        "content_type": data.get("contentType", "unknown"),
        "time_created": data["timeCreated"]
    }

    errors = client.insert_rows_json(table, [row])
    if errors:
        print("BigQuery insert errors:", errors)
    else:
        print("Insert successful")
```


requirements.txt:
```
functions-framework
google-cloud-bigquery
```

## Step 4: IAM Configuration

Cloud Functions Gen 2 involves:

-Eventarc
-Cloud Run
-A service account

I granted BigQuery permissions to the default compute service account:

    gcloud projects add-iam-policy-binding project-<project-guid> --member="serviceAccount:<project-num>-compute@developer.gserviceaccount.com" --role="roles/bigquery.dataEditor"

    gcloud projects add-iam-policy-binding project-<project-guid> --member="serviceAccount:<project-num>-compute@developer.gserviceaccount.com" --role="roles/bigquery.jobUser"

I also had to grant Eventarc permissions:

    gcloud projects add-iam-policy-binding project-<project-guid> --member="serviceAccount:<project-num>-compute@developer.gserviceaccount.com" --role="roles/eventarc.eventReceiver"

Without these, I encountered errors such as:

eventarc.events.receiveEvent denied

bigquery.tables.get denied

This was one of the most annoying parts of the process.

## Step 5: Deploy the Function

Deployment looked like this:

    gcloud functions deploy log-video-metadata --gen2 --runtime=python311 --region=us-central1 --source=. --entry-point=log_video_metadata --trigger-event-filters=type=google.cloud.storage.object.v1.finalized --trigger-event-filters=bucket=bjj-video-raw

After IAM was corrected, deployment succeeded and the trigger was active.

## Step 6: Test the Pipeline

To test:

    echo "bjj test" > test.txt
    gcloud storage cp test.txt gs://bjj-video-raw/

Then check logs:

    gcloud functions logs read log-video-metadata --region=us-central1

I saw:

    Insert successful

And when I checked BigQuery, the row appeared in the table.

Now:

Existing videos are logged

New uploads are automatically processed

The pipeline is fully functional.