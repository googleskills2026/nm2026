# Prepare Data for ML APIs on Google Cloud: Challenge Lab || GSP323 ||

### Run the following Commands in CloudShell

```bash
cat > gsp323.sh <<'SCRIPT_END'
#!/bin/bash
# GSP323 - Prepare Data for ML APIs on Google Cloud: Challenge Lab
# These values are DIFFERENT in every lab. Defaults are from one lab; override them, e.g.:
#   DATASET=lab_999 TABLE=customers_123 TASK3_FILE=task3-gcs-111.result \
#   TASK4_FILE=task4-cnl-222.result REGION=us-central1 bash gsp323.sh
PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
REGION=${REGION:-us-central1}
DATASET=${DATASET:-lab_134}
TABLE=${TABLE:-customers_578}
BUCKET=${BUCKET:-${PROJECT_ID}-marking}
TASK3_FILE=${TASK3_FILE:-task3-gcs-327.result}
TASK4_FILE=${TASK4_FILE:-task4-cnl-180.result}
SA="${PROJECT_NUMBER}-compute@developer.gserviceaccount.com"
echo "Project: $PROJECT_ID | Region: $REGION | Dataset: $DATASET.$TABLE | Bucket: $BUCKET"
echo "Task 3 file: $TASK3_FILE | Task 4 file: $TASK4_FILE"
gcloud config set compute/region $REGION >/dev/null 2>&1

# ---------- Setup: APIs + permissions ----------
gcloud services enable dataflow.googleapis.com dataproc.googleapis.com language.googleapis.com speech.googleapis.com bigquery.googleapis.com
for ROLE in roles/storage.admin roles/dataflow.worker; do
  gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:$SA" --role=$ROLE --condition=None >/dev/null
done
echo "Waiting for IAM to propagate..."; sleep 30

# ---------- Task 1: BigQuery dataset, bucket, Dataflow job ----------
echo "=== Task 1: Dataset, bucket and Dataflow job ==="
bq mk --dataset $PROJECT_ID:$DATASET || echo "dataset exists"
gcloud storage buckets describe gs://$BUCKET >/dev/null 2>&1 || gcloud storage buckets create gs://$BUCKET --location=$REGION

JOB=$(gcloud dataflow jobs run gsp323-job \
  --gcs-location gs://dataflow-templates-$REGION/latest/GCS_Text_to_BigQuery \
  --region $REGION --worker-machine-type e2-standard-2 \
  --staging-location gs://$BUCKET/temp \
  --parameters inputFilePattern=gs://spls/gsp323/lab.csv,JSONPath=gs://spls/gsp323/lab.schema,outputTable=$PROJECT_ID:$DATASET.$TABLE,bigQueryLoadingTemporaryDirectory=gs://$BUCKET/bigquery_temp,javascriptTextTransformGcsPath=gs://spls/gsp323/lab.js,javascriptTextTransformFunctionName=transform \
  --format="value(id)")
echo "Dataflow job: $JOB (runs in the background while the next tasks proceed)"

# ---------- Task 2: Dataproc cluster + Spark PageRank job ----------
echo "=== Task 2: Dataproc cluster ==="
for i in {1..30}; do gcloud compute networks subnets describe default --region=$REGION >/dev/null 2>&1 && break; sleep 10; done
for a in 1 2 3; do
  gcloud dataproc clusters create example-cluster --region=$REGION \
    --master-machine-type=n2d-standard-2 --master-boot-disk-type=pd-standard --master-boot-disk-size=100GB \
    --worker-machine-type=n2d-standard-2 --worker-boot-disk-type=pd-standard --worker-boot-disk-size=100GB \
    --num-workers=2 && break
  echo "Cluster creation failed, cleaning up and retrying..."
  gcloud dataproc clusters delete example-cluster --region=$REGION --quiet 2>/dev/null; sleep 60
done

CZONE=$(gcloud dataproc clusters describe example-cluster --region=$REGION --format="value(config.gceClusterConfig.zoneUri.basename())")
echo "Copying data.txt into HDFS via master node (zone $CZONE)..."
gcloud compute ssh example-cluster-m --zone=$CZONE --quiet \
  --command="hdfs dfs -cp -f gs://spls/gsp323/data.txt /data.txt"

gcloud dataproc jobs submit spark --cluster=example-cluster --region=$REGION \
  --class=org.apache.spark.examples.SparkPageRank \
  --jars=file:///usr/lib/spark/examples/jars/spark-examples.jar \
  --max-failures-per-hour=1 -- /data.txt

# ---------- Task 3: Speech-to-Text ----------
echo "=== Task 3: Speech-to-Text ==="
cat > request.json <<'EOF'
{"config":{"encoding":"FLAC","languageCode":"en-US"},"audio":{"uri":"gs://spls/gsp323/task3.flac"}}
EOF
curl -s -X POST -H "Content-Type: application/json" -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  --data-binary @request.json "https://speech.googleapis.com/v1/speech:recognize" > $TASK3_FILE
if ! grep -q '"results"' $TASK3_FILE; then
  echo "Bearer token failed, trying an API key..."
  gcloud services api-keys create --display-name="gsp323-key" --api-target=service=speech.googleapis.com
  KN=$(gcloud services api-keys list --filter="displayName=gsp323-key" --format="value(name)" | head -n 1)
  API_KEY=$(gcloud services api-keys get-key-string "$KN" --format="value(keyString)")
  for i in {1..12}; do
    curl -s -X POST -H "Content-Type: application/json" --data-binary @request.json \
      "https://speech.googleapis.com/v1/speech:recognize?key=${API_KEY}" > $TASK3_FILE
    grep -q '"results"' $TASK3_FILE && break; sleep 10
  done
fi
cat $TASK3_FILE
gcloud storage cp $TASK3_FILE gs://$BUCKET/$TASK3_FILE --content-type=application/json

# ---------- Task 4: Natural Language API ----------
echo "=== Task 4: Natural Language API ==="
gcloud ml language analyze-entities \
  --content="Old Norse texts portray Odin as one-eyed and long-bearded, frequently wielding a spear named Gungnir and wearing a cloak and a broad hat." > $TASK4_FILE
cat $TASK4_FILE
gcloud storage cp $TASK4_FILE gs://$BUCKET/$TASK4_FILE --content-type=application/json

# ---------- Wait for the Dataflow job ----------
echo "=== Waiting for the Dataflow job to finish ==="
for i in {1..60}; do
  STATE=$(gcloud dataflow jobs describe $JOB --region=$REGION --format="value(currentState)")
  echo "Dataflow state: $STATE ($i/60)"
  case "$STATE" in JOB_STATE_DONE) break;; JOB_STATE_FAILED|JOB_STATE_CANCELLED) echo "!! Dataflow job failed. Check Console > Dataflow."; break;; esac
  sleep 20
done
bq query --use_legacy_sql=false "SELECT COUNT(*) AS rows_loaded FROM \`$PROJECT_ID.$DATASET.$TABLE\`"
echo "Done! Now click 'Check my progress' for each task."
SCRIPT_END
bash gsp323.sh
```

### Congratulations 🎉 for completing the Lab !

##### ***You Have Successfully Demonstrated Your Skills And Determination.***

##### ***Well done!***
