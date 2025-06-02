# Submit Your Lab to RKB

1. Create a logo or generate an AI image that best describes your lab, add it, then export your notebook as a PDF.
2. Name your file: `firstname_lastname_cloudchronicles_lab001.pdf`
3. Upload the file to your Github repo's 'submission' folder and email your updated repo link to: submit@labs.rkblueprints.com 
4. Complete the [submission form from your RKB Labs dashboard](https://labs.rkblueprints.com/dashboard) to add your lab within [the showcase laboratory gallery](https://labs.rkblueprints.com/projects).
5. [Rate your lab experience](https://forms.gle/XAkMqvphMSvXDzFj8)

Done. ✅🎉 


echo "--- 1. Enabling required Google Cloud APIs ---"
gcloud services enable pubsub.googleapis.com \
                       storage.googleapis.com \
                       bigquery.googleapis.com \
                       dataflow.googleapis.com \
                       compute.googleapis.com \
                       cloudbuild.googleapis.com \
                       logging.googleapis.com \
                       monitoring.googleapis.com \
                       cloudkms.googleapis.com \
                       serviceusage.googleapis.com \
                       --quiet
echo "APIs enabled."

echo "--- 2. Defining Global Variables ---"
export PROJECT_ID=$(gcloud config get-value project)
export PRIMARY_REGION="europe-west1"
export SECONDARY_REGION="europe-west4"
export DATA_TOPIC_NAME="dr-pipeline-data-topic"
export RAW_DATA_BUCKET_NAME="dr-lab-raw-data-${PROJECT_ID}" # Unique bucket name
export BQ_DATASET_ID="dr_pipeline_dataset"
export BQ_TABLE_NAME="raw_events"
export DATAFLOW_JOB_NAME_PRIMARY="dr-pipeline-primary"
export DATAFLOW_JOB_NAME_SECONDARY="dr-pipeline-secondary"
export GCS_TEMP_LOCATION="gs://${RAW_DATA_BUCKET_NAME}/tmp/dataflow/" # Dataflow temporary files location

echo "Project ID: $PROJECT_ID"
echo "Primary Region: $PRIMARY_REGION"
echo "Secondary Region: $SECONDARY_REGION"
echo "Raw Data Bucket: $RAW_DATA_BUCKET_NAME"
echo "Variables defined."

echo "--- 3. Setting Up Pub/Sub Topic and Subscriptions ---"
# Create the global Pub/Sub topic
gcloud pubsub topics create $DATA_TOPIC_NAME --project=$PROJECT_ID

# Create a subscription in the primary region for the Dataflow job
gcloud pubsub subscriptions create ${DATA_TOPIC_NAME}-sub-${PRIMARY_REGION} \
    --topic=$DATA_TOPIC_NAME \
    --ack-deadline=60 \
    --message-retention-duration=7d \
    --project=$PROJECT_ID

# Create a subscription in the secondary region for the Dataflow job
gcloud pubsub subscriptions create ${DATA_TOPIC_NAME}-sub-${SECONDARY_REGION} \
    --topic=$DATA_TOPIC_NAME \
    --ack-deadline=60 \
    --message-retention-duration=7d \
    --project=$PROJECT_ID

echo "Pub/Sub topic '$DATA_TOPIC_NAME' and subscriptions created."

echo "--- 4. Creating Multi-Region Cloud Storage Bucket ---"
gcloud storage buckets create gs://$RAW_DATA_BUCKET_NAME \
    --location=EU \
    --uniform-bucket-level-access \
    --project=$PROJECT_ID

echo "Multi-Region Cloud Storage bucket '$RAW_DATA_BUCKET_NAME' created."

echo "--- 5. Creating Multi-Region BigQuery Dataset and Table ---"
# Create a multi-region dataset (EU for europe-west regions)
bq --location=EU mk --dataset $PROJECT_ID:$BQ_DATASET_ID

# Create a table (if it doesn't exist)
bq query --use_legacy_sql=false \
    "CREATE TABLE IF NOT EXISTS \`${PROJECT_ID}.${BQ_DATASET_ID}.${BQ_TABLE_NAME}\` (
        message STRING,
        timestamp TIMESTAMP,
        processed_by STRING
    )" \
    --project_id=$PROJECT_ID

echo "BigQuery dataset '$BQ_DATASET_ID' and table '$BQ_TABLE_NAME' created."

echo "--- 6. Granting Dataflow Service Account Permissions ---"
# Get the Compute Engine default service account email
export DATAFLOW_SA=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")-compute@developer.gserviceaccount.com

# Grant necessary roles
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:$DATAFLOW_SA" \
    --role="roles/dataflow.worker" \
    --role="roles/pubsub.editor" \
    --role="roles/bigquery.dataEditor" \
    --role="roles/storage.objectAdmin" \
    --role="roles/logging.logWriter" \
    --role="roles/monitoring.metricWriter" \
    --project=$PROJECT_ID \
    --quiet

echo "Permissions granted to Dataflow service account: $DATAFLOW_SA"

echo "--- 7. Launching Primary Dataflow Job in $PRIMARY_REGION ---"
echo "This may take a few minutes to start and auto-scale."

gcloud dataflow jobs run $DATAFLOW_JOB_NAME_PRIMARY \
    --gcs-location="gs://dataflow-templates/latest/PubSub_to_BigQuery" \
    --region=$PRIMARY_REGION \
    --staging-location=$GCS_TEMP_LOCATION \
    --temp-location=$GCS_TEMP_LOCATION \
    --service-account-email=$DATAFLOW_SA \
    --parameters \
        inputTopic=projects/${PROJECT_ID}/topics/${DATA_TOPIC_NAME},\
        outputTableSpec=${PROJECT_ID}:${BQ_DATASET_ID}.${BQ_TABLE_NAME},\
        javascriptTextTransformGcsPath=gs://dataflow-templates/latest/javascript/PubSub_to_BigQuery_XLS_Schema.js,\
        javascriptTextTransformFunctionName=transform,\
        outputDeadletterTable=projects/${PROJECT_ID}/datasets/${BQ_DATASET_ID}/tables/dead_letter_events \
    --project=$PROJECT_ID

echo "Primary Dataflow job launched. Monitor its status in the Dataflow console (console.cloud.google.com/dataflow/jobs)."
echo "Waiting for Dataflow job to transition to 'Running' state..."
sleep 120 # Give some time for job to show up in console, might need more

read -p "Check Dataflow console. Once '$DATAFLOW_JOB_NAME_PRIMARY' shows 'Running', press Enter to continue..."

echo "--- 8. Setting Up Data Generator ---"
echo "IMPORTANT: Open a NEW Cloud Shell tab or a separate terminal for this step."
echo "Paste the following Python code into the new tab/terminal and save it as 'data_generator.py'."
echo "Then, run it using the 'python3 data_generator.py' command as shown below."
echo "--------------------------------------------------------------------------------"
cat << 'EOF' > data_generator.py
# data_generator.py
import os
import time
import json
import datetime
from google.cloud import pubsub_v1

project_id = os.environ.get('GCP_PROJECT_ID')
topic_name = os.environ.get('PUBSUB_TOPIC_NAME')

if not project_id or not topic_name:
    print("Error: GCP_PROJECT_ID and PUBSUB_TOPIC_NAME environment variables must be set.")
    print("Please set them: export GCP_PROJECT_ID='your-project-id'; export PUBSUB_TOPIC_NAME='your-topic-name'")
    exit(1)

publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path(project_id, topic_name)

print(f"Publishing messages to {topic_path} every 5 seconds...")

counter = 0
try:
    while True:
        message_data = {
            "event_id": counter,
            "timestamp": datetime.datetime.now(datetime.timezone.utc).isoformat(),
            "data": f"This is message number {counter} from the generator."
        }
        data_json = json.dumps(message_data).encode("utf-8")
        
        future = publisher.publish(topic_path, data_json)
        future.result() # Blocks until the publish request is complete
        print(f"Published message {counter}: {message_data['data']}")
        
        counter += 1
        time.sleep(5)
except KeyboardInterrupt:
    print("\nData generator stopped by user.")
except Exception as e:
    print(f"An error occurred: {e}")
finally:
    publisher.transport.close()
    print("Pub/Sub publisher client closed.")
EOF

echo "File 'data_generator.py' created."
echo "In your NEW Cloud Shell tab/terminal, run these commands:"
echo ""
echo "export GCP_PROJECT_ID=$PROJECT_ID"
echo "export PUBSUB_TOPIC_NAME=$DATA_TOPIC_NAME"
echo "python3 data_generator.py"
echo ""
echo "--------------------------------------------------------------------------------"
read -p "After starting the data generator in the NEW tab/terminal, press Enter to continue..."

echo "--- 9. Verifying Initial Data Flow ---"
echo "Go to the BigQuery Console (console.cloud.google.com/bigquery)."
echo "Navigate to your dataset: '$BQ_DATASET_ID', then the table: '$BQ_TABLE_NAME'."
echo "Run a query like: SELECT * FROM \`${PROJECT_ID}.${BQ_DATASET_ID}.${BQ_TABLE_NAME}\` ORDER BY timestamp DESC LIMIT 10"
echo "You should see new messages appearing every few seconds."

read -p "Once you confirm data is flowing, press Enter to continue to failover simulation..."

echo "--- 1. Simulating europe-west1 Outage (Canceling Primary Dataflow Job) ---"
echo "This will stop processing from the primary region, but Pub/Sub will buffer messages."

# Get the job ID.
export PRIMARY_JOB_ID=$(gcloud dataflow jobs list --region=$PRIMARY_REGION --filter="name=${DATAFLOW_JOB_NAME_PRIMARY} AND state=RUNNING" --format="value(id)" | head -n 1)

if [ -z "$PRIMARY_JOB_ID" ]; then
    echo "Primary Dataflow job '$DATAFLOW_JOB_NAME_PRIMARY' not found or not running in $PRIMARY_REGION. Please verify its status in the Dataflow console."
else
    gcloud dataflow jobs cancel $PRIMARY_JOB_ID --region=$PRIMARY_REGION --project=$PROJECT_ID --quiet
    echo "Primary Dataflow job ($PRIMARY_JOB_ID) canceled. Data will continue to accumulate in Pub/Sub."
fi

echo "Wait for the Dataflow job to show as 'Canceled' in the console before proceeding."
read -p "Once the primary job is 'Canceled', press Enter to verify and proceed..."

read -p "Confirm that new data has stopped appearing in BigQuery. Press Enter to launch the secondary Dataflow job..."

echo "--- 2. Launching Secondary Dataflow Job in $SECONDARY_REGION ---"
echo "This job will take over processing from the Pub/Sub topic."

gcloud dataflow jobs run $DATAFLOW_JOB_NAME_SECONDARY \
    --gcs-location="gs://dataflow-templates/latest/PubSub_to_BigQuery" \
    --region=$SECONDARY_REGION \
    --staging-location=$GCS_TEMP_LOCATION \
    --temp-location=$GCS_TEMP_LOCATION \
    --service-account-email=$DATAFLOW_SA \
    --parameters \
        inputTopic=projects/${PROJECT_ID}/topics/${DATA_TOPIC_NAME},\
        outputTableSpec=${PROJECT_ID}:${BQ_DATASET_ID}.${BQ_TABLE_NAME},\
        javascriptTextTransformGcsPath=gs://dataflow-templates/latest/javascript/PubSub_to_BigQuery_XLS_Schema.js,\
        javascriptTextTransformFunctionName=transform,\
        outputDeadletterTable=projects/${PROJECT_ID}/datasets/${BQ_DATASET_ID}/tables/dead_letter_events \
    --project=$PROJECT_ID

echo "Secondary Dataflow job launched. Monitor its status in the Dataflow console."
echo "Waiting for the secondary Dataflow job to transition to 'Running' state and start processing."
sleep 180 # Give some time for job to show up and start processing

echo "--- 3. Verifying Failover to Secondary Region ---"
echo "Go back to the BigQuery Console and query the 'raw_events' table again."
echo "You should now see new messages appearing, processed by the job in '$SECONDARY_REGION'."

read -p "Once you confirm data is flowing again into BigQuery, press Enter to proceed to cleanup..."

echo "--- PART 3: Lab Cleanup ---"
echo "!!! IMPORTANT: ALL LAB RESOURCES WILL BE DELETED TO AVOID ONGOING CHARGES. !!!"
echo "!!! THIS ACTION IS IRREVERSIBLE. !!!"

read -p "Are you sure you want to proceed with cleanup? (yes/no): " CONFIRM_CLEANUP
if [ "$CONFIRM_CLEANUP" != "yes" ]; then
    echo "Cleanup aborted. Remember to manually delete resources to avoid charges."
    exit 0
fi

echo "1. Stopping the Data Generator (if still running)..."
echo "Go to the terminal/Cloud Shell tab running 'data_generator.py' and press Ctrl+C to stop it."
read -p "Press Enter after you've stopped the data generator..."

echo "2. Canceling Dataflow Jobs..."
# Re-fetch job IDs to ensure we have the correct ones
PRIMARY_JOB_ID=$(gcloud dataflow jobs list --region=$PRIMARY_REGION --filter="name=${DATAFLOW_JOB_NAME_PRIMARY}" --format="value(id)" | head -n 1)
SECONDARY_JOB_ID=$(gcloud dataflow jobs list --region=$SECONDARY_REGION --filter="name=${DATAFLOW_JOB_NAME_SECONDARY}" --format="value(id)" | head -n 1)

if [ -n "$PRIMARY_JOB_ID" ]; then
    echo "Canceling primary Dataflow job: $PRIMARY_JOB_ID"
    gcloud dataflow jobs cancel $PRIMARY_JOB_ID --region=$PRIMARY_REGION --project=$PROJECT_ID --quiet
fi

if [ -n "$SECONDARY_JOB_ID" ]; then
    echo "Canceling secondary Dataflow job: $SECONDARY_JOB_ID"
    gcloud dataflow jobs cancel $SECONDARY_JOB_ID --region=$SECONDARY_REGION --project=$PROJECT_ID --quiet
fi

echo "Waiting for Dataflow jobs to terminate... (this may take a few minutes)"
sleep 300 # Give Dataflow jobs some time to terminate (5 minutes)

echo "3. Deleting Pub/Sub Subscriptions..."
gcloud pubsub subscriptions delete ${DATA_TOPIC_NAME}-sub-${PRIMARY_REGION} --project=$PROJECT_ID --quiet
gcloud pubsub subscriptions delete ${DATA_TOPIC_NAME}-sub-${SECONDARY_REGION} --project=$PROJECT_ID --quiet

echo "4. Deleting Pub/Sub Topic..."
gcloud pubsub topics delete $DATA_TOPIC_NAME --project=$PROJECT_ID --quiet

echo "5. Deleting BigQuery Dataset '$BQ_DATASET_ID' and all its tables..."
# CAUTION: This command irrevocably deletes all data in the dataset!
bq rm -r -f $PROJECT_ID:$BQ_DATASET_ID

echo "6. Deleting Cloud Storage Bucket '$RAW_DATA_BUCKET_NAME' and all its contents..."
# CAUTION: This command irrevocably deletes all data in the bucket!
gcloud storage rm -r gs://$RAW_DATA_BUCKET_NAME --project=$PROJECT_ID --quiet

echo "--- Lab Cleanup Complete ---"
echo "Please verify in the Google Cloud Console (console.cloud.google.com) that all resources have been removed to avoid unexpected charges."
